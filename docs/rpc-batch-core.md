# Направление 3 — оптимизация БД в ядре (batch-SELECT)

> Часть глобального плана — см. `rpc-batch-roadmap.md` (архитектура, термины, порядок).
> Это третий и последний пункт: делает первые два направления по-настоящему быстрыми.
> Ветка: `feat/rpc-batch-migration` (Android-клиент). Изменения здесь — в форке `chatmail/core`.

## TL;DR

Батчевые JSON-RPC методы (`getMessages`, `getChatlistItemsByEntries`), которые переводят
клиент на DTO (направления 1–2), внутри ядра реализованы **циклом по id**: на каждый
элемент — отдельные SELECT'ы, на сообщение ~8, на чат ~10. То есть батч сейчас срезает
только транспорт (round-trip/JNI/блокировки main), но не число SQL-запросов. Чтобы батч
резал и БД, надо заменить циклы на один `SELECT … WHERE id IN (…)` с пакетными подзапросами.

## Контекст

Ядро (`chatmail/core`, Rust) хранит всё состояние в SQLite. Слой JSON-RPC
(`deltachat-jsonrpc`) — тонкая обёртка над внутренним API ядра (`deltachat`). Внутренние
функции чтения спроектированы под «один объект → один SELECT» (`load_from_db`, `get_by_id`,
`get_fresh_msg_cnt` …). Батчевые методы RPC просто вызывают их в цикле — отсюда N+1 problem:
на N элементов выполняется N × K запросов.

**N+1 problem** — антипаттерн: один запрос на список id, затем по запросу на каждый id.
Лечится заменой на один запрос с `WHERE id IN (…)` и группировкой (`GROUP BY`).

Ниже — факты из исходников ядра (пути и строки от версии core 2.67).

---

## Где сейчас циклы и SELECT'ы

### 1. `get_messages` — цикл по id

`deltachat-jsonrpc/src/api.rs:1495`:
```rust
async fn get_messages(&self, account_id: u32, message_ids: Vec<u32>) -> Result<HashMap<u32, MessageLoadResult>> {
    let mut messages = HashMap::new();
    for message_id in message_ids {
        let message_result = MessageObject::from_msg_id(&ctx, MsgId::new(message_id)).await;
        ...
    }
    Ok(messages)
}
```

### 2. `MessageObject::from_msg_id` — ~8 SELECT на одно сообщение

`deltachat-jsonrpc/src/api/types/message.rs:143`. На каждое сообщение выполняется:

| # | Операция | SQL |
|---|---|---|
| 1 | `Message::load_from_db_optional` | SELECT (с `LEFT JOIN chats` + `msgs_mdns`) |
| 2 | `Contact::get_by_id(from_id)` — отправитель | SELECT |
| 3 | `ContactObject::try_from_dc_contact` — обогащение | SELECT (аватар/профиль) |
| 4 | `message.get_filebytes` | SELECT (размер файла) |
| 5 | `message.parent` → `rfc724_mid_exists` + `load_from_db_optional` | 2 SELECT |
| 6 | `message.quoted_message` → снова `parent` | +2 SELECT (если есть цитата) |
| 7 | `Contact::get_by_id` — автор цитаты | SELECT |
| 8 | `quote.get_file` — картинка цитаты | SELECT (если image/gif/sticker) |
| 9 | `get_msg_reactions` | SELECT |

Сам `load_from_db_optional` (`src/message.rs:488`) — это:
```sql
SELECT … FROM msgs m LEFT JOIN chats c ON c.id=m.chat_id LEFT JOIN msgs_mdns mdns ON mdns.msg_id=m.id WHERE m.id=?
```

### 3. `get_chatlist_items_by_entries` — цикл по id

`deltachat-jsonrpc/src/api.rs:787`:
```rust
for &entry in entries.iter() {
    result.insert(entry, get_chat_list_item_by_id(&ctx, entry).await ...);
}
```

### 4. `get_chat_list_item_by_id` — ~9–11 SELECT на один чат

`deltachat-jsonrpc/src/api/types/chat_list.rs`. На каждый чат:

| # | Операция | SQL |
|---|---|---|
| 1 | `get_fresh_msg_cnt` | SELECT COUNT |
| 2 | `get_last_message_for_chat` | SELECT (последний msg id) |
| 3 | `Chat::load_from_db` | SELECT |
| 4 | `Chatlist::get_summary2` | SELECT (lastmsg) + contact + reactions |
| 5 | `chat.get_profile_image` | SELECT |
| 6 | `Message::load_from_db_optional` (последнее сообщение — **дубль** п.4) | SELECT |
| 7 | `get_chat_contacts` | SELECT |
| 8 | `Contact::get_by_id` (freshness) | SELECT |
| 9 | `chat.get_color` | SELECT |
| 10 | `chat.is_encrypted` | SELECT |
| 11 | `chat.is_self_in_chat` | SELECT |

### 5. `get_fresh_msg_cnt` — SELECT на чат, с комментарием авторов

`src/chat.rs:915`. Комментарий прямо описывает N+1:
> "…query-time … have to be multiplied by the number of items shown at once on the chatlist".

Индекс `(state, hidden, chat_id)` уже есть — батч-запрос должен его использовать.

### 6. `get_last_message_for_chat` — SELECT на чат

`src/chatlist.rs:437`:
```sql
SELECT id FROM msgs WHERE chat_id=?2 AND (hidden=0 OR state=?1) ORDER BY timestamp DESC, id DESC LIMIT 1
```

---

## Что менять (по функции)

### A. Батч-чтение сообщений

Ввести `Message::load_many(context, &[MsgId])` — один запрос:
```sql
SELECT … FROM msgs m LEFT JOIN chats c ON c.id=m.chat_id LEFT JOIN msgs_mdns mdns ON mdns.msg_id=m.id
WHERE m.id IN (…ids…) AND chat_id != 3
```
и переписать `get_messages` с цикла на `load_many`.

### B. Батч контактов

`Contact::get_by_id` → `Contact::load_many(&[ContactId])`: `SELECT … WHERE id IN (…)`.
Применяется к отправителям, авторам цитат, dm-контактам, freshness.

### C. Батч реакций

`get_msg_reactions` → вариант `WHERE msg_id IN (…)` одним запросом (или `GROUP BY msg_id`),
вместо по одному на сообщение.

### D. Батч счётчиков непрочитанных (список чатов)

Вместо N × `get_fresh_msg_cnt`:
```sql
SELECT chat_id, COUNT(*) FROM msgs WHERE chat_id IN (…) AND state=? AND hidden=0 GROUP BY chat_id
```
(переиспользовать индекс `(state, hidden, chat_id)`).

### E. Батч последнего сообщения по чату

Вместо N × `get_last_message_for_chat`:
```sql
SELECT chat_id, id FROM msgs WHERE chat_id IN (…) AND (hidden=0 OR state=?)
  AND id IN (SELECT MAX(id) …)  -- либо оконная функция row_number() OVER (PARTITION BY chat_id ORDER BY timestamp DESC, id DESC)
```

### F. Батч `get_chat_list_item_by_id`

Собрать шаги A–E в один проход: чаты (`WHERE c.id IN`), последние сообщения (E),
счётчики (D), контакты (B), профильные картинки — пакетно, без дублей (убрать повторную
загрузку последнего сообщения, см. п.6 таблицы выше).

### G. Цитата и родитель (п.5–8 таблицы сообщений)

Каскад `quoted_message → parent → load_from_db` на каждое сообщение — самый дорогой.
Два пути:
1. **Отложить.** DTO `MessageQuote` уже содержит `text` и `authorDisplayName` — рендеру
   хватает без полной загрузки цитируемого сообщения. Полную догрузку (`get_message` по id)
   делать только по клику на цитату.
2. **Каскадный батч.** Один `WHERE id IN (…)` по всем `parent`/`quote` id собранным за проход.

Рекомендация: начать с варианта 1 (меньше риска, выигрыш уже большой).

---

## План по файлам ядра

1. `deltachat/src/message.rs`
   - добавить `load_many(&[MsgId])`; `get_filebytes`/`parent`/`quoted_message` — оставить
     поштучными, убрать их из горячего пути `from_msg_id` (или сделать каскадный батч).
2. `deltachat/src/contact.rs`
   - добавить `load_many(&[ContactId])`.
3. `deltachat/src/chat.rs`
   - `get_fresh_msg_cnt` → пакетный вариант с `GROUP BY chat_id`.
4. `deltachat/src/chatlist.rs`
   - `get_last_message_for_chat` → пакетный; `get_summary2` → пакетный для списка чатов.
5. `deltachat-jsonrpc/src/api.rs`
   - `get_messages` — убрать цикл, использовать `load_many` + пакетные контакты/реакции.
   - `get_chatlist_items_by_entries` — убрать цикл, использовать пакетный `get_chat_list_item`.
6. `deltachat-jsonrpc/src/api/types/message.rs` и `chat_list.rs`
   - переписать `from_msg_id`/`get_chat_list_item_by_id` на батч-версии.

## Сложности и риски

- **N+1 — не только в JSON-RPC.** Внутренние `load_from_db`/`get_by_id` тоже поштучные;
  батчификация затрагивает ORM-слой (`Sql::query_map` и друзья), а не только api.rs.
- **Согласованность снимка.** Батч из нескольких запросов должен читать согласованное
  состояние (одна транзакция read-only), иначе между запросами данные могут поменяться.
- **Индексы.** Пакетные `WHERE id IN`/`GROUP BY` должны попадать в существующие индексы
  (`(state, hidden, chat_id)` для fresh-счётчика и др.), иначе запросы деградируют.
- **Вложенные догрузки (quote/parent/reactions).** Их проще отложить (вариант 1 в G), чем
  батчифицировать каскад — DTO уже покрывает текст цитаты.
- **Пересборка натива.** Изменения ядра требуют Rust toolchain + NDK и пересборки `.so`;
  это не собирается в текущей среде — правки идут в форк `chatmail/core`.

## Связь с направлениями 1–2

- Без этого пункта направления 1–2 дают только «main не блокируется» (jank/ANR уходят),
  но число SQL остаётся прежним (даже растёт из-за обогащения DTO).
- С этим пунктом батч начинает резать именно БД — то есть реализуется главный тезис
  плана: узкое место — SQL, а не копирование в памяти или сериализация.
- Порядок: пункты 1–2 можно делать и мерджить независимо; пункт 3 — отдельный форк ядра,
  но его выигрыш масштабируется на оба.

## Ограничение среды

Rust toolchain + NDK для пересборки ядра в этой среде отсутствуют. Правки пишутся в форк
`chatmail/core`; сборка/тесты — на стороне заказчика или в CI форка.
