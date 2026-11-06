# Vulnerability Tracker API

REST API на ASP.NET Core 8 для учёта уязвимостей и фонового сканирования репозиториев. Поддерживает очередь, ограничение параллельности и отмену задач. Данные хранятся в памяти и теряются при перезапуске.

**Сканер демонстрационный:** не скачивает репозиторий и не анализирует код, а возвращает синтетический результат с `isDemonstration: true`. Реальный сканер можно подключить через `IRepositoryScanner`. Результаты сканирования не добавляются автоматически в реестр уязвимостей. Сервис предназначен для локальной демонстрации: нет аутентификации, ограничения размера очереди и очистки истории.

## Быстрый запуск

Нужен .NET 8 SDK; для восстановления пакетов - доступ к NuGet. База данных и Docker не требуются. Выполните из папки с `VulnerabilityTracker.sln`:

```bash
dotnet restore VulnerabilityTracker.sln
dotnet build VulnerabilityTracker.sln -c Release --no-restore
dotnet test VulnerabilityTracker.sln -c Release --no-build
dotnet run --project src/VulnerabilityTracker.Api --launch-profile http
```

```Через Visual Studio
- Откройте VulnerabilityTracker.sln
- Назначьте VulnerabilityTracker.Api запускаемым проектом
- Выбери профиль http
- Нажми F5 для запуска с отладкой или Ctrl+F5 без неё
```

- Swagger: http://localhost:5080/swagger
- OpenAPI: http://localhost:5080/swagger/v1/swagger.json
- Базовый адрес API: http://localhost:5080/api

Остановка - `Ctrl+C`: новые задания больше не принимаются, ожидающие и выполняющиеся отменяются. Бюджет корректной остановки хоста - 30 секунд; это не принудительное прерывание произвольного кода.

## Технологии и зависимости

C# 12, .NET 8, ASP.NET Core Controllers, встроенный DI, Swashbuckle (Swagger/OpenAPI). Фоновая обработка: `BackgroundService`, `Channel`, `CancellationToken`; хранение и синхронизация: `ConcurrentDictionary` и блокировка состояния каждой задачи. Время и логи: `TimeProvider`, `ILogger`.

Тесты: xUnit и `Microsoft.AspNetCore.Mvc.Testing`. Управляемый тестовый сканер использует `TaskCompletionSource` без случайных задержек. Проверяются валидация, HTTP-контракты, состояния, очередь, параллельность, отмена, гонки и остановка.

## Структура и расположение файлов

Пути компонентов ниже указаны относительно `src/VulnerabilityTracker.Api/`.

| Путь | Назначение |
|---|---|
| `Program.cs` | DI, middleware и запуск |
| `Controllers/`, `DTOs/` | Эндпоинты, запросы и валидация |
| `Models/` | Уязвимости, задачи и результаты |
| `Interfaces/`, `Repositories/` | Контракты и хранилище уязвимостей в памяти |
| `Services/` | Учёт уязвимостей, реестр и очередь сканирований, демонстрационный сканер |
| `Workers/` | Фоновые потребители очереди |
| `Middleware/` | Обработка исключений |
| `appsettings.json` | Настройки приложения |
| `Properties/launchSettings.json` | Профиль локального запуска |

В корне: `VulnerabilityTracker.sln`, `README.md`, `requests.http` с примерами запросов, `.gitignore` и `.github/workflows/dotnet.yml` для CI. Тестовый проект - `tests/VulnerabilityTracker.Tests/`. Каталоги `bin/`, `obj/`, `TestResults/` создаются инструментами .NET.

## Контракты API

JSON использует camelCase, перечисления - строки, ID - GUID, время - UTC в ISO 8601.

| Метод и путь | Успешный ответ | Ошибки |
|---|---|---|
| `POST /api/vulnerabilities` | `201`, `{ "id": "…" }`, заголовок `Location` | `400` |
| `GET /api/vulnerabilities/{id}` | `200`: `id`, `name`, `severity`, `createdAt` | `400`, `404` |
| `POST /api/scans` | `202`, `{ "id": "…" }`, `Location`; без ожидания сканирования | `400`, `503` при остановке |
| `GET /api/scans` | `200`, массив задач или `[]` | — |
| `GET /api/scans/{id}` | `200`: `id`, `repositoryUrl`, `status`, `createdAt`, `updatedAt`, `result`, `error` | `400`, `404` |
| `POST /api/scans/{id}/cancel` | `200`: `id`, `requestedAt`, `success`, `status`, `message` | `400`, `404` |

**Входные данные:** `name` обязателен, не состоит только из пробелов, длина до 200 символов проверяется до удаления пробелов по краям. Ограничение задано в `Vulnerability.MaxNameLength`. Повторяющиеся названия разрешены. `severity`: `Low`, `Medium`, `High`, `Critical`; числа не принимаются.

`repositoryUrl` - абсолютный HTTP(S)-адрес до 2048 символов без пробелов, логина/пароля, query и fragment. SSH/git URL не поддерживаются; доступность репозитория не проверяется.

**Состояния:** `Queued → Running → Completed/Failed/Cancelled`; ожидающую задачу также можно отменить. Терминальные состояния не изменяются. `result` заполнен только для `Completed` и содержит `isDemonstration`, `summary`, `findings` (код, описание и критичность находки). `error` заполнен только для `Failed`; иначе эти поля равны `null`. `updatedAt` меняется только при смене состояния.

Список отсортирован: `Running → Queued → Completed → Failed → Cancelled`, внутри группы - новые по `createdAt` первыми, при равенстве - по `id`. Фильтров и пагинации нет. Состояние каждой задачи читается согласованно, но весь список не является снимком на один момент времени.

Ошибки возвращаются в формате `ProblemDetails`, ошибки валидации - `ValidationProblemDetails` со словарём `errors`. `400` означает неверные данные, `404` - неизвестный ID, `500` - непредвиденную серверную ошибку без стека исключения. Статус сканирования `Failed` возвращается с HTTP `200`: запрос состояния выполнен успешно.

## Базовые сценарии использования

В Swagger выберите метод, нажмите **Try it out**, заполните данные и нажмите **Execute**. В последующих запросах используйте ID из ответа сервера.

**1. Добавить и прочитать уязвимость.** Отправьте `POST /api/vulnerabilities`:

```json
{ "name": "SQL-инъекция", "severity": "Critical" }
```

Сервис назначит ID и дату создания. Вызовите `GET /api/vulnerabilities/{id}` для просмотра записи.

**2. Запустить сканирование.** Отправьте `POST /api/scans`:

```json
{ "repositoryUrl": "https://gitlab.com/example/project.git" }
```

Проверяйте `GET /api/scans/{id}` до завершения. При `Completed` появится демонстрационный результат; все задачи доступны через `GET /api/scans`.

**3. Проверить очередь.** При `MaxConcurrentScans = 1` запустите две задачи до завершения первой: первая будет `Running`, вторая - `Queued`. После освобождения слота вторая запускается автоматически. Фоновый сервис использует заданное число потребителей очереди.

**4. Отменить задачу.** Вызовите `POST /api/scans/{id}/cancel` без тела. Для ожидающей или выполняющейся задачи возвращаются `success: true` и `Cancelled`. Отмена передаётся через `CancellationToken`; слот освобождается после фактического выхода сканера. Если завершение успело раньше отмены, оно сохраняется. Повторная отмена или отмена завершённой задачи возвращает `success: false` и текущий статус.

**5. Проверить валидацию.** Пустое название, неизвестная критичность или некорректный URL дают `400`; неизвестный корректный GUID — `404`. Ошибка самого сканера проверяется тестами с исключением в тестовой реализации.

## Настройки

Файл: `src/VulnerabilityTracker.Api/appsettings.json`. Значения ниже соответствуют последнему варианту проекта.

| Параметр | Значение | Назначение |
|---|---|---|
| `Scanning:MaxConcurrentScans` | `2` | Максимум одновременно выполняющихся сканирований; диапазон 1–64 |
| `Scanning:DemoDelayMilliseconds` | `60000` | Демонстрационная задержка в миллисекундах (60 секунд); диапазон 1–60000 |
| `Logging:LogLevel:Default` | `Information` | Минимальный уровень логов по умолчанию |
| `Logging:LogLevel:Microsoft.AspNetCore` | `Warning` | Минимальный уровень логов ASP.NET Core |
| `Logging:LogLevel:Microsoft.AspNetCore.HttpLogging.HttpLoggingMiddleware` | `Information` | Включает HTTP-логи: метод, путь, статус и длительность |
| `AllowedHosts` | `*` | Разрешает любое значение заголовка `Host`; не управляет CORS |

Логи выводятся в консоль, включая события и ошибки сканирования; тела HTTP-запросов не логируются. Неверные значения `Scanning` препятствуют запуску. После изменения параметров сканирования перезапустите приложение.
