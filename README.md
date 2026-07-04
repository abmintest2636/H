Проведи аудит file upload pipeline в кодобазі.
DevOps каже що зараз так:
файл повністю буферизується в пам'яті
ключ шифрування завантажується на кожен запит заново
десь є base64 в pipeline шифрування
зашифрований буфер цілком летить на S3
Знайди відповідний код і перевір чи це правда. По кожному пункту — що знайшов і чи збігається з описом.



Ти працюєш з Node.js/TypeScript backend. Проведи оптимізацію file upload pipeline на основі аудиту.
Що знайдено в коді:
Buffer.from(attachment.Data, 'base64') — файл повністю буферизується в пам'яті на вході
scryptSync(passCode, salt, 32) — синхронний KDF викликається на кожен upload і download, CPU-cost на кожен запит
IV зберігається в БД як base64 і щоразу конвертується туди-назад
Upload через encryptDataStream вже стримується, але оригінальний буфер все одно в пам'яті до початку стриму
renameDocument — повний регрес: getObjectAsBuffer тягне весь файл з S3 в Buffer і uploadToExistingBucket відправляє цілком
Що треба зробити:
Замінити буферизацію на stream pipeline від вхідного файлу до S3
Закешувати результат scryptSync — ключ для одного файлу не змінюється, не треба рахувати його двічі
Прибрати зайві base64 конвертації де можливо
Переписати renameDocument щоб не тягнути файл в пам'ять — використати S3 server-side copy або stream
Вимоги:
TypeScript
Не ламати існуючу логіку шифрування (AES pipeline залишається)
Зберегти сумісність з існуючою схемою БД (IV як base64 в БД — можна залишити, просто не конвертувати зайвий раз в пам'яті)
Показати diff або до/після для кожного пункту


Зроби новий endpoint для file upload який приймає multipart/form-data. Pipeline: busboy → AES transform → S3 multipart upload. Старий base64-JSON endpoint не чіпай, він залишається для мобілки.


---

Зроби #1, #3, #7.

#1 — team lookup один раз, передай teamId у всі persist-виклики замість N запитів.
#3 — Promise.all для незалежних перевірок в контролері.
#7 — тюнінг multipart: partSize 8-16MB, queueSize 6-8.

---

Шукай далі що можна зробити на сервері без архітектурних змін і без девопса. #2, #4, #6 ще не робили.


Зроби паралельну відправку файлу чанками з браузера. Браузер ріже файл на N частин і відправляє їх паралельно на сервер. Сервер збирає частини в правильному порядку, шифрує і заливає в S3.
Старий endpoint не чіпай — новий флоу окремо. Порядок частин має зберігатись для коректного шифрування.


Зараз браузер відправляє файл як один великий запит. Сервер отримує його, шифрує і заливає в S3 паралельними чанками (те що ми вже налаштували).

Ідея — браузер сам ріже файл на частини і відправляє їх паралельно кількома з'єднаннями одночасно. Браузер має кілька TCP каналів і може використати їх всі.



Backend

- Replaced buffering with streaming — files are streamed through AES directly to S3 without being fully loaded into memory.
- Removed per-file team lookup — a single database query is used instead of N queries.
- Parallelized independent validation checks using "Promise.all".
- Tuned S3 multipart upload configuration ("partSize" / "queueSize").
- Added scrypt caching — encryption key is derived only once and reused.
- Optimized "renameDocument" using S3 server-side copy instead of download/upload operations.

Frontend (A1 / A2)

- A1 — Files are passed directly to "FormData" without unnecessary binary string conversion.
- A2 — "File" objects are stored outside of "cloneDeep" to prevent them from being lost during modal editing.


# Задача: PR AI Reviewer — GitHub Actions пайплайн з AI-рев'ю коду

Побудуй повністю робочу систему автоматичного рев'ю Pull Request-ів на GitHub, яка аналізує diff, порівнює зміни з внутрішніми стандартами (SOP) організації, викликає AI (DeepSeek) для аналізу, і публікує фідбек прямо в PR як коментарі.

## Контекст інфраструктури (вже готово, не чіпати)

- GitHub Actions self-hosted runner вже розгорнутий на AWS spot instance (AMI, subnets, security group налаштовані)
- AI-провайдер: DeepSeek API (OpenAI-сумісний формат: `https://api.deepseek.com/chat/completions`, модель `deepseek-chat`)
- SOP-документація вже існує у вигляді `.md` файлів у репозиторії (наприклад: Django/backend, React/frontend, Testing, Security, Performance, Code style)
- Авторизація в GitHub: використовувати стандартний `GITHUB_TOKEN`, який GitHub Actions видає автоматично для кожного workflow run (не потрібен окремий GitHub App з App ID/private key)

## Що потрібно розробити

### 1. GitHub Actions workflow (`.github/workflows/ai-review.yml`)

Тригери:
- `pull_request` types: `opened`, `synchronize`, `reopened`

Джоба запускається на `runs-on: self-hosted`, чекаутить репозиторій з `fetch-depth: 0` (потрібна повна історія для коректного diff), передає секрети (`DEEPSEEK_API_KEY`, `GITHUB_TOKEN`) і номер PR в окремий Python-скрипт.

### 2. Обчислення diff

Порахувати diff тільки змінених файлів/рядків між base-гілкою PR і head-коммітом (не весь файл цілком — для фокусу, швидкості і економії токенів AI). Використати `git diff` між base SHA і head SHA конкретного PR (доступні через контекст GitHub Actions `github.event.pull_request.base.sha` і `github.event.pull_request.head.sha`).

Результат — список змінених файлів з їх diff-патчами (унікальний формат для кожного файлу, зберігаючи оригінальну нумерацію рядків з git diff, вона знадобиться для inline-коментарів).

### 3. Завантаження релевантних SOP

Матчити SOP `.md` файли до типів файлів, змінених у PR (наприклад: `*.py` в backend-папці → `sops/django.md`, `*.tsx`/`*.jsx` → `sops/react.md`, файли тестів → `sops/testing.md`, завжди підключати `sops/security.md` і `sops/performance.md` незалежно від типу файлу). Логіка матчингу має бути легко розширювана (додавання нових SOP-файлів без переписування коду).

### 4. Побудова AI-контексту (промпт-білдер)

Зібрати структурований payload для LLM, що містить:
- diff PR (якщо занадто великий для контекстного вікна DeepSeek — розбити на чанки по файлах або логічних блоках, обробляти послідовно і об'єднати результати)
- релевантні SOP-документи
- шляхи файлів, яких стосується diff
- чіткі інструкції рев'ю: просити модель повернути **строго валідний JSON** у визначеній структурі (без жодного тексту поза JSON), щоб результат можна було детерміновано розпарсити

Формат очікуваної відповіді від AI:
```json
{
  "summary": "Загальний висновок і головні ризики",
  "high_level_issues": [
    {"category": "bug|architecture|sop_violation|security|performance|missing_tests", "description": "..."}
  ],
  "line_comments": [
    {"file": "path/to/file.py", "line": 42, "description": "...", "why_it_matters": "...", "suggested_fix": "..."}
  ]
}
```

### 5. Виклик DeepSeek API

Відправити побудований промпт, обробити відповідь з парсингом JSON (з обробкою помилок — якщо модель повернула невалідний JSON, зробити повторний запит з жорсткішою інструкцією або graceful fallback без падіння всього пайплайна).

### 6. Публікація рев'ю в GitHub

Конвертувати AI-вивід у GitHub PR review через API `POST /repos/{owner}/{repo}/pulls/{pull_number}/reviews`:
- `body` — executive summary + high-level issues списком
- `comments` — масив inline-коментарів, кожен прив'язаний до конкретного файлу і рядка (з урахуванням особливостей GitHub API: позиція рахується відносно diff-патчу, а не абсолютного номера рядка у файлі — це критичний момент, обробити акуратно, з мапінгом номерів рядків з git diff на diff position, якого вимагає GitHub API)
- де можливо, використати suggestion blocks (```suggestion``` синтаксис у тілі коментаря) для запропонованих виправлень коду

### 7. Повторюваність

Оскільки workflow тригериться на `synchronize` (новий push у той самий PR), пайплайн автоматично запускається знову при кожному оновленні PR — додаткової логіки для цього не потрібно, крім того щоб уникнути дублювання старих коментарів (опційно: видаляти/оновлювати попередній AI-review comment перед публікацією нового, через пошук існуючих коментарів від бота за унікальним маркером у тілі коментаря).

## Технічні вимоги

- Мова: Python (скрипти, що викликаються з workflow)
- Структура файлів:
```
.github/workflows/ai-review.yml
scripts/
  run_review.py        # головний entrypoint
  compute_diff.py
  load_sops.py
  build_prompt.py
  call_deepseek.py
  post_review.py
sops/
  django.md
  react.md
  testing.md
  security.md
  performance.md
  code_style.md
```
- Обробка помилок на кожному кроці (мережеві збої, невалідний JSON від AI, відсутні SOP-файли) — пайплайн не повинен падати без пояснювального повідомлення в логах Actions
- Секрети (`DEEPSEEK_API_KEY`, `GITHUB_TOKEN`) передавати через `env`, ніде не хардкодити
- Код має бути читабельний, з докстрінгами на кожну функцію, без зайвої абстракції — це production-скрипт для конкретного пайплайна, не бібліотека

## Результат, який очікується

Кожен PR отримує автоматичне інженерне рев'ю, яке:
- Послідовне для всіх розробників
- Базується на внутрішніх SOP компанії, а не на загальних AI-порадах
- Фокусується тільки на реальному diff
- Публікується прямо в GitHub
- Повторюється при кожному оновленні PR
- Містить summary, high-level issues, inline-коментарі і пропозиції виправлень

Напиши повний робочий код усіх файлів вище, готовий до копіювання в репозиторій і запуску.

