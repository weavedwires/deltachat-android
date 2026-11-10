# Перевод рендера списка сообщений с `DcMsg` на батчевый RPC (DTO)

> Статус: план, этап 1 из глобального `rpc-batch-roadmap.md`.
> Ветка: `feat/rpc-batch-migration`.

## Цель

Убрать блокировку main-потока при прокрутке чата, не трогая при этом virtual-list
и не переписывая логику действий с сообщениями.

## Проблема (факты из кода)

Скелет virtual list уже есть — `ConversationFragment.reloadList()`:

```java
int[] msgs = dcContext.getChatMsgs((int) chatId, 0, 0);   // 1 SELECT → массив id
```

Но дальше каждая видимая позиция дёргает нативный вызов **на main-потоке**:

| Место | Вызов | SQL |
|---|---|---|
| `ConversationAdapter.getItemViewType(i)` | `getMsg(i)` | 1 SELECT (на каждый layout-pass) |
| `ConversationAdapter.onBindViewHolder` | `getMsg(position)` | 1 SELECT (первый бинд) |
| `ConversationAdapter.getMsg(pos)` | `dcContext.getMsg(id)` | 1 SELECT, кеш `recordCache` (SoftReference) |

- `dc_get_msg` → `Message::load_from_db` → один `SELECT … FROM msgs m LEFT JOIN chats c LEFT JOIN msgs_mdns mdns WHERE m.id=?` (`src/message.rs:488`).
- Геттеры полей (`dc_msg_get_text`, `dc_msg_get_timestamp`, …) → **0 SQL**, читают `Message` из памяти.

Итого: первый доступ к каждой позиции = 1 полный SELECT на main; геттеры полей бесплатны.
`recordCache` (SoftReference) лишь смягчает повторный rebind, пока объект жив.

## Решение

Оставить virtual list (список id от `getChatMsgs`), но рендер перевести на плоский
DTO, заполняемый **батчем**:

1. `getChatMsgs` → `int[] msgs` (id) — не меняем.
2. Вместо `recordCache` (SoftReference<DcMsg>) — кеш DTO `Map<Integer, MessageLoadResult>`.
3. `getItemViewType`/`onBindViewHolder` читают из DTO, а не из `getMsg`.
4. Prefetch окна + скролл → `getMessages(accountId, List<Integer> ids)` одним вызовом в фоне → `notifyItemRangeChanged`.
5. Действия (клик/свайп/reply/детали) — точечный `getMsg(id)` по требованию (или RPC-методы по id).

Методы уже есть в ядре (`Rpc.java`):

- `getMessageIds(accountId, chatId, infoOnly, addDaymarker)` → `List<Integer>` (`:945`)
- `getMessages(accountId, List<Integer> ids)` → `Map<String, MessageLoadResult>` (`:1005`)

## Покрытие DTO — проверено

`ConversationItem` (обычные сообщения) читает из `DcMsg` 12 методов:
`getType, isOutgoing, getText, getId, getFromId, isForwarded, hasHtml, getQuotedText,
getOverrideSenderName, getOriginalMsgId, getDownloadState, getChatId`.

Все покрываются полями `MessageLoadResult.Message`:
`viewType, state/fromId, text, id, fromId, isForwarded, hasHtml, quote, overrideSenderName,
originalMsgId, downloadState, chatId`.

Цитата — DTO даже выгоднее: `MessageQuote` уже несёт `text`, `authorDisplayName`,
`authorDisplayColor`, `image`, `viewType`, `messageId` — то, ради чего legacy FFI делал
ленивый `getQuotedMsg()` (отдельный SELECT). В DTO цитата лежит внутри ответа.

## Граница «рендер / действия / композер»

Проверено по коду: `DcMsg` как объект действия встречается **только** в композере,
над свежесозданным `new DcMsg(context, viewtype)`:

- `SendRelayedMessageUtil.createMessage()` — `setText`, `setFileAndDeduplicate` → `sendMsg`.
- `ConversationActivity` — `setQuote`, `setDraft`, `sendMsg`.
- `VideoRecodeManager` — `setQuote`, `setDimension`, `setDuration`, `setDraft`, `sendMsg`.
- `AttachmentManager` — черновик аудио/вложения → `setDraft`.
- `RemoteReplyReceiver`, `ConversationFragment` (reply privately).

Действия над существующими сообщениями уже идут **по id**, не по `DcMsg`:

- `deleteMessages(ids)`, `markseenMsgs(ids)`, `sendReaction(msgId, …)`,
  `forwardMessages(ids)`, `setPinnedMessageState(msgId, …)`.

Единственное «действие» из спискового `DcMsg` — `setQuote(DcMsg)` при reply
(`DcMsg.setQuote` = `setQuoteCPtr(quote.msgCPtr)`). Заменяется на `quotedMessageId`
(RPC уже принимает: `miscSendMsg(…quotedMessageId)`, `sendMsg(MessageData)`).

**Вывод:** рендер = DTO, действия = id, композер = DcMsg. Потоки не пересекаются
нигде, кроме reply (закрывается `quotedMessageId`).

## Что даёт / что не даёт

- **Даёт:** убирает N блокировок main (jank/ANR от `getMsg` в `getItemViewType`/`bind`),
  N round-trip'ов JNI и N JSON-сериализаций.
- **Не даёт (пока):** меньше SQL — ядро `get_messages` внутри цикл
  (`for id { load_from_db }`), а не `WHERE id IN (...)`. Это этап «БД в ядре»
  из глобального плана.

## План по файлам

1. `ConversationAdapter.java`
   - заменить `recordCache` (SoftReference<DcMsg>) на `Map<Integer, MessageLoadResult>` (+ очередь prefetch);
   - `getItemViewType(i)` читать `viewType` из DTO;
   - `onBindViewHolder` читать DTO; передавать в `bind(...)` DTO вместо `DcMsg`;
   - prefetch окна через `getMessages(ids)` в `Util.runOnBackground`.
2. `BindableConversationItem` / `BaseConversationItem` / `ConversationItem` / `ConversationUpdateItem`
   - `bind(...)` принимает DTO (`MessageLoadResult.Message`) и читает поля из него.
   - `ConversationUpdateItem` проверить отдельно: `getParent()`/`getQuotedMsg()` (в DTO только `parentId`, не текст).
3. `ConversationFragment.java`
   - действия (reply/forward/delete/download) оставить на `getMsg(id)`;
   - reply перевести на `quotedMessageId`.
4. Композер (`SendRelayedMessageUtil`, `ConversationActivity` compose, `VideoRecodeManager`, `AttachmentManager`) — **не трогаем**.

## Риски

- Не размазать «и DTO, и DcMsg» — граница рендер/действия должна быть жёсткой.
- `ConversationUpdateItem` (info-сообщения) — убедиться, что `parentId`/`quote` хватает
  для рендера, иначе точечная догрузка.
- Сборка/проверка — на стороне заказчика (в этой среде нет Android SDK/NDK).
