# Перевод списка чатов на батчевый RPC (`getChatlistItemsByEntries`)

> Часть глобального плана — см. `rpc-batch-roadmap.md` (там архитектура, термины, порядок).
> Это направление 1: делается первым, так как зона локализована и риск низкий.
> Ветка: `feat/rpc-batch-migration`.

## TL;DR

В списке чатов каждая видимая строка грузится на main-потоке несколькими нативными
вызовами (`getChat` + `getSummary` + `getFreshMsgCount`), каждый из которых делает SELECT в
SQLite — итого 3–5 SELECT на строку. План: заменить поштучную загрузку на один батчевый
JSON-RPC вызов `getChatlistItemsByEntries`, который возвращает готовые объекты `ChatListItem`
пачкой, и грузить их в фоне. Main-поток перестаёт блокироваться.

## Контекст

`DcChatlist`, `DcChat`, `DcLot` — нативные обёртки: Java-классы с `long`-указателем на
Rust-объект в куче. `DcChatlist` хранит только список id (загруженных одним SELECT), а
данные строки (`getSummary`) грузятся по одной и без кеша. Альтернатива — батчевый
JSON-RPC, отдающий плоские DTO. Термины (FFI, DTO, CPtr, main-поток, ANR) раскрыты в
роадмапе.

## Текущее состояние (факты из кода)

`ConversationListFragment.loadChatlist()` уже грузит список id **в фоне**
(`loadChatlistAsync` → `Util.runOnAnyBackgroundThread`):

```java
DcChatlist chatlist = DcHelper.getContext(context).getChatlist(listflags, null, 0); // 1 SELECT, фон
...
((ConversationListAdapter) list.getAdapter()).changeData(chatlist); // на main
```

Но `onBindViewHolder` на **main** дёргает ядро по строке:

```java
DcChat chat = dcContext.getChat(dcChatlist.getChatId(i));     // 1 SELECT (load_from_db)
DcLot summary = dcChatlist.getSummary(i, chat);               // 1–3 SELECT (lastmsg + contact + reactions)
DcHelper.getThreadRecord(context, summary, chat)              // → getFreshMsgCount(chatId) = ещё 1 SELECT
```

Итого **3–5 SELECT на строку на main**. Плюс `getItemViewType(i)` тоже читает
`dcChatlist.getChatId(i)` (0 SQL, из памяти — не критично).

## Зависимости от нативных указателей (что надо убрать)

| Класс | Указатель | Методы |
|---|---|---|
| `ConversationListAdapter` | поле `DcChatlist dcChatlist` | `getCnt()`, `getChatId(i)`, `getMsgId(i)`, `getSummary(i, chat)` |
| `ConversationListAdapter` | `DcContext.getChat(id)` → `DcChat` | `getId/getName/getVisibility/isSendingLocations/isMuted/isContactRequest/isOutBroadcast/getProfileImage/getColor/getType/isSelfTalk/isDeviceTalk` |
| `ConversationListItem.bind` | `DcLot dcSummary` | `getState()`, `getText1()`, `getText2()`, `getText1Meaning()`, `getTimestamp()` |
| `ConversationListItem.bind` | `Recipient` (обёртка `DcChat`) | `getName()`, аватар (`profileImage`+`color`), `getDcContact()`, `wasSeenRecently()` |
| `ThreadRecord` | поле `DcLot dcSummary` | `getDisplayBody()` (text1/2/meaning), `isOutChannel()` (`isOutBroadcast`) |
| `DcHelper.getThreadRecord` | `DcLot` + `DcChat` + `getFreshMsgCount(chatId)` | собирает `ThreadRecord` (ещё 1 SELECT) |

Дополнительно `DcChatlist` используется вне этого экрана: `SearchViewModel`/`SearchResult`/
`SearchListAdapter` (поиск), `ProfileAdapter`/`ProfileFragment` (shared chats),
`DirectShareUtil` (direct share). Это **вторичные** потребители — их трогаем вторым заходом,
первый шаг ограничиваем рендером списка чатов.

## DTO `ChatListItem` — полное покрытие

Рендер читает поля, и все они есть в `ChatListItemFetchResult.ChatListItem`:

| Рендерное поле (legacy) | Поле DTO |
|---|---|
| `chat.getName()` / `recipient.getName()` | `name` |
| аватар (`getProfileImage` + `getColor`) | `avatarPath`, `color` |
| `getFreshMsgCount(chatId)` (SELECT!) | `freshMessageCounter` |
| `summary.getState()` (delivery status, bold) | `summaryStatus` |
| `summary.getText1() + text2()` | `summaryText1` (prefix) + `summaryText2` (text) |
| `summary.getTimestamp()` (дата) | `lastUpdated` |
| `chat.getVisibility()` (pinned/archived) | `isPinned`, `isArchived` |
| `chat.isMuted()` | `isMuted` |
| `chat.isContactRequest()` | `isContactRequest` |
| `chat.isSendingLocations()` | `isSendingLocation` |
| `chat.isOutBroadcast()` | `chatType` |
| `chat.isSelfTalk()` / `isDeviceTalk()` | `isSelfTalk`, `isDeviceTalk` |
| `chat.isGroup()` / `isEncrypted()` | `isGroup`, `isEncrypted` |
| dm-контакт + `wasSeenRecently()` | `dmChatContact`, `freshness` |
| `chatId`, `msgId` | `id`, `lastMessageId` |

Единственное, что **не** в DTO напрямую: `msgId` для `onBindViewHolder` сейчас берётся
`dcChatlist.getMsgId(i)` — в DTO есть `lastMessageId`, его и использовать.

## Новые RPC-методы (уже есть)

```
getChatlistEntries(accountId, listFlags, queryString, queryContactId) → List<Integer>   // Rpc.java:469
getChatlistItemsByEntries(accountId, List<Integer> entries) → Map<String, ChatListItemFetchResult>  // Rpc.java:482
```

Связка ровно как для сообщений (`getMessageIds` + `getMessages`):
1. `getChatlistEntries(...)` → список id (включая спец. id `DC_CHAT_ID_ARCHIVED_LINK=6`,
   `DC_CHAT_ID_ALLDONE_HINT=7`).
2. `getChatlistItemsByEntries(ids)` → полные DTO.

## Как сократить 3–5 запросов в один — честно, два уровня

**Уровень клиента (что делаем сейчас):** 3–5 SELECT на строку на main → **один RPC-вызов**
на окно в фоне. Main перестаёт блокироваться полностью. Это главный выигрыш (jank/ANR).

**Уровень SQL (направление 3 глобального плана, НЕ сейчас):** внутри ядра
`get_chatlist_items_by_entries` (`api.rs:787`) — это цикл:

```rust
for &entry in entries.iter() {
    result.insert(entry, get_chat_list_item_by_id(&ctx, entry).await ...);
}
```

а `get_chat_list_item_by_id` делает на каждый чат **~9–11 SELECT**:
`get_fresh_msg_cnt` + `get_last_message_for_chat` + `Chat::load_from_db` + `get_summary2`
(внутри снова lastmsg) + `get_profile_image` + `Message::load_from_db_optional` (дубль lastmsg)
+ `get_chat_contacts` + `Contact::get_by_id` + `get_color` + `is_encrypted` + `is_self_in_chat`.

То есть батч срезает **round-trip/main-блокировки**, но не число SQL (даже увеличивает его
за счёт обогащения). Честный «1 SELECT на пачку» — это batch-SELECT в ядре
(`WHERE c.id IN (…)` + JOIN'ы + один проход), отдельная задача ядра.

## Подводные камни

1. **Полиморфный DTO.** `ChatListItemFetchResult` — `ChatListItem` / `ArchiveLink` / `Error`.
   `ARCHIVED_LINK` возвращает `ArchiveLink` (только `freshMessageCounter`), а не `ChatListItem`.
   `ALLDONE_HINT` — проверить, что возвращает (предположительно `Error`/особый). Адаптер
   должен ветвиться по `kind`.
2. **`Recipient` — глубокая обёртка.** Для списка чатов надо строить Recipient из DTO
   (`name`, `avatarPath`, `color`, `chatType`) или обойти `avatar.setAvatar(...)` —
   `AvatarView` принимает Recipient, потребуется штамм из DTO. Риск: `Recipient` используется
   далеко за пределами списка — менять его глобально нельзя, только точечный фабричный метод.
3. **Batch selection.** `BaseConversationListAdapter` хранит `batchSet` по `chatId` — DTO `id`
   покрывает, но надо проверить базовый класс (selection/selectAll по id).
4. **Обновления по событиям.** Список реагирует на ~12 событий (`MSGS_CHANGED`,
   `INCOMING_MSG`, `MSG_READ`, …) через `loadChatlistAsync()`. Перезагрузка батча должна
   остаться debounced-фоновой, как сейчас.
5. **`isSendingLocation` / `freshness`** — вычисляемые поля; в DTO уже готовы. Legacy
   `chat.isSendingLocations()` — 0 SQL (поле), `getFreshMsgCount` — SELECT; DTO снимает SELECT.

## План по файлам (первый шаг — только рендер списка)

1. `ConversationListAdapter.java`
   - `DcChatlist dcChatlist` → `List<ChatListItemFetchResult> items` (или кеш по id);
   - `getItemCount` → `items.size()`; `getItemId` → `item.id`; `getItemViewType` → `item.id`
     (спец. id) / `kind`;
   - `onBindViewHolder` → читать DTO, передавать в `bind(...)` вместо `ThreadRecord`+`DcLot`;
   - `changeData(DcChatlist)` → `changeData(List<ChatListItemFetchResult>)`.
2. `ConversationListFragment.java`
   - `loadChatlist()`: вместо `getChatlist(...)` → `getChatlistEntries(...)` → id,
     затем `getChatlistItemsByEntries(ids)` в фоне → `changeData(items)` на main;
   - сохранить debounce (`loadChatlistAsync`) и обработку событий как есть.
3. `ConversationListItem.java`
   - `bind(...)` принимать DTO (или облегчённый `ThreadRecord`, собранный из DTO);
   - заменить чтение `dcSummary`/`recipient` на поля DTO (`summaryText1/2`, `summaryStatus`,
     `lastUpdated`, `freshMessageCounter`, `name`, `avatarPath`, `color`, `chatType`, `freshness`).
4. `ThreadRecord.java` / `DcHelper.getThreadRecord`
   - убрать `DcLot dcSummary`; тело/дата/unread брать из DTO. Либо ввести легковесный
     `ThreadRecord.fromChatListItem(...)`.
5. `AvatarView`/Recipient-штамм — минимальный фабричный метод «Recipient из ChatListItem»
   (имя + аватар + цвет + тип), не трогая глобальный `Recipient`.

## Что НЕ делаем в первом шаге

- Поиск (`SearchViewModel`/`SearchResult`/`SearchListAdapter`) и shared-chats (`ProfileAdapter`)
  и `DirectShareUtil` — оставляем на `DcChatlist`, переносим вторым заходом (зона локализована
  и не на горячем пути списка).
- Оптимизацию SQL в ядре (batch-SELECT) — это направление 3 роадмапа, отдельный форк `chatmail/core`.

## Ограничение среды

Android SDK/NDK в этой среде нет — сборка/проверка на стороне заказчика. Изменения пишутся
минимальными шагами.
