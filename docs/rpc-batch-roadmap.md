# Глобальный план: перевод UI на батчевые RPC + оптимизация БД в ядре

> Ветка: `feat/rpc-batch-migration`. Форк: `weavedwires/deltachat-android`.
> Это дорожная карта из трёх направлений. Детальный план сообщений — в `rpc-batch-messages.md`.

## Контекст (зачем всё это)

Delta Chat Android общается с Rust-ядром двумя путями, оба в одном нативном модуле:

- **Прямой C-ABI** (`DcContext` → `dc_get_msg`/`dc_get_chatlist`/`dc_chatlist_get_summary`) —
  синхронные указатели, загрузка по одному. Основной путь UI сегодня.
- **JSON-RPC** (`Rpc`/`FFITransport` → `dc_jsonrpc_*` → `CommandApi`) — DTO, батчи.

RPC вводился (PR #2479) как путь «не писать новый CFFI под каждую фичу», а не как
замена UI; миграция идёт точечно (location #4400, relay #4637, transports #4657).

**Ключевой вывод ресёрча:** узкое место — БД (SQL), а не копирование в памяти.
`getMsg`/`getSummary` = SELECT на каждую строку на main-потоке. Нативные указатели
(`dc_msg_t` = `Box`) — это lifetime C ABI, а не «оптимизация памяти»; оптимизацию
SELECT'ов в коде делает только `recordCache` (SoftReference).

Батчевые RPC-методы **уже есть в ядре**, но под капотом пока цикл SELECT — батч
срезает round-trip/JNI/main-блокировки, но не число SQL-запросов.

---

## 1. Сообщения (рендер списка чата)

**Что:** перевести рендер списка сообщений с `DcMsg` на батчевый DTO, оставив virtual list.

**Схема:** `getChatMsgs` → id (не меняем) → кеш DTO `Map<Integer, MessageLoadResult>`
→ `getItemViewType`/`bind` читают DTO → prefetch окна через `getMessages(ids)` в фоне
→ действия по id, композер на `DcMsg`.

**Граница (проверено):** рендер = DTO, действия = id (`deleteMessages`/`markseenMsgs`/
`sendReaction`), композер = DcMsg (`SendRelayedMessageUtil`/`ConversationActivity`/
`VideoRecodeManager`/`AttachmentManager`), reply = `quotedMessageId`.

**Подробно:** `rpc-batch-messages.md`.

---

## 2. Чаты (список чатов)

**Что:** перевести рендер списка чатов на батч `getChatlistItemsByEntries`.

**Проблема:** `ConversationListAdapter.onBindViewHolder` на main делает
`dcContext.getChat(chatId)` + `dcChatlist.getSummary(i, chat)` = 2–4 SELECT на строку,
без кеша. `DcChatlist.getSummary(index)` в legacy FFI — по одному, `DcLot` с поштучными
`getText1/2/getTimestamp/getState` (сами геттеры — 0 SQL, но `getSummary` грузит lastmsg
+ contact + reactions).

**Решение:** `getChatlist` (id) → `getChatlistItemsByEntries(accountId, List<Integer> entries)`
→ `Map<String, ChatListItemFetchResult>`. DTO `ChatListItem` полный: `name`, `summaryText1/2`,
`timestamp`, `avatarPath`, `freshMessageCounter`, `isMuted`, `isPinned`, `lastMessageId` и т.д.
Метод уже есть (`Rpc.java:482`).

**Зона (локализована):** `ConversationListAdapter`, `ConversationListItem`/`InboxZero`,
`ThreadRecord`, `DcHelper.getThreadRecord`, `SearchResult`/`SearchListAdapter`, `ProfileAdapter`.

**Подводный камень:** `DcLot` имеет двойное назначение — это и summary списка чатов,
и результат `checkQr()` (QR/прокси/онбординг/поиск). Заменять только в пути рендера
списка, **не** трогая QR-путь (`checkQr` → `ProxySettingsActivity`, `InstantOnboardingActivity`,
`RegistrationQrActivity`, `QrInviteData`).

**Порядок:** делать **первым** — зона проще и безопаснее, батч-метод полный.

---

## 3. БД в ядре (batch-SELECT)

**Что:** научить ядро отдавать пачку сообщений/чатов одним SQL-запросом, а не циклом.

**Проблема:** `deltachat-jsonrpc/src/api.rs` `get_messages`:
```rust
for message_id in message_ids {
    MessageObject::from_msg_id(&ctx, MsgId::new(message_id)).await;
    // → Message::load_from_db → SELECT … WHERE m.id=? (по одному)
}
```

Плюс `load_from_db` тащит `LEFT JOIN chats` + `LEFT JOIN msgs_mdns` на каждое сообщение
(`src/message.rs:488`).

**Решение:**
1. `SELECT … WHERE m.id IN (…ids…)` одним запросом вместо цикла `load_from_db`.
2. Вынести `LEFT JOIN chats`/`msgs_mdns` из per-message в общий (или подзапрос/пакетизация),
   чтобы джойны не повторялись N раз.
3. Аналогично для `get_chatlist_items_by_entries` (сейчас `get_summary` на каждый entry —
   см. `chatlist.rs get_summary2`: lastmsg + contact + reactions).

**Эффект:** батч начнёт резать не только транспорт, но и число SQL — это и есть главный
тезис «оптимизировать БД, а не копирование».

**Ограничение:** это форк `chatmail/core` + пересборка натива через NDK. Отдельная история,
не в этой ветке Android-клиента (если только не подтягивать submodule).

---

## Порядок и приоритеты

| # | Направление | Выигрыш | Риск | Зависимость |
|---|---|---|---|---|
| 1 | Чаты → батч | средний (jank списка чатов) | низкий | нет |
| 2 | Сообщения → батч (рендер) | высокий (главный экран) | средний | нет |
| 3 | БД в ядре (batch-SELECT) | максимальный (режет SQL) | высокий (ядро+NDK) | отдельный форк core |

## Ограничения среды

- В этой среде нет Android SDK/NDK — сборку и проверку APK выполняет заказчик
  (IntelliJ/Android Studio). Изменения пишутся минимальными локальными шагами.
- Правки ядра (п.3) требуют Rust toolchain + NDK и живут в `chatmail/core`, не здесь.
