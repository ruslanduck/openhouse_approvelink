# OPENHOUSE — Client Proof Approval

Страница одобрения пруфов. **Хостинг — GitHub Pages, бэкенд — Make (вебхуки).**
Ключ Airtable нигде в этом репозитории не хранится — он живёт внутри Make.

```
openhouse-proof/
├─ approval/
│  └─ index.html        ← страница, открывается на client.byopenhouse.com/approval?id=…
├─ make/
│  └─ scenarioA_response_template.json   ← JSON для ответа вебхука "чтение пруфа"
└─ CNAME                ← домен для GitHub Pages
```

## Как это работает
1. В Airtable формула `approval_link` уже даёт `client.byopenhouse.com/approval?id=RECORD_ID()`.
2. Клиент открывает ссылку → страница берёт `id` из URL → стучит в **вебхук Make A** → Make достаёт запись из Airtable → возвращает JSON → страница рисует пруф.
3. Клиент жмёт **Approve** или **Make changes** → страница шлёт в **вебхук Make B** → Make обновляет запись в Airtable (Status, дата, заметка) и уведомляет Rasa.

## Что нужно один раз настроить

### 1. GitHub Pages
- Создать репозиторий, залить эти файлы.
- Settings → Pages → Source = ветка `main`, папка `/ (root)`.
- В поле Custom domain указать `client.byopenhouse.com` (файл `CNAME` уже в репо).
- У регистратора домена byopenhouse.com добавить CNAME-запись: `client` → `<твой-юзернейм>.github.io`.
  *(Сейчас этот субдомен указывает на Bubble — перенаправляешь его на GitHub.)*

### 2. Make — Сценарий A (чтение пруфа)
- Модуль **Webhooks → Custom webhook** → скопировать URL.
- Модуль **Airtable → Get a record**: Base `appksApRmCQWucB5I`, Table `tblOOr2B1QVDIkumP`, Record ID = `{{webhook.id}}`.
- Модуль **Webhooks → Webhook response**: Status 200, заголовок `Content-Type: application/json`, тело — из `make/scenarioA_response_template.json` (значения перетащить из модуля Airtable).

### 3. Make — Сценарий B (ответ клиента)
- Модуль **Custom webhook** → скопировать URL. Принимает `{ id, action, note, categories }`.
- Модуль **Router** на два пути:
  - **action = "approve"** → Airtable **Update a record** (Record ID `{{webhook.id}}`):
    - `Status` = `<СТАТУС "ОДОБРЕНО">`
    - `Work Approval Date` = `{{now}}`
  - **action = "changes"** → Airtable **Update a record**:
    - `Back to Work Note` = `{{webhook.note}}`  (+ `{{join(webhook.categories; ", ")}}`)
    - `Status` = `<СТАТУС "НУЖНЫ ПРАВКИ">`
- (По желанию) ещё модуль Slack/Email → уведомить Rasa.
- Модуль **Webhook response**: тело `{"ok":true}`.

### 4. Привязать страницу к Make
В `approval/index.html`, блок CONFIG сверху скрипта:
```js
const USE_MOCK         = false;                  // выключить демо
const WEBHOOK_PROOF    = "URL вебхука Сценария A";
const WEBHOOK_DECISION = "URL вебхука Сценария B";
```
Запушить → GitHub Pages обновится сам.

### 5. Проверка
Открыть `client.byopenhouse.com/approval?id=rectkLj0T3LCQiOlI` — должен подтянуться реальный пруф из Airtable.

## Заметки
- Вебхук A публичный: данные пруфа доступны всем, у кого есть `id`. ID нечитаемые (`rec…`) — та же защита, что у нынешней Bubble-ссылки.
- Ссылки на картинки из Airtable (attachment) временные — открывать пруф лучше вскоре после отправки. Если нужно надолго — складывать мокап-PNG в постоянное хранилище.
- `productName` и мерки пока берутся как есть/константой — если у тебя они в Product DB → Dye Lines, потом завяжем на эти поля.
