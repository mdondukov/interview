# 03 — Notification System

Подборка вопросов и ответов о проектировании масштабируемой системы уведомлений: многоканальная доставка (push, SMS, email), гарантии доставки, работа с внешними провайдерами (APNS, FCM, Twilio), управление приоритетами и обработка сбоев.

**Навигация:** [Трек System Design](./README.md) · Предыдущая тема: [02-rate-limiter](./02-rate-limiter.md) · Следующая тема: [04-chat-messenger](./04-chat-messenger.md)

---

## Ключевые концепции

Прежде чем переходить к вопросам, разберёмся с базовыми терминами, которые будут встречаться в этом модуле.

**Push notification** — уведомление, инициируемое сервером и отправляемое на устройство пользователя (смартфон, планшет, десктоп) без явного запроса со стороны клиента. Push notifications обеспечивают мгновенное оповещение пользователей о важных событиях в реальном времени. Это ключевой механизм для повышения вовлечённости и удержания пользователей в приложениях.

**APNS** — Apple Push Notification Service, официальный сервис Apple для доставки push-уведомлений на iOS устройства (iPhone, iPad, Apple Watch). Каждое iOS приложение подключается к APNS для получения уведомлений; система уведомлений должна интегрироваться с APNS API и обрабатывать особенности iOS кодирования и сертификатов.

**FCM** — Firebase Cloud Messaging, сервис Google для доставки push-уведомлений на Android устройства. FCM заменил старый Google Cloud Messaging (GCM) и является стандартом для Android. Система уведомлений должна поддерживать отправку через FCM API и обрабатывать Android-специфичные форматы сообщений.

**Adapter pattern** — паттерн проектирования, в котором разные провайдеры (APNS, FCM, Twilio) оборачиваются под единый унифицированный интерфейс. Adapter pattern позволяет менять провайдеров или добавлять новые каналы без изменения основного кода системы уведомлений. Это обеспечивает гибкость и масштабируемость архитектуры.

**Task queue** — очередь асинхронных задач (реализуется с помощью Kafka, RabbitMQ, AWS SQS), в которую помещаются уведомления для последующей обработки. Task queue буферизирует пики нагрузки, позволяет воркерам обрабатывать сообщения независимо и упрощает масштабирование путём добавления новых воркеров. Это критична для высоконагруженных систем.

**At-least-once delivery** — гарантия, что сообщение будет доставлено минимум один раз, но возможны дубликаты. At-least-once delivery проще реализовать, чем exactly-once, и используется, когда дубликаты приемлемы. Часто сочетается с идемпотентностью: приложение должно обработать дубликат без побочных эффектов.

**Exactly-once delivery** — гарантия, что сообщение будет доставлено ровно один раз, без дубликатов. Exactly-once требует дополнительной синхронизации и логики отслеживания (distributed transactions, координаторы), что значительно усложняет архитектуру. Это необходимо для критичных операций, но дорого в реализации.

**Idempotency key** — уникальный идентификатор, отправляемый клиентом с каждым запросом на создание ресурса. Система должна отслеживать эти ключи и гарантировать, что повторный запрос с тем же ключом не создаст дубликат. Idempotency key — механизм реализации идемпотентности для at-least-once систем.

**DLQ (Dead Letter Queue)** — специальная очередь для сообщений, которые не удалось обработать после всех повторных попыток. Сообщения попадают в DLQ по разным причинам: невалидные данные, недоступные внешние сервисы, баги в коде. DLQ позволяет сохранить эти сообщения для последующего анализа и ручного разбора.

**Exponential backoff** — стратегия повторных попыток, где задержка между попытками экспоненциально увеличивается (1с, 2с, 4с, 8с и т.д.). Exponential backoff предотвращает overload upstream сервисов при массовых сбоях, так как не создаёт волну одновременных повторов. Часто добавляется jitter (случайная составляющая) для избежания thundering herd проблемы.

---

## Вопросы и разборы

### 1. Как спроектировать архитектуру системы уведомлений?

**Зачем спрашивают.**
Система уведомлений — критическая часть современных приложений. Интервьюер проверяет умение работать с асинхронной обработкой, очередями сообщений, множественными каналами доставки и масштабированием при высокой нагрузке.

**Короткий ответ.**
Архитектура строится на асинхронной обработке: API принимает запрос на отправку, добавляет его в очередь, воркеры обрабатывают сообщения и отправляют через разные каналы (push, SMS, email). Отдельные очереди для каждого канала позволяют масштабировать независимо.

**Детальный разбор.**

Общая архитектура системы:
```
┌─────────────────────────────────────────────────────────────────────┐
│                      Client Services                                │
│                    (Auth, Feed, Search, etc)                        │
└─────────────────────────┬───────────────────────────────────────────┘
                          │
                          ▼
┌─────────────────────────────────────────────────────────────────────┐
│                      Notification API                               │
│           (POST /notifications, GET /status, etc)                   │
└─────────────────────────┬───────────────────────────────────────────┘
                          │
         ┌────────────────┴────────────────┐
         │                                 │
         ▼                                 ▼
┌──────────────────────┐        ┌──────────────────────┐
│  Notification DB     │        │  Cache (Redis)       │
│  (store + state)     │        │  (preferences,       │
│                      │        │   device tokens)     │
└──────────────────────┘        └──────────────────────┘
         │                                 ▲
         │                                 │
         ▼                                 │
┌──────────────────────────────────────────────────────────┐
│                  Task Queue (Kafka/RabbitMQ)            │
│  ┌───────────────┐ ┌───────────────┐ ┌───────────────┐ │
│  │  Push Queue   │ │  SMS Queue    │ │  Email Queue  │ │
│  └───────────────┘ └───────────────┘ └───────────────┘ │
└──────────┬──────────────────┬──────────────────┬────────┘
           │                  │                  │
    ┌──────▼──────┐    ┌──────▼──────┐    ┌──────▼──────┐
    │ Push Worker │    │ SMS Worker  │    │Email Worker │
    │ (N instances)   │ (M instances)   │(K instances)│
    └──────┬──────┘    └──────┬──────┘    └──────┬──────┘
           │                  │                  │
    ┌──────▼──────┐    ┌──────▼──────┐    ┌──────▼──────┐
    │ APNS/FCM    │    │ Twilio/Nexmo│    │ SendGrid/SES│
    │  External   │    │  External   │    │  External   │
    │ Providers   │    │ Providers   │    │ Providers   │
    └─────────────┘    └─────────────┘    └─────────────┘
```

**Ключевые компоненты:**

1. **Notification API** — HTTP interface для отправки уведомлений
   - Принимает batch запросы на 1000+ пользователей
   - Возвращает notification_id для отслеживания
   - Валидирует preferences и параметры

2. **Task Queue** — асинхронная обработка
   - Разделение по каналам (push/SMS/email)
   - Разделение по приоритету (urgent/normal/low)
   - Буферизация при пиковых нагрузках

3. **Workers** — независимые воркеры для каждого канала
   - Масштабируются независимо
   - Обрабатывают retry логику
   - Обновляют статус в БД

4. **Storage**
   - Notifications table: id, user_id, content, status, created_at
   - User preferences: channels enabled, quiet hours, timezone
   - Device tokens: per-user push device identifiers

**Пример.**

Отправка уведомления:
```python
# API Layer
@app.post("/api/v1/notifications")
async def send_notification(request: NotificationRequest):
    """
    Request:
    {
        "user_ids": ["user123", "user456"],
        "template_id": "welcome_email",
        "data": {"name": "John", "code": "SAVE20"},
        "channels": ["push", "email"],  # optional, use user preferences
        "priority": "high",  # optional
        "scheduled_at": "2024-01-15T10:00:00Z"  # optional
    }
    """

    # Validate request
    if len(request.user_ids) > 10000:
        return APIError("batch too large", 400)

    # Create notification record
    notif_id = str(uuid4())
    notification = {
        "id": notif_id,
        "users": request.user_ids,
        "template_id": request.template_id,
        "data": request.data,
        "channels": request.channels,
        "priority": request.priority or "normal",
        "scheduled_at": request.scheduled_at or now(),
        "status": "queued",
        "created_at": now()
    }

    # Store in DB
    await db.notifications.insert_one(notification)

    # Publish to task queue (one message per channel)
    for channel in (request.channels or user_preferred_channels):
        message = {
            "notification_id": notif_id,
            "user_ids": request.user_ids,
            "channel": channel,
            "priority": request.priority,
            "template_id": request.template_id,
            "data": request.data,
            "scheduled_at": request.scheduled_at
        }
        await queue.publish(f"{channel}_queue", message)

    return {
        "notification_id": notif_id,
        "status": "queued",
        "estimated_delivery_seconds": 5
    }

# Worker Layer
async def push_notification_worker():
    """Обработчик Push уведомлений"""
    while True:
        message = await queue.consume("push_queue")

        try:
            await process_push_notification(message)
            await queue.acknowledge(message)
        except RetryableError as e:
            # Перепубликуем с delay для retry
            message["retry_count"] = message.get("retry_count", 0) + 1
            if message["retry_count"] <= 3:
                delay = 2 ** message["retry_count"]  # exponential backoff
                await queue.publish("push_queue", message, delay=delay)
            else:
                # Отправляем в DLQ
                await queue.publish("dead_letter_queue", message)
                await db.notifications.update_one(
                    {"id": message["notification_id"]},
                    {"$set": {"status": "failed", "error": str(e)}}
                )
        except Exception as e:
            await db.notifications.update_one(
                {"id": message["notification_id"]},
                {"$set": {"status": "failed", "error": str(e)}}
            )

async def process_push_notification(message: dict):
    """Отправка push уведомлений через FCM/APNS"""
    user_ids = message["user_ids"]
    template_id = message["template_id"]
    data = message["data"]

    # Получаем девайс токены пользователей
    device_tokens = await db.device_tokens.find(
        {"user_id": {"$in": user_ids}}
    )

    # Группируем по платформе для эффективной отправки
    ios_tokens = []
    android_tokens = []

    for token_record in device_tokens:
        if token_record["platform"] == "ios":
            ios_tokens.append(token_record)
        else:
            android_tokens.append(token_record)

    # Отправляем через провайдеров
    if ios_tokens:
        await send_apns_batch(ios_tokens, template_id, data)

    if android_tokens:
        await send_fcm_batch(android_tokens, template_id, data)

    # Обновляем статус
    await db.notifications.update_one(
        {"id": message["notification_id"]},
        {"$set": {"status": "sent", "sent_at": now()}}
    )
```

**Типичные ошибки.**
1. **Синхронная отправка в API** — огромная latency при большом количестве пользователей
2. **Единая очередь для всех каналов** — медленный SMS блокирует быструю push доставку
3. **Отсутствие обработки preferences** — спам пользователям, которые отключили уведомления
4. **Нет retry логики** — потеря уведомлений при временных сбоях провайдера
5. **Одинаковое количество воркеров** — нужно больше воркеров для быстрых каналов (push), меньше для медленных (email)

**На интервью.**
Нарисуй архитектуру на доске, объясняя каждый компонент. Обсуди, почему нужны отдельные очереди для каналов. Покажи, как масштабировать при 100M уведомлений/день. Какой trade-off между at-least-once и exactly-once доставкой?

*Частые follow-up вопросы:*
- Как обрабатывать 1 млрд уведомлений в день (масштабирование)?
- Как приоритизировать urgent уведомления над marketing?
- Как отслеживать delivery rate по каналам?
- Как справиться с rate limits провайдеров?

---

### 2. Как интегрироваться с внешними провайдерами (APNS, FCM, Twilio)?

**Зачем спрашивают.**
Вся реальная ценность системы уведомлений зависит от надежной интеграции с провайдерами. Интервьюер проверяет практическое знание особенностей каждого сервиса и обработки их ограничений.

**Короткий ответ.**
Каждый провайдер требует собственной интеграции: APNS использует HTTP/2 с JWT, FCM — REST API с batch поддержкой, Twilio — REST API для SMS. Оборачиваем каждый в adapter паттерн для единого интерфейса.

**Детальный разбор.**

Сравнение провайдеров:
```
┌──────────────┬──────────┬──────────┬──────────┬─────────────┐
│ Провайдер    │ Протокол │ Auth     │ Batch    │ Feedback    │
├──────────────┼──────────┼──────────┼──────────┼─────────────┤
│ APNS (iOS)   │ HTTP/2   │ JWT      │ Нет      │ Webhook     │
│ FCM          │ HTTP v1  │ OAuth2   │ Да (500) │ Webhook     │
│ Twilio (SMS) │ HTTPS    │ API Key  │ Нет      │ Callback    │
│ SendGrid     │ HTTPS    │ API Key  │ Да       │ Webhook     │
└──────────────┴──────────┴──────────┴──────────┴─────────────┘
```

**APNS (Apple Push Notification service):**
```
Характеристики:
- Требует сертификат (key + cert для auth)
- HTTP/2 долгоживущие соединения
- Device token: 64 hex chars
- Payload: JSON до 4KB (iOS) / 2KB (watchOS)
- Response: 200 = success, иначе error code

Обработка ошибок:
- 400: InvalidPayload, BadTopicId
- 403: Forbidden (auth failed)
- 410: InvalidToken (девайс удален)
- 429: TooManyRequests (rate limit)
- 500-599: Server error (retry)
```

**FCM (Firebase Cloud Messaging):**
```
Характеристики:
- OAuth2 authentication (service account JSON)
- REST API: POST https://fcm.googleapis.com/v1/projects/{project}/messages:send
- Device token: registration token (~160 chars)
- Payload: JSON до 4KB
- Batch API: до 500 messages per request

Преимущества:
- Batch отправка эффективнее
- Topic-based messaging для millions пользователей
- Conditioning (отправка если версия > X)
```

**Adapter pattern:**
```python
# Интерфейс для всех провайдеров
class PushProvider(ABC):
    async def send(
        self,
        tokens: List[str],
        notification: dict
    ) -> SendResult:
        """Отправить push, вернуть результаты"""
        pass

# APNS адаптер
class APNSProvider(PushProvider):
    def __init__(self, key: str, cert: str, team_id: str):
        self.client = APNSClient(
            certificate=cert,
            private_key=key,
            default_error_timeout=10
        )
        self.team_id = team_id

    async def send(self, tokens: List[str], notification: dict) -> SendResult:
        title = notification.get("title")
        body = notification.get("body")

        payload = Payload(
            alert=PayloadAlert(
                title=title,
                body=body
            ),
            sound="default",
            badge=notification.get("badge", 1),
            custom=notification.get("data", {})
        )

        results = SendResult()

        for token in tokens:
            try:
                # APNS отправляет один по одному
                await asyncio.to_thread(
                    self.client.send_notification,
                    token,
                    payload
                )
                results.success.append(token)
            except APNSException as e:
                if e.status == 410:
                    # Invalid token — нужно удалить
                    results.invalid_tokens.append(token)
                else:
                    results.failed.append((token, str(e)))

        return results

# FCM адаптер
class FCMProvider(PushProvider):
    def __init__(self, credentials_json: str):
        self.project_id = load_json(credentials_json)["project_id"]
        self.credentials = load_service_account_credentials(credentials_json)

    async def send(self, tokens: List[str], notification: dict) -> SendResult:
        results = SendResult()

        # Батчим по 500
        for batch in chunks(tokens, 500):
            messages = []
            for token in batch:
                messages.append(
                    messaging.Message(
                        notification=messaging.Notification(
                            title=notification.get("title"),
                            body=notification.get("body")
                        ),
                        data=notification.get("data", {}),
                        android=messaging.AndroidConfig(
                            ttl=duration_pb2.Duration(seconds=86400),
                            priority="high"
                        ),
                        apns=messaging.APNSConfig(
                            headers={
                                "apns-priority": "10"
                            }
                        ),
                        token=token
                    )
                )

            try:
                resp = await asyncio.to_thread(
                    self.app.send_all,
                    messages
                )

                for i, send_response in enumerate(resp.responses):
                    if send_response.success:
                        results.success.append(batch[i])
                    else:
                        if "INVALID_ARGUMENT" in str(send_response.exception):
                            results.invalid_tokens.append(batch[i])
                        else:
                            results.failed.append((batch[i], str(send_response.exception)))

            except Exception as e:
                results.failed.extend([(t, str(e)) for t in batch])

        return results

# Twilio SMS адаптер
class TwilioProvider:
    def __init__(self, account_sid: str, auth_token: str, from_number: str):
        self.client = Client(account_sid, auth_token)
        self.from_number = from_number

    async def send(self, phone_numbers: List[str], message: str) -> SendResult:
        results = SendResult()

        for phone in phone_numbers:
            try:
                resp = await asyncio.to_thread(
                    self.client.messages.create,
                    body=message,
                    from_=self.from_number,
                    to=phone
                )

                if resp.status in ["queued", "sending", "sent"]:
                    results.success.append(phone)
                else:
                    results.failed.append((phone, f"status={resp.status}"))

            except TwilioRestException as e:
                if e.status == 400:  # Invalid phone number
                    results.invalid_tokens.append(phone)
                else:
                    results.failed.append((phone, str(e)))

        return results
```

**Обработка feedback и invalidity:**
```python
# Webhook от APNS/FCM для обновления состояния девайсов
@app.post("/webhooks/push/feedback")
async def push_feedback_webhook(request: Request):
    """
    Получаем feedback от провайдеров:
    - Девайс удален (410 APNS, invalid token FCM)
    - Ошибка аутентификации
    - Rate limit
    """
    provider = request.headers.get("X-Provider")  # apns, fcm
    data = await request.json()

    if provider == "apns":
        for feedback in data.get("feedback", []):
            token = feedback["token"]
            reason = feedback["reason"]  # "BadDeviceToken", "Uninstalled"

            # Удаляем невалидный токен
            await db.device_tokens.delete_one({
                "device_token": token,
                "platform": "ios"
            })

    elif provider == "fcm":
        for result in data.get("results", []):
            if "error" in result:
                error = result["error"]
                if error in ["INVALID_ARGUMENT", "NOT_FOUND"]:
                    token = result["token"]
                    await db.device_tokens.delete_one({
                        "device_token": token,
                        "platform": "android"
                    })

    return {"status": "ok"}
```

**Типичные ошибки.**
1. **Синхронные HTTP запросы** — блокирует worker, снижает throughput в 10+ раз
2. **Отсутствие retry при rate limit** — потеря нотификаций при peak нагрузке
3. **Игнорирование invalid tokens** — продолжаем отправлять мертвым девайсам
4. **Отсутствие circuit breaker** — одна перегруженная APNS блокирует весь сервис
5. **Хранение credentials в коде** — security issue, нужен secrets manager

**На интервью.**
Объясни различия между APNS и FCM. Как ты будешь обрабатывать invalid tokens? Какой rate limit у Twilio и как его не превысить? Как сделать failover между провайдерами?

*Частые follow-up вопросы:*
- Как реализовать circuit breaker для провайдера?
- Как обрабатывать rate limits без потери нотификаций?
- Как интегрировать DLQ для мертвых девайсов?

---

### 3. Как гарантировать доставку уведомлений (at-least-once, exactly-once)?

**Зачем спрашивают.**
Доставка уведомлений — критический аспект системы. Интервьюер проверяет понимание trade-off между гарантиями доставки и сложностью реализации, а также практическое знание идемпотентности.

**Короткий ответ.**
At-least-once: сохраняем состояние в БД перед отправкой, используем retry. Exactly-once: добавляем idempotency key в каждое сообщение, провайдеры дедупликируют по этому ключу.

**Детальный разбор.**

Уровни гарантии:
```
┌─────────────────────┬──────────────┬──────────────┬─────────┐
│ Гарантия            │ Дубликаты    │ Сложность    │ Когда   │
├─────────────────────┼──────────────┼──────────────┼─────────┤
│ At-most-once        │ Нет          │ Простая      │ N/A     │
│ At-least-once       │ Возможны     │ Средняя      │ Usual   │
│ Exactly-once        │ Нет          │ Сложная      │ Critical│
└─────────────────────┴──────────────┴──────────────┴─────────┘
```

**At-least-once delivery:**

Гарантия: каждое сообщение доставляется минимум один раз, может быть дубликат.

Механизм:
```
1. Сохраняем notification в БД со статусом "pending"
2. Отправляем в provider
3. Если успех — обновляем статус на "sent" + delivered_at
4. Если ошибка — retry с exponential backoff
5. После N неудачных попыток → status = "failed"

┌──────────────┐     ┌──────────────┐     ┌──────────────┐
│  DB: pending │────▶│   Provider   │────▶│  DB: sent    │
└──────────────┘     └──────┬───────┘     └──────────────┘
                            │
                    ┌───────▼────────┐
                    │ Fail/Timeout   │
                    └────┬───────────┘
                         │
                   ┌─────▼──────┐
                   │ Retry after│ (1s, 2s, 4s, ...)
                   │ exponential │
                   │ backoff    │
                   └─────┬──────┘
                         │
                    Max 3 retries?
                    │ ├─ yes → DLQ
                    │ └─ no → retry
```

```python
async def send_with_at_least_once(notification_id: str):
    """At-least-once delivery"""

    # 1. Загружаем notification
    notif = await db.notifications.find_one({"id": notification_id})
    if not notif:
        raise NotFoundError()

    # 2. Получаем девайс токены
    tokens = await get_user_device_tokens(notif["user_id"])

    try:
        # 3. Отправляем
        result = await fcm_provider.send(tokens, notif)

        # 4. Обновляем статус (atomic operation)
        await db.notifications.update_one(
            {"id": notification_id, "status": "pending"},
            {
                "$set": {
                    "status": "sent",
                    "sent_at": now(),
                    "successful_tokens": result.success,
                    "failed_tokens": result.failed
                }
            }
        )

        # 5. Отмечаем invalid tokens
        for invalid_token in result.invalid_tokens:
            await db.device_tokens.delete_one({"device_token": invalid_token})

    except Exception as e:
        # 6. На ошибку — retry
        retry_count = notif.get("retry_count", 0)

        if retry_count < MAX_RETRIES:
            delay = 2 ** retry_count  # exponential backoff
            await db.notifications.update_one(
                {"id": notification_id},
                {
                    "$set": {"retry_count": retry_count + 1},
                    "$push": {"retry_log": {"error": str(e), "at": now()}}
                }
            )

            # Re-publish to queue with delay
            await queue.publish("push_queue", {
                "notification_id": notification_id,
                "retry_count": retry_count + 1
            }, delay=delay)
        else:
            # 7. Максимум попыток — DLQ
            await db.notifications.update_one(
                {"id": notification_id},
                {"$set": {"status": "failed"}}
            )
            await queue.publish("dead_letter_queue", {
                "notification_id": notification_id,
                "error": str(e)
            })
```

**Exactly-once delivery:**

Гарантия: каждое сообщение доставляется ровно один раз, без дубликатов.

Реализация требует idempotency key:
```
Идея: провайдер видит один и тот же ключ и дедупликирует

Должны быть уникальны:
- Per user (иначе 2 пользователя не получат)
- Per notification (иначе retry создаст дубликат)
- Per provider (APNS и FCM — разные ключи)
```

```python
async def send_with_exactly_once(notification_id: str, user_id: str):
    """Exactly-once с idempotency key"""

    notif = await db.notifications.find_one({"id": notification_id})

    # Создаем уникальный ключ для дедупликации
    idempotency_key = f"{notification_id}#{user_id}#apns"

    # Проверяем, не отправляли ли уже
    sent_record = await db.sent_log.find_one({
        "idempotency_key": idempotency_key,
        "status": "success"
    })

    if sent_record:
        # Уже отправили успешно
        await db.notifications.update_one(
            {"id": notification_id},
            {"$set": {"status": "sent", "sent_at": sent_record["sent_at"]}}
        )
        return

    # Отправляем с idempotency key
    try:
        result = await apns_provider.send(
            tokens=get_user_tokens(user_id),
            notification=notif,
            idempotency_key=idempotency_key  # APNS это поддерживает
        )

        # Записываем в log
        await db.sent_log.insert_one({
            "idempotency_key": idempotency_key,
            "notification_id": notification_id,
            "user_id": user_id,
            "status": "success",
            "sent_at": now(),
            "provider": "apns"
        })

        await db.notifications.update_one(
            {"id": notification_id},
            {"$set": {"status": "sent", "sent_at": now()}}
        )

    except Exception as e:
        # Проверяем, может быть успех был до краша
        sent_record = await db.sent_log.find_one({
            "idempotency_key": idempotency_key
        })

        if sent_record and sent_record.get("status") == "success":
            # Было успешно — игнорируем ошибку
            pass
        else:
            # Реально ошибка — retry или fail
            raise

# В адаптере провайдера (APNS)
class APNSProvider:
    async def send(self, tokens, notification, idempotency_key=None):
        """APNS поддерживает apns-id header для дедупликации"""
        headers = {}
        if idempotency_key:
            headers["apns-id"] = idempotency_key

        # HTTP/2 запрос с идемпотентностью
        response = await self.http2_client.post(
            f"{APNS_URL}/3/device/{token}",
            json=payload,
            headers=headers
        )

        return response
```

**Типичные ошибки.**
1. **Обновлять статус после отправки, но до сохранения в БД** — потеря информации при краше
2. **Использовать одинаковый idempotency key для всех юзеров** — дубликаты между юзерами
3. **Не проверять duplicates перед retry** — exponential explosion дубликатов
4. **Полагаться только на провайдера** — они не дедупликируют forever
5. **Хранить sent_log в памяти** — потеря данных при перезагрузке

**На интервью.**
Нарисуй диаграмму at-least-once с retry логикой. Объясни, когда нужна exactly-once. Как ты будешь дедупликировать при distributed retries? Что если провайдер лжет о успехе?

*Частые follow-up вопросы:*
- Как обнаружить дубликаты постфактум?
- Что делать с orphaned notificattions (sent_log есть, но DB потеряна)?
- Как масштабировать sent_log при 1B notifi/день?

---

### 4. Как реализовать приоритизацию и очередь с разными приоритетами?

**Зачем спрашивают.**
Не все уведомления одинаково важны: urgent (оплата, безопасность) vs marketing. Интервьюер проверяет умение работать с priority queues и fairness scheduling.

**Короткий ответ.**
Разделяем очередь по приоритетам: high, normal, low с разным количеством воркеров. Urgent messages обрабатываются первыми. Можно использовать взвешенное распределение (60% high, 30% normal, 10% low).

**Детальный разбор.**

Архитектура с приоритизацией:
```
Notification API
    │
    ├─ priority = urgent → HIGH_QUEUE
    ├─ priority = normal → NORMAL_QUEUE
    └─ priority = low    → LOW_QUEUE

┌────────────────┬────────────────┬────────────────┐
│  HIGH_QUEUE    │ NORMAL_QUEUE   │  LOW_QUEUE     │
│ 100+ workers   │  50 workers    │  10 workers    │
└────────┬───────┴────────┬───────┴────────┬───────┘
         │                │                │
         ▼                ▼                ▼
    ┌─────────────────────────────┐
    │   Weighted Round-Robin      │
    │   (60%, 30%, 10%)           │
    │   Обрабатываем все до конца │
    └──────────┬──────────────────┘
               │
               ▼
          Provider
```

Альтернатива — single queue с priority heap:
```
Insert: O(log N)
Pop min: O(1)
```

```python
import heapq
from dataclasses import dataclass
from enum import IntEnum

class Priority(IntEnum):
    URGENT = 0      # Highest priority
    HIGH = 1
    NORMAL = 2
    LOW = 3         # Lowest priority

    @staticmethod
    def from_string(s: str):
        mapping = {
            "urgent": Priority.URGENT,
            "high": Priority.HIGH,
            "normal": Priority.NORMAL,
            "low": Priority.LOW
        }
        return mapping.get(s.lower(), Priority.NORMAL)

@dataclass
class PriorityNotification:
    priority: Priority
    timestamp: int  # для FIFO внутри одного приоритета
    notification_id: str
    user_ids: List[str]
    channel: str

    def __lt__(self, other):
        """Для heapq: сначала по приоритету, потом по времени"""
        if self.priority != other.priority:
            return self.priority < other.priority
        return self.timestamp < other.timestamp

class PriorityQueue:
    """Priority queue на основе heap"""

    def __init__(self):
        self.heap = []
        self.counter = 0  # Для уникальности при одинаковом приоритете/времени

    def put(self, notification: dict):
        """Добавить сообщение"""
        priority = Priority.from_string(notification.get("priority", "normal"))

        item = (
            priority.value,
            self.counter,
            self.counter,
            notification
        )

        heapq.heappush(self.heap, item)
        self.counter += 1

    def get(self) -> dict:
        """Получить сообщение с максимальным приоритетом"""
        while self.heap:
            priority, seq, _, notification = heapq.heappop(self.heap)
            return notification
        return None

    def size(self) -> int:
        return len(self.heap)

# Использование с Kafka/RabbitMQ
class PriorityNotificationConsumer:
    """Consumer с приоритизацией"""

    def __init__(self, num_workers: int = 10):
        self.priority_queue = PriorityQueue()
        self.num_workers = num_workers

    async def start(self):
        """Запустить воркеры"""
        tasks = []

        # Задача 1: Потребление из Kafka
        tasks.append(asyncio.create_task(self._consume_from_kafka()))

        # Задача 2: Обработка priority queue
        for i in range(self.num_workers):
            tasks.append(asyncio.create_task(self._process_queue(i)))

        await asyncio.gather(*tasks)

    async def _consume_from_kafka(self):
        """Потребляем из разных топиков Kafka"""
        async with aiokafka.AIOKafkaConsumer(
            "push_queue_urgent",
            "push_queue_high",
            "push_queue_normal",
            "push_queue_low",
            bootstrap_servers='localhost:9092'
        ) as consumer:
            async for message in consumer:
                notification = json.loads(message.value)
                self.priority_queue.put(notification)

    async def _process_queue(self, worker_id: int):
        """Обработка сообщений из priority queue"""
        while True:
            notification = self.priority_queue.get()

            if not notification:
                await asyncio.sleep(0.1)  # Очередь пуста, ждем
                continue

            try:
                await self._send_notification(notification)
            except Exception as e:
                logger.error(f"Worker {worker_id} failed: {e}")
                # Retry logic...

    async def _send_notification(self, notification: dict):
        """Отправить уведомление"""
        user_ids = notification["user_ids"]
        tokens = await get_device_tokens(user_ids)
        await fcm_provider.send(tokens, notification)

# Альтернатива: несколько очередей с взвешенным распределением
class WeightedPriorityConsumer:
    """Несколько физических очередей с взвешенным потреблением"""

    async def start(self):
        """
        3 очереди Kafka:
        - push_queue_urgent (100 воркеров, вес 0.6)
        - push_queue_normal (50 воркеров, вес 0.3)
        - push_queue_low (10 воркеров, вес 0.1)
        """

        queues = {
            "urgent": {
                "topic": "push_queue_urgent",
                "workers": 100,
                "weight": 0.6
            },
            "normal": {
                "topic": "push_queue_normal",
                "workers": 50,
                "weight": 0.3
            },
            "low": {
                "topic": "push_queue_low",
                "workers": 10,
                "weight": 0.1
            }
        }

        tasks = []

        for priority, config in queues.items():
            for i in range(config["workers"]):
                task = asyncio.create_task(
                    self._consume_and_process(
                        topic=config["topic"],
                        worker_id=f"{priority}_{i}"
                    )
                )
                tasks.append(task)

        await asyncio.gather(*tasks)

    async def _consume_and_process(self, topic: str, worker_id: str):
        """Потреблять и обрабатывать из одного топика"""
        async with aiokafka.AIOKafkaConsumer(
            topic,
            bootstrap_servers='localhost:9092',
            group_id="push_workers"
        ) as consumer:
            async for message in consumer:
                notification = json.loads(message.value)
                await self._send_notification(notification)
```

**SLA (Service Level Agreement) по приоритетам:**
```
┌───────────┬──────────────┬──────────────┐
│ Priority  │ SLA          │ Backoff      │
├───────────┼──────────────┼──────────────┤
│ URGENT    │ < 10 seconds │ < 30 seconds │
│ HIGH      │ < 1 minute   │ < 5 minutes  │
│ NORMAL    │ < 5 minutes  │ < 30 minutes │
│ LOW       │ < 1 hour     │ < 1 hour     │
└───────────┴──────────────┴──────────────┘
```

**Мониторинг:**
```python
@app.get("/metrics/queue")
async def queue_metrics():
    return {
        "queue_sizes": {
            "urgent": priority_queue.get_size("urgent"),
            "normal": priority_queue.get_size("normal"),
            "low": priority_queue.get_size("low")
        },
        "age_of_oldest_message": {
            "urgent": priority_queue.get_oldest_age("urgent"),
            "normal": priority_queue.get_oldest_age("normal")
        },
        "worker_utilization": {
            "urgent": worker_pool.get_utilization("urgent"),
            "normal": worker_pool.get_utilization("normal")
        }
    }
```

**Типичные ошибки.**
1. **Одна очередь для всех приоритетов** — low priority сообщения никогда не обрабатываются
2. **Одинаковое количество воркеров** — waste ресурсов на медленный low priority
3. **Статический приоритет** — нельзя повысить priority заблокированного сообщения
4. **Отсутствие SLA мониторинга** — не видим violations пока клиенты не жалуются
5. **Age-based стarvation** — старые low priority forever ждут

**На интервью.**
Как ты определишь приоритет? Как масштабировать воркеры? Как обрабатывать случай, когда urgent очередь постоянно полна? Как мониторить SLA нарушения?

*Частые follow-up вопросы:*
- Как динамически перебалансировать воркеры?
- Как повысить приоритет старого сообщения?
- Как избежать starvation low priority?
- Как обрабатывать срочный случай (все на urgent)?

---

### 5. Как реализовать rate limiting и throttling для уведомлений?

**Зачем спрашивают.**
Rate limiting защищает от спама и перегрузки. Интервьюер проверяет различие между server-side и user-side ограничениями, а также умение работать с различными алгоритмами (token bucket, leaky bucket, sliding window).

**Короткий ответ.**
Rate limiting реализуется на разных уровнях: per-user (Redis), per-sender (DB), per-provider (exponential backoff). Token bucket позволяет задать rate в сообщениях/часу и burst size.

**Детальный разбор.**

Уровни rate limiting:
```
┌──────────────────────────────────────────────┐
│        Notification API (Input)              │
│  Rate limit: max 100 requests/min per user   │
└──────────┬──────────────────────────────────┘
           │
┌──────────▼──────────────────────────────────┐
│      Notification Service (Processing)      │
│  Rate limit: max 1000 messages/hour per user│
└──────────┬──────────────────────────────────┘
           │
┌──────────▼──────────────────────────────────┐
│      Provider Integration (Output)          │
│  APNS: 10000/sec, Twilio: 100/sec per acc   │
└──────────────────────────────────────────────┘
```

**User-level rate limiting:**
```python
from redis import Redis
from typing import Tuple

class UserRateLimiter:
    """Rate limit per user (token bucket)"""

    def __init__(self, redis_client: Redis):
        self.redis = redis_client

    async def can_send(
        self,
        user_id: str,
        channel: str,
        rate_per_hour: int = 100,
        burst_size: int = 10
    ) -> Tuple[bool, dict]:
        """
        Token bucket algorithm:
        - rate_per_hour: сколько разрешено в час
        - burst_size: сколько максимально можно сделать сразу
        """

        key = f"rate:{user_id}:{channel}"

        # Получаем текущий bucket
        bucket = self.redis.hgetall(key)

        now = time.time()
        last_refill = float(bucket.get(b"last_refill", now))
        tokens = float(bucket.get(b"tokens", burst_size))

        # Сколько времени прошло с последнего пополнения
        elapsed = now - last_refill

        # Пополняем токены: (rate_per_hour / 3600) = tokens per second
        tokens_per_second = rate_per_hour / 3600
        tokens += elapsed * tokens_per_second

        # Не превышаем burst
        tokens = min(tokens, burst_size)

        # Можем ли отправить?
        if tokens >= 1:
            tokens -= 1
            allowed = True
        else:
            allowed = False

        # Сохраняем состояние
        self.redis.hset(key, mapping={
            b"tokens": tokens,
            b"last_refill": now
        })
        self.redis.expire(key, 3600)  # Очищаем старые ключи

        remaining = int(tokens)
        reset_in = (1 - tokens) / tokens_per_second if tokens < 1 else 0

        return allowed, {
            "remaining": remaining,
            "burst_size": burst_size,
            "reset_in_seconds": int(reset_in)
        }

# Использование в API
@app.post("/api/v1/notifications")
async def send_notification(request: NotificationRequest, user_id: str):
    channel = request.channels[0]

    allowed, info = await user_rate_limiter.can_send(
        user_id=user_id,
        channel=channel,
        rate_per_hour=100,  # 100 сообщений в час для обычного юзера
        burst_size=5        # Но максимум 5 одновременно
    )

    if not allowed:
        return {
            "error": "rate_limit_exceeded",
            "retry_after": info["reset_in_seconds"]
        }, 429

    # ... отправляем уведомление
```

**Quiet hours и per-user preferences:**
```python
class UserPreferences:
    """Управление предпочтениями пользователя"""

    async def should_send(
        self,
        user_id: str,
        channel: str,
        priority: str = "normal"
    ) -> Tuple[bool, str]:
        """
        Проверить, может ли быть отправлено уведомление
        Учитываем: enabled/disabled, quiet hours, spam limit
        """

        prefs = await db.user_preferences.find_one({"user_id": user_id})

        if not prefs:
            return True, "no_preference_set"

        # 1. Проверяем, включен ли канал
        if not prefs.get(f"{channel}_enabled", True):
            return False, f"{channel}_disabled"

        # 2. Проверяем quiet hours
        if self._is_quiet_hours(user_id, prefs):
            # Для urgent можем пренебречь
            if priority != "urgent":
                return False, "in_quiet_hours"

        # 3. Проверяем spam limit (только для marketing)
        if priority == "low":
            can_send, reason = await self._check_daily_limit(
                user_id, channel
            )
            if not can_send:
                return False, reason

        return True, "ok"

    def _is_quiet_hours(self, user_id: str, prefs: dict) -> bool:
        """Проверить, в quiet hours ли сейчас"""

        quiet_hours = prefs.get("quiet_hours")
        if not quiet_hours:
            return False

        # Учитываем timezone пользователя
        tz = pytz.timezone(prefs.get("timezone", "UTC"))
        now = datetime.now(tz)
        current_time = now.time()

        start = datetime.strptime(
            quiet_hours.get("start", "22:00"), "%H:%M"
        ).time()
        end = datetime.strptime(
            quiet_hours.get("end", "08:00"), "%H:%M"
        ).time()

        # Обработка переноса через полночь
        if start <= end:
            return start <= current_time <= end
        else:
            return current_time >= start or current_time <= end

    async def _check_daily_limit(
        self,
        user_id: str,
        channel: str
    ) -> Tuple[bool, str]:
        """Проверить daily limit для маркетингового контента"""

        today = datetime.now().date()
        key = f"daily:{user_id}:{channel}:{today}"

        count = self.redis.incr(key)

        if count == 1:
            self.redis.expire(key, 86400)  # 1 день

        daily_limit = 10  # 10 маркетинг уведомлений в день

        if count > daily_limit:
            return False, f"daily_limit_exceeded ({count}/{daily_limit})"

        return True, "ok"

# Middleware в API
@app.middleware("http")
async def rate_limit_middleware(request: Request, call_next):
    # Применяем rate limiting
    user_id = request.headers.get("X-User-ID")

    allowed, info = await user_rate_limiter.can_send(
        user_id=user_id,
        channel="api",
        rate_per_hour=100
    )

    if not allowed:
        return JSONResponse(
            {
                "error": "rate_limit_exceeded",
                "retry_after": info["reset_in_seconds"]
            },
            status_code=429,
            headers={
                "X-RateLimit-Limit": "100",
                "X-RateLimit-Remaining": str(info["remaining"]),
                "X-RateLimit-Reset": str(int(time.time()) + info["reset_in_seconds"])
            }
        )

    return await call_next(request)
```

**Provider-level rate limiting:**
```python
class ProviderRateLimiter:
    """Обработка rate limits внешних провайдеров"""

    # Rate limits провайдеров
    LIMITS = {
        "apns": {"per_second": 10000},
        "fcm": {"per_second": 10000},
        "twilio": {"per_second": 100}
    }

    async def handle_provider_limit(
        self,
        provider: str,
        error: Exception
    ) -> dict:
        """
        Обработать rate limit от провайдера
        Возвращает: delay и action (retry, queue, fail)
        """

        if "rate" in str(error).lower():
            # Provider сказал нам слишком быстро
            current_rate = self._get_current_rate(provider)
            limit = self.LIMITS[provider]["per_second"]

            # Exponential backoff
            delay = max(1, (current_rate / limit) ** 2)

            return {
                "action": "backoff",
                "delay_seconds": delay,
                "reduce_concurrency": True
            }

        return {
            "action": "retry",
            "delay_seconds": 1
        }

    def _get_current_rate(self, provider: str) -> float:
        """Получить текущую rate отправки к провайдеру"""
        key = f"rate:{provider}:current"
        return float(self.redis.get(key) or 0)
```

**Типичные ошибки.**
1. **Только server-side rate limiting** — все равно может быть спам от baggy client'а
2. **Одинаковый limit для всех** — VIP юзеры должны иметь выше лимит
3. **Отсутствие quiet hours** — уведомления посреди ночи раздражают
4. **Игнорировать user preferences** — пользователи выключают уведомления
5. **Не обрабатывать 429 от провайдера** — продолжаем спамить → black list

**На интервью.**
Как ты отличаешь rate limit от user preference? Как масштабировать Redis при 100K+ юзеров? Как обрабатывать VIP юзеров с выше лимитом? Как обнаружить и зафиксить baggy client?

*Частые follow-up вопросы:*
- Как реализовать distributed rate limiting через несколько инстансов?
- Как обрабатывать burst (например, breaking news)?
- Как динамически отключать каналы при перегрузке?

---

### 6. Как реализовать fanout для миллионов пользователей?

**Зачем спрашивают.**
Fanout — отправка одного сообщения миллионам пользователей (breaking news, events) — это сложная задача масштабирования. Интервьюер проверяет понимание различных стратегий и их trade-offs.

**Короткий ответ.**
Три стратегии: 1) Direct sending (медленно), 2) Pre-queue per user (много памяти), 3) Topic-based messaging (efficient). Для миллионов используем topic-based (Kafka, FCM topics).

**Детальный разбор.**

Сравнение стратегий:
```
┌──────────────────────┬─────────────┬──────────┬────────────┐
│ Стратегия            │ CPU         │ Memory   │ Scalability│
├──────────────────────┼─────────────┼──────────┼────────────┤
│ Direct send          │ High        │ Low      │ Poor       │
│ Per-user queue       │ Medium      │ High     │ Poor       │
│ Topic-based          │ Low         │ Low      │ Excellent  │
│ Timeline fanout      │ Medium      │ Very High│ Good       │
└──────────────────────┴─────────────┴──────────┴────────────┘
```

**Стратегия 1: Direct Sending (медленно, но простая)**
```
┌─────────────────┐
│  Notification   │
│   (broadcast)   │
└────────┬────────┘
         │
         ▼
    Loop: for each user
         │
    ┌────▼────┐
    │  Queue  │
    │per user │
    └────┬────┘
         │
         ▼
    Send to provider
```

```python
async def fanout_direct(notification_id: str, target_user_ids: List[str]):
    """Direct send — медленно для большого количества"""

    for user_id in target_user_ids:
        message = {
            "notification_id": notification_id,
            "user_id": user_id,
            "channel": "push"
        }
        await queue.publish("push_queue", message)
```

Проблема: для 100M пользователей это 100M publish операций. Медленно.

**Стратегия 2: Per-User Queue (быстро до лимита памяти)**
```
Single message → create M queues (один на пользователя)

┌──────────────────────┐
│   Broadcast Message  │
└──────────────┬───────┘
               │
    ┌──────────┴──────────┐
    │                     │
    ▼                     ▼
 User1_queue          User_N_queue
   (message)            (message)
```

```python
async def fanout_per_user_queue(
    notification_id: str,
    target_user_ids: List[str],
    batch_size: int = 1000
):
    """Per-user queue — быстро но много операций"""

    # Батчим для efficiency
    for batch in chunks(target_user_ids, batch_size):
        # Multi-publish в Redis/RabbitMQ
        messages = [
            {
                "notification_id": notification_id,
                "user_id": uid,
                "type": "broadcast"
            }
            for uid in batch
        ]

        await queue.publish_multi("user_queue", messages)
```

Проблема: для каждого пользователя отдельный key в Redis = N записей. При 100M юзеров = 100M keys в памяти Redis.

**Стратегия 3: Topic-Based (эффективно)**
```
┌──────────────────────┐
│  Broadcast Message   │
└──────────┬───────────┘
           │
    ┌──────▼──────┐
    │ Topic: news │ ← Single copy
    └──────┬──────┘
           │
    ┌──────┴──────┬──────────┬──────────┐
    │             │          │          │
    ▼             ▼          ▼          ▼
 Worker1      Worker2    Worker3   Worker4
 consumes    consumes   consumes   consumes
  from topic from topic from topic from topic

Each worker sends to its own users
```

Как работает:
1. Одно сообщение в topic (не M сообщений)
2. Много воркеров подписаны на topic
3. Каждый воркер обрабатывает подмножество пользователей

```python
async def fanout_topic_based(
    notification_id: str,
    topic: str,
    total_users: int
):
    """Topic-based — эффективно для масштабирования"""

    # 1. Публикуем один раз в topic
    message = {
        "notification_id": notification_id,
        "topic": topic,
        "total_users": total_users,
        "created_at": now()
    }

    await kafka.publish("broadcast_topic", message, key=topic)

    # 2. Воркеры читают из topic и обрабатывают
    # (see below)

# Worker side
async def broadcast_worker(worker_id: int, num_workers: int):
    """
    Несколько воркеров обрабатывают одно broadcast сообщение
    Каждый берет свой диапазон пользователей
    """

    async with aiokafka.AIOKafkaConsumer(
        "broadcast_topic",
        bootstrap_servers='localhost:9092',
        group_id="broadcast_workers"
    ) as consumer:
        async for message in consumer:
            broadcast = json.loads(message.value)

            # Определяем, какую часть обрабатывает этот воркер
            user_batch = await get_user_batch(
                broadcast["topic"],
                total_users=broadcast["total_users"],
                worker_id=worker_id,
                num_workers=num_workers
            )

            # Отправляем для своего подмножества
            for user_id in user_batch:
                await send_to_user(broadcast["notification_id"], user_id)

async def get_user_batch(
    topic: str,
    total_users: int,
    worker_id: int,
    num_workers: int
) -> List[str]:
    """Получить пользователей для этого воркера (range-based)"""

    batch_size = total_users // num_workers
    start = worker_id * batch_size
    end = start + batch_size if worker_id < num_workers - 1 else total_users

    # Получаем user_ids из БД или cache
    users = await db.users.find(
        {"id": {"$gte": start, "$lt": end}}
    ).to_list(None)

    return [u["id"] for u in users]
```

**FCM Topic Messaging (native broadcast):**
```python
# FCM имеет встроенную поддержку topic-based messaging
async def fanout_fcm_topics(notification_id: str, topic: str):
    """
    FCM handles fanout natively
    - Subscribe users to topic once
    - Send to topic, FCM handles distribution
    """

    message = messaging.Message(
        notification=messaging.Notification(
            title="Breaking News",
            body="Check out this update"
        ),
        webpush=messaging.WebpushConfig(
            data={"notification_id": notification_id}
        ),
        topic=topic  # FCM handles the fanout!
    )

    # Отправляем один раз, FCM доставляет всем подписанным
    response = await asyncio.to_thread(
        firebase_app.send,
        message
    )

    logger.info(f"Sent to topic {topic}: {response}")

# When user logs in, subscribe to topics
async def subscribe_user_to_topics(user_id: str, device_token: str):
    """Подписываем пользователя на интересующие темы"""

    user = await db.users.find_one({"id": user_id})

    topics = []
    if user.get("subscribed_to_news"):
        topics.append("news")
    if user.get("subscribed_to_sports"):
        topics.append("sports")
    if user.get("location"):
        topics.append(f"location_{user['location']}")

    for topic in topics:
        await asyncio.to_thread(
            firebase_app.make_topic_management_app().subscribe_to_topic,
            device_token,
            topic
        )
```

**Расчет для 100M пользователей:**

```
Direct send:
- 100M publish операций
- Time: 100M / (1000 ops/sec) = ~28 hours ❌

Per-user queue:
- 100M keys в Redis
- Memory: 100M keys × 1KB = 100GB ❌

Topic-based (10 воркеров):
- 1 publish операция
- 10M users per worker
- Time: ~10 seconds ✓
- Memory: ~constant ✓
```

**Мониторинг fanout:**
```python
@app.post("/api/v1/broadcast")
async def broadcast_notification(request: BroadcastRequest):
    """Отправить broadcast уведомление"""

    notification_id = str(uuid4())
    target_user_ids = await db.users.find(request.filter).to_list(None)

    logger.info(
        f"Broadcasting {notification_id} to {len(target_user_ids)} users"
    )

    start_time = time.time()

    if len(target_user_ids) > 10_000_000:
        # Для > 10M используем topic-based
        topic = f"broadcast_{notification_id}"
        await fanout_topic_based(notification_id, topic, len(target_user_ids))
    else:
        # Для меньше используем per-user queue
        await fanout_per_user_queue(notification_id, target_user_ids)

    elapsed = time.time() - start_time

    metrics.record_fanout_time(notification_id, elapsed, len(target_user_ids))

    return {
        "notification_id": notification_id,
        "target_users": len(target_user_ids),
        "fanout_time_seconds": elapsed
    }
```

**Типичные ошибки.**
1. **Direct send для больших broadcast** — system overload
2. **Per-user queue для всех** — OOM при 100M users
3. **Отсутствие worker coordination** — duplicates или gaps
4. **Не используя native topic messaging (FCM)** — reinventing the wheel
5. **Отсутствие backpressure** — when workers slow down

**На интервью.**
Как ты отправишь 100M уведомлений за 10 минут? Какую стратегию выберешь и почему? Как обрабатывать, если в процессе broadcast система упадет? Как мониторить прогресс?

*Частые follow-up вопросы:*
- Как обеспечить exactly-once delivery при topic fanout?
- Как переисправить сообщение, если заметили ошибку?
- Как обрабатывать динамические user группы (real-time filtering)?

---

### 7. Как обработать сбои и реализовать retry с exponential backoff?

**Зачем спрашивают.**
Надежность системы зависит от обработки сбоев. Интервьюер проверяет понимание различных типов ошибок и стратегий их восстановления.

**Короткий ответ.**
Различаем retryable ошибки (timeout, 5xx) и non-retryable (4xx). Используем exponential backoff: delay = base * 2^retry_count + jitter. Max retries: 3-5 попыток. Остаток в DLQ.

**Детальный разбор.**

Классификация ошибок:
```
┌──────────────┬─────────────────┬──────────┐
│ Ошибка       │ Тип             │ Action   │
├──────────────┼─────────────────┼──────────┤
│ 400, 401, 403│ Client error    │ Fail     │
│ 404, 410     │ Invalid resource│ Fail+Log │
│ 429          │ Rate limit      │ Backoff  │
│ 500, 502, 503│ Server error    │ Retry    │
│ Timeout      │ Network issue   │ Retry    │
│ Connection   │ Network issue   │ Retry    │
└──────────────┴─────────────────┴──────────┘
```

```python
from enum import Enum
from typing import Optional
import random
import asyncio

class ErrorType(Enum):
    RETRYABLE = "retryable"      # timeout, 5xx, temporary
    NON_RETRYABLE = "non_retryable"  # 4xx (except 429), invalid
    RATE_LIMIT = "rate_limit"    # 429

class RetryPolicy:
    """Политика повторных попыток"""

    def __init__(
        self,
        max_retries: int = 3,
        base_delay_ms: int = 1000,
        max_delay_ms: int = 30000,
        jitter: bool = True
    ):
        self.max_retries = max_retries
        self.base_delay_ms = base_delay_ms
        self.max_delay_ms = max_delay_ms
        self.jitter = jitter

    def get_delay(self, retry_count: int) -> int:
        """Вычислить delay для retry (exponential backoff + jitter)"""

        if retry_count >= self.max_retries:
            return None  # Max retries reached

        # Exponential: 1s, 2s, 4s, 8s, ...
        delay_ms = self.base_delay_ms * (2 ** retry_count)
        delay_ms = min(delay_ms, self.max_delay_ms)

        # Jitter: добавляем случайность чтобы избежать thundering herd
        if self.jitter:
            jitter_ms = random.randint(0, delay_ms // 2)
            delay_ms += jitter_ms

        return delay_ms

    def classify_error(self, error: Exception) -> ErrorType:
        """Классифицировать ошибку"""

        error_str = str(error).lower()

        # Rate limit
        if "rate" in error_str or "429" in error_str:
            return ErrorType.RATE_LIMIT

        # Retryable
        if any(x in error_str for x in [
            "timeout", "connection", "500", "502", "503", "504",
            "toobusy", "unavailable"
        ]):
            return ErrorType.RETRYABLE

        # Non-retryable
        if any(x in error_str for x in [
            "400", "401", "403", "404", "410", "badrequest",
            "unauthorized", "forbidden", "invalid"
        ]):
            return ErrorType.NON_RETRYABLE

        # Default: retry если не уверены
        return ErrorType.RETRYABLE

# Worker с retry логикой
class NotificationWorker:
    def __init__(self, provider_name: str):
        self.provider_name = provider_name
        self.retry_policy = RetryPolicy()

    async def process_notification(
        self,
        notification: dict,
        max_retries: int = 3
    ):
        """Обработать уведомление с retries"""

        retry_count = notification.get("retry_count", 0)
        notification_id = notification["id"]

        try:
            # Отправляем уведомление
            result = await self._send_notification(notification)

            # Успех
            await db.notifications.update_one(
                {"id": notification_id},
                {
                    "$set": {
                        "status": "sent",
                        "sent_at": now(),
                        "sent_by": self.provider_name
                    }
                }
            )

            logger.info(f"Notification {notification_id} sent successfully")

        except Exception as e:
            # Классифицируем ошибку
            error_type = self.retry_policy.classify_error(e)

            if error_type == ErrorType.NON_RETRYABLE:
                # Non-retryable — отправляем в fail
                await self._handle_non_retryable_error(
                    notification_id, e
                )

            elif error_type == ErrorType.RATE_LIMIT:
                # Rate limit — более агрессивный backoff
                delay_ms = min(
                    self.retry_policy.base_delay_ms * (3 ** retry_count),
                    60000  # max 1 minute
                )
                await self._schedule_retry(
                    notification, retry_count, delay_ms
                )

            else:  # RETRYABLE
                # Retryable — normal exponential backoff
                if retry_count < max_retries:
                    delay_ms = self.retry_policy.get_delay(retry_count)
                    await self._schedule_retry(
                        notification, retry_count, delay_ms
                    )
                else:
                    # Max retries — в DLQ
                    await self._handle_max_retries_exceeded(
                        notification_id, e
                    )

    async def _send_notification(self, notification: dict) -> dict:
        """Отправить через провайдер (может бросить исключение)"""

        tokens = await get_user_device_tokens(
            notification["user_id"]
        )

        if not tokens:
            raise ValueError("No device tokens for user")

        # Отправляем с timeout
        try:
            result = await asyncio.wait_for(
                self.provider.send(tokens, notification),
                timeout=10  # 10 second timeout
            )
            return result

        except asyncio.TimeoutError:
            raise TimeoutError(f"Provider {self.provider_name} timeout")

    async def _schedule_retry(
        self,
        notification: dict,
        current_retry: int,
        delay_ms: int
    ):
        """Запланировать retry"""

        notification_id = notification["id"]
        new_retry_count = current_retry + 1

        # Обновляем в БД
        await db.notifications.update_one(
            {"id": notification_id},
            {
                "$set": {
                    "retry_count": new_retry_count,
                    "next_retry_at": now() + timedelta(milliseconds=delay_ms)
                },
                "$push": {
                    "retry_log": {
                        "attempt": new_retry_count,
                        "at": now(),
                        "scheduled_next": now() + timedelta(milliseconds=delay_ms)
                    }
                }
            }
        )

        # Переотправляем в queue с delay
        notification["retry_count"] = new_retry_count
        await queue.publish(
            f"{notification['channel']}_queue",
            notification,
            delay_ms=delay_ms
        )

        logger.warning(
            f"Scheduled retry {new_retry_count} for {notification_id} "
            f"in {delay_ms}ms"
        )

    async def _handle_non_retryable_error(
        self,
        notification_id: str,
        error: Exception
    ):
        """Обработать non-retryable ошибку"""

        await db.notifications.update_one(
            {"id": notification_id},
            {
                "$set": {
                    "status": "failed",
                    "failed_at": now(),
                    "failure_reason": str(error),
                    "failure_type": "non_retryable"
                }
            }
        )

        logger.error(f"Non-retryable error for {notification_id}: {error}")

        # Можно отправить alert для важных уведомлений
        if should_alert_on_failure(notification_id):
            await alerting_service.send_alert(
                f"Notification {notification_id} failed with non-retryable error"
            )

    async def _handle_max_retries_exceeded(
        self,
        notification_id: str,
        error: Exception
    ):
        """Обработать превышение максимума retry"""

        await db.notifications.update_one(
            {"id": notification_id},
            {
                "$set": {
                    "status": "failed",
                    "failed_at": now(),
                    "failure_reason": str(error),
                    "failure_type": "max_retries_exceeded"
                }
            }
        )

        # Отправляем в DLQ для дальнейшего анализа
        await queue.publish(
            "dead_letter_queue",
            {
                "notification_id": notification_id,
                "error": str(error),
                "reason": "max_retries_exceeded"
            }
        )

        logger.error(
            f"Max retries exceeded for {notification_id}: {error}"
        )

# Dead Letter Queue обработка
async def dlq_processor():
    """Обработка мертвых писем"""

    while True:
        message = await queue.consume("dead_letter_queue")

        notification_id = message["notification_id"]
        error = message["error"]
        reason = message["reason"]

        # Логируем для анализа
        await db.dead_letter_queue.insert_one({
            "notification_id": notification_id,
            "error": error,
            "reason": reason,
            "processed_at": now()
        })

        # Alert для важных случаев
        notif = await db.notifications.find_one({"id": notification_id})
        if notif.get("priority") in ["urgent", "high"]:
            await alerting_service.send_alert(
                f"Important notification {notification_id} in DLQ: {error}"
            )

        await queue.acknowledge(message)
```

**Мониторинг retry:**
```python
@app.get("/metrics/retries")
async def retry_metrics():
    """Метрики по retry"""

    return {
        "pending_retries": await db.notifications.count_documents({
            "status": "pending",
            "next_retry_at": {"$lt": now()}
        }),
        "retry_distribution": await db.notifications.aggregate([
            {"$match": {"retry_count": {"$gt": 0}}},
            {
                "$group": {
                    "_id": "$retry_count",
                    "count": {"$sum": 1}
                }
            },
            {"$sort": {"_id": 1}}
        ]).to_list(None),
        "failed_count": await db.notifications.count_documents({
            "status": "failed"
        }),
        "dlq_count": await db.dead_letter_queue.estimated_document_count()
    }
```

**Типичные ошибки.**
1. **Retry всех ошибок** — 400 ошибки никогда не будут успешны
2. **Отсутствие jitter** — thundering herd (все переотправляют одновременно)
3. **Очень большой max_delay** — долго ждем перед DLQ
4. **Хранить retry_count только в памяти** — потеря при краше воркера
5. **Отсутствие DLQ мониторинга** — не видим какие сообщения упали

**На интервью.**
Как ты различаешь retryable и non-retryable ошибки? Почему нужен jitter? Как сделать retry логику, если воркер crash'а? Как мониторить DLQ и alert на критичные случаи?

*Частые follow-up вопросы:*
- Как обработать circuit breaker для постоянно падающего провайдера?
- Как перепроцессировать сообщения из DLQ?
- Как различить временный timeout от permanent failure?

---

### 8. Как управлять user preferences и отслеживать delivery?

**Зачем спрашивают.**
User preferences и delivery tracking — это скорее не technical challenges, а фичи, которые критичны для юзера. Интервьюер проверяет, насколько полно ты думаешь о UX.

**Короткий ответ.**
User preferences: channels enabled, quiet hours, timezone, frequency caps — хранятся в БД и кэшируются в Redis. Delivery tracking: уведомления об открытии/взаимодействии через webhooks от провайдеров, статус в DB: pending → sent → delivered → opened.

**Детальный разбор.**

User Preferences Schema:
```
┌──────────────────────────────────────────┐
│       User Preferences                   │
├──────────────────────────────────────────┤
│ user_id (PK)                             │
│ push_enabled: bool (default: true)       │
│ email_enabled: bool (default: true)      │
│ sms_enabled: bool (default: false)       │
│ quiet_hours: JSON                        │
│   {                                      │
│     "start": "22:00",                    │
│     "end": "08:00",                      │
│     "timezone": "America/New_York"       │
│   }                                      │
│ frequency_cap: JSON                      │
│   {                                      │
│     "promotional": "3/week",             │
│     "transactional": "unlimited",        │
│     "news": "daily"                      │
│   }                                      │
│ topic_preferences: JSON                  │
│   {                                      │
│     "sports": true,                      │
│     "politics": false,                   │
│     "tech": true                         │
│   }                                      │
│ do_not_track: bool (for analytics)       │
│ updated_at: timestamp                    │
└──────────────────────────────────────────┘
```

```python
class UserPreferencesManager:
    """Управление preferences пользователя"""

    CACHE_TTL = 3600  # 1 hour

    async def get_preferences(self, user_id: str) -> dict:
        """Получить preferences (с кэшированием)"""

        # 1. Пробуем Redis
        cache_key = f"prefs:{user_id}"
        cached = await redis.get(cache_key)

        if cached:
            return json.loads(cached)

        # 2. Иначе из DB
        prefs = await db.user_preferences.find_one({"user_id": user_id})

        if not prefs:
            # Первый раз — default preferences
            prefs = {
                "user_id": user_id,
                "push_enabled": True,
                "email_enabled": True,
                "sms_enabled": False,
                "quiet_hours": {
                    "start": "22:00",
                    "end": "08:00",
                    "timezone": "UTC"
                },
                "frequency_cap": {
                    "promotional": "3/week",
                    "transactional": "unlimited",
                    "news": "1/day"
                },
                "created_at": now()
            }
            await db.user_preferences.insert_one(prefs)

        # 3. Кэшируем
        await redis.setex(
            cache_key,
            self.CACHE_TTL,
            json.dumps(prefs, default=str)
        )

        return prefs

    async def update_preferences(
        self,
        user_id: str,
        updates: dict
    ) -> dict:
        """Обновить preferences"""

        # Validate updates
        allowed_fields = {
            "push_enabled", "email_enabled", "sms_enabled",
            "quiet_hours", "frequency_cap", "topic_preferences"
        }

        for field in updates:
            if field not in allowed_fields:
                raise ValueError(f"Invalid field: {field}")

        # Обновляем в DB
        result = await db.user_preferences.find_one_and_update(
            {"user_id": user_id},
            {
                "$set": {
                    **updates,
                    "updated_at": now()
                }
            },
            return_document=True
        )

        # Инвалидируем кэш
        await redis.delete(f"prefs:{user_id}")

        return result

# API
@app.get("/api/v1/users/{user_id}/preferences")
async def get_user_preferences(user_id: str):
    """Получить preferences пользователя"""

    prefs = await preferences_manager.get_preferences(user_id)

    return {
        "push": prefs["push_enabled"],
        "email": prefs["email_enabled"],
        "sms": prefs["sms_enabled"],
        "quiet_hours": prefs["quiet_hours"],
        "frequency_cap": prefs["frequency_cap"],
        "updated_at": prefs.get("updated_at")
    }

@app.put("/api/v1/users/{user_id}/preferences")
async def update_user_preferences(
    user_id: str,
    request: UpdatePreferencesRequest
):
    """Обновить preferences"""

    updated = await preferences_manager.update_preferences(
        user_id,
        request.dict()
    )

    return {"status": "ok", "updated_at": updated["updated_at"]}
```

**Delivery Tracking:**

Статусы уведомления:
```
┌─────────┐   ┌────────┐   ┌─────────┐
│ queued  │──▶│ sent   │──▶│delivered│
└─────────┘   └────────┘   └─────────┘
                  │
                  │ (error)
                  ▼
            ┌──────────┐
            │ failed   │
            └──────────┘

После delivery может быть:
- opened (user opened the notification)
- clicked (user clicked on notification)
- dismissed (user dismissed)
```

```python
class NotificationTracker:
    """Отслеживание delivery и interaction"""

    async def update_delivery_status(
        self,
        notification_id: str,
        status: str,
        timestamp: datetime = None
    ):
        """Обновить статус доставки"""

        if not timestamp:
            timestamp = now()

        status_field = {
            "delivered": {"delivered_at": timestamp},
            "opened": {"opened_at": timestamp},
            "clicked": {"clicked_at": timestamp, "opened_at": timestamp},
            "dismissed": {"dismissed_at": timestamp}
        }.get(status, {})

        await db.notifications.update_one(
            {"id": notification_id},
            {
                "$set": {
                    "status": status,
                    **status_field,
                    "updated_at": timestamp
                }
            }
        )

    async def handle_provider_webhook(
        self,
        provider: str,
        event: dict
    ):
        """Обработать webhook от провайдера (APNS feedback, FCM, etc)"""

        if provider == "apns":
            # APNS callback format
            notification_id = event.get("notification_id")
            status = event.get("status")  # 'received', 'failed'

            await self.update_delivery_status(notification_id, "delivered")

        elif provider == "fcm":
            # FCM delivery receipt
            notification_id = event.get("notification_id")
            data = event.get("data", {})

            if data.get("delivered"):
                await self.update_delivery_status(notification_id, "delivered")

        elif provider == "segment":
            # Email open tracking
            notification_id = event.get("notification_id")
            event_type = event.get("type")  # 'open', 'click'

            if event_type == "open":
                await self.update_delivery_status(notification_id, "opened")
            elif event_type == "click":
                await self.update_delivery_status(notification_id, "clicked")

# Webhook endpoints
@app.post("/webhooks/provider/{provider}/feedback")
async def provider_feedback_webhook(provider: str, request: Request):
    """Получить feedback от провайдера"""

    try:
        body = await request.json()

        await tracker.handle_provider_webhook(provider, body)

        return {"status": "ok"}

    except Exception as e:
        logger.error(f"Webhook error: {e}")
        return {"status": "error"}, 500

# Получение статистики доставки
@app.get("/api/v1/notifications/{notification_id}/stats")
async def get_notification_stats(notification_id: str):
    """Получить статистику доставки"""

    notif = await db.notifications.find_one(
        {"id": notification_id}
    )

    if not notif:
        return {"error": "not found"}, 404

    return {
        "notification_id": notification_id,
        "status": notif["status"],
        "created_at": notif["created_at"],
        "sent_at": notif.get("sent_at"),
        "delivered_at": notif.get("delivered_at"),
        "opened_at": notif.get("opened_at"),
        "clicked_at": notif.get("clicked_at"),
        "delivery_time_ms": (
            (notif.get("delivered_at") - notif["created_at"]).total_seconds() * 1000
            if notif.get("delivered_at") else None
        ),
        "retry_count": notif.get("retry_count", 0)
    }

# Batch analytics
@app.get("/api/v1/notifications/batch/{batch_id}/analytics")
async def get_batch_analytics(batch_id: str):
    """Получить аналитику для batch отправки"""

    notifications = await db.notifications.find(
        {"batch_id": batch_id}
    ).to_list(None)

    stats = {
        "total": len(notifications),
        "sent": sum(1 for n in notifications if n.get("sent_at")),
        "delivered": sum(1 for n in notifications if n.get("delivered_at")),
        "opened": sum(1 for n in notifications if n.get("opened_at")),
        "clicked": sum(1 for n in notifications if n.get("clicked_at")),
        "failed": sum(1 for n in notifications if n["status"] == "failed")
    }

    stats["delivery_rate"] = stats["delivered"] / stats["sent"] if stats["sent"] > 0 else 0
    stats["open_rate"] = stats["opened"] / stats["delivered"] if stats["delivered"] > 0 else 0

    return stats
```

**Типичные ошибки.**
1. **Игнорирование user preferences** — спам → users disable notifications
2. **Хранить preferences в памяти** — потеря данных при перезагрузке
3. **Отсутствие кэша** — N queries в DB на каждое уведомление
4. **Не инвалидировать кэш при обновлении** — stale data
5. **Отсутствие delivery tracking** — не знаешь success rate

**На интервью.**
Как ты будешь гарантировать, что preferences обновились везде? Как обрабатывать timezone в quiet hours? Как отслеживать, открыл ли юзер push уведомление? Как масштабировать analytics при 100M notifi/день?

*Частые follow-up вопросы:*
- Как обрабатывать privacy (opt-out, GDPR)?
- Как детально отслеживать user engagement (для A/B testing)?
- Как alert'ить на anomalies (очень низкий delivery rate)?

---

### 9. Как мониторить и алертить на проблемы?

**Зачем спрашивают.**
Система уведомлений — production service, который должен работать 24/7. Интервьюер проверяет умение метрик, логирования и оперативного обнаружения проблем.

**Короткий ответ.**
Четыре уровня мониторинга: 1) Метрики (rate, latency, errors), 2) Логи (структурированные, с ID tracking), 3) Алерты (rate limit, delivery rate < 95%), 4) Dashboards (real-time визуализация).

**Детальный разбор.**

Key Metrics:
```
┌─────────────────────┬──────────────┬──────────────┐
│ Метрика             │ Target       │ Alert        │
├─────────────────────┼──────────────┼──────────────┤
│ Delivery rate       │ > 99%        │ < 95%        │
│ Latency (p99)       │ < 5s         │ > 10s        │
│ Error rate          │ < 0.1%       │ > 1%         │
│ Queue depth         │ < 10K        │ > 100K       │
│ API latency         │ < 100ms      │ > 500ms      │
│ Provider success    │ > 99%        │ < 95%        │
└─────────────────────┴──────────────┴──────────────┘
```

```python
from prometheus_client import Counter, Histogram, Gauge
import logging
import structlog

# Метрики Prometheus
notifications_sent = Counter(
    'notifications_sent_total',
    'Total notifications sent',
    ['channel', 'priority', 'provider']
)

notifications_failed = Counter(
    'notifications_failed_total',
    'Total notifications failed',
    ['channel', 'reason']
)

delivery_latency = Histogram(
    'notification_delivery_seconds',
    'Time to deliver notification',
    ['channel'],
    buckets=[0.1, 0.5, 1, 5, 10, 30, 60]
)

queue_depth = Gauge(
    'notification_queue_depth',
    'Current queue depth',
    ['queue_name']
)

retry_count = Counter(
    'notification_retries_total',
    'Total retries',
    ['channel', 'reason']
)

# Структурированное логирование
logger = structlog.get_logger()

async def process_notification_with_metrics(notification: dict):
    """Обработка с метриками и логированием"""

    start_time = time.time()
    notification_id = notification["id"]
    channel = notification["channel"]
    priority = notification.get("priority", "normal")
    user_id = notification["user_id"]

    # Log: начало обработки
    logger.info(
        "notification.processing.started",
        notification_id=notification_id,
        user_id=user_id,
        channel=channel,
        priority=priority
    )

    try:
        # Отправляем
        result = await send_notification(notification)

        elapsed = time.time() - start_time

        # Успех
        logger.info(
            "notification.delivered",
            notification_id=notification_id,
            channel=channel,
            latency_seconds=elapsed,
            provider=result.get("provider")
        )

        notifications_sent.labels(
            channel=channel,
            priority=priority,
            provider=result.get("provider")
        ).inc()

        delivery_latency.labels(channel=channel).observe(elapsed)

    except RetryableError as e:
        retry_count = notification.get("retry_count", 0)

        logger.warning(
            "notification.retry_needed",
            notification_id=notification_id,
            channel=channel,
            retry_count=retry_count,
            error=str(e)
        )

        retry_count.labels(
            channel=channel,
            reason=type(e).__name__
        ).inc()

    except Exception as e:
        elapsed = time.time() - start_time

        logger.error(
            "notification.failed",
            notification_id=notification_id,
            channel=channel,
            elapsed_seconds=elapsed,
            error=str(e),
            exc_info=True
        )

        notifications_failed.labels(
            channel=channel,
            reason=type(e).__name__
        ).inc()

# Health checks
@app.get("/health")
async def health_check():
    """System health status"""

    # Проверяем критичные компоненты
    checks = {
        "database": await check_database(),
        "redis": await check_redis(),
        "kafka": await check_kafka(),
        "fcm": await check_fcm(),
        "apns": await check_apns()
    }

    all_healthy = all(checks.values())
    status = "ok" if all_healthy else "degraded"

    return {
        "status": status,
        "components": checks,
        "timestamp": now()
    }

# Alert rules (Prometheus alerting rules format)
ALERT_RULES = """
# Alert: High delivery latency
- alert: NotificationHighLatency
  expr: histogram_quantile(0.99, notification_delivery_seconds) > 10
  for: 5m
  labels:
    severity: warning
  annotations:
    summary: "Notification delivery latency > 10s"

# Alert: Low delivery rate
- alert: NotificationLowDeliveryRate
  expr: |
    (
      rate(notifications_sent_total[5m]) - rate(notifications_failed_total[5m])
    ) / rate(notifications_sent_total[5m]) < 0.95
  for: 10m
  labels:
    severity: critical
  annotations:
    summary: "Delivery rate < 95%"

# Alert: Queue buildup
- alert: NotificationQueueBacklog
  expr: notification_queue_depth > 100000
  for: 10m
  labels:
    severity: warning
  annotations:
    summary: "Queue depth > 100K (potential slowdown)"

# Alert: High error rate
- alert: NotificationHighErrorRate
  expr: rate(notifications_failed_total[5m]) / rate(notifications_sent_total[5m]) > 0.01
  for: 5m
  labels:
    severity: critical
  annotations:
    summary: "Error rate > 1%"

# Alert: Provider unavailable
- alert: ProviderUnavailable
  expr: rate(notifications_failed_total{reason="ProviderError"}[5m]) > 0.5
  for: 2m
  labels:
    severity: critical
  annotations:
    summary: "Provider error rate > 50%"
"""

# Alerting service
class AlertManager:
    """Управление алертами"""

    async def send_alert(
        self,
        title: str,
        message: str,
        severity: str,
        context: dict = None
    ):
        """Отправить alert на мониторинг/Slack"""

        alert = {
            "title": title,
            "message": message,
            "severity": severity,
            "context": context or {},
            "timestamp": now()
        }

        # Логируем
        logger.critical(
            "alert.triggered",
            **alert
        )

        # Slack notification (для critical)
        if severity == "critical":
            await self._send_to_slack(alert)

        # Database (для истории)
        await db.alerts.insert_one(alert)

        # PagerDuty (для critical issues)
        if severity == "critical":
            await self._send_to_pagerduty(alert)

    async def _send_to_slack(self, alert: dict):
        """Отправить в Slack"""

        color = {
            "critical": "#FF0000",
            "warning": "#FFA500",
            "info": "#0099FF"
        }.get(alert["severity"])

        slack_message = {
            "attachments": [
                {
                    "color": color,
                    "title": alert["title"],
                    "text": alert["message"],
                    "fields": [
                        {
                            "title": "Severity",
                            "value": alert["severity"],
                            "short": True
                        },
                        {
                            "title": "Time",
                            "value": alert["timestamp"],
                            "short": True
                        }
                    ]
                }
            ]
        }

        async with aiohttp.ClientSession() as session:
            await session.post(
                SLACK_WEBHOOK_URL,
                json=slack_message
            )

# Dashboard query examples
DASHBOARD_QUERIES = """
# Top panel: Real-time throughput
SELECT
  channel,
  rate(notifications_sent_total[1m]) as throughput_per_second
FROM metrics

# Middle panel: Delivery rate
SELECT
  (rate(notifications_sent_total[5m]) - rate(notifications_failed_total[5m]))
  / rate(notifications_sent_total[5m]) * 100 as delivery_rate_percent

# Bottom panel: Queue depth by channel
SELECT channel, notification_queue_depth
FROM metrics

# Latency percentiles
SELECT
  channel,
  histogram_quantile(0.50, rate(notification_delivery_seconds_bucket[5m])) as p50,
  histogram_quantile(0.95, rate(notification_delivery_seconds_bucket[5m])) as p95,
  histogram_quantile(0.99, rate(notification_delivery_seconds_bucket[5m])) as p99
"""
```

**Типичные ошибки.**
1. **Только счетчики (counts)** — не видим rate и trend
2. **Отсутствие трассировки ID** — не можешь найти конкретное уведомление в логах
3. **Логирование всего** — noise, невозможно найти проблему
4. **Алерты на все** — alert fatigue, люди игнорируют
5. **Отсутствие корреляции** — не видишь что нарушилось (например: rate limit → latency spike)

**На интервью.**
Какие метрики ты будешь мониторить? Как детектировать аномалии (например, внезапный рост latency)? Как быстро обнаружить что провайдер упал? Как корреляировать события между компонентами?

*Частые follow-up вопросы:*
- Как мониторить per-user delivery rate?
- Как обнаружить DDoS к системе?
- Как track'ить P99 latency при 1M QPS?

---

## Итоговый чек-лист

При разработке notification system на интервью следует упомянуть:

**Архитектура:**
- [ ] Асинхронная обработка (очереди сообщений)
- [ ] Разделение по каналам (push, SMS, email)
- [ ] Разделение по приоритетам (urgent, normal, low)

**Надежность:**
- [ ] At-least-once доставка + retry с exponential backoff
- [ ] Обработка provider ошибок и rate limits
- [ ] DLQ для неудачных сообщений
- [ ] Idempotency keys для exactly-once (если требуется)

**Масштабирование:**
- [ ] Per-channel workers (разное количество для разных скоростей)
- [ ] Topic-based fanout для millions users
- [ ] Batch processing (gruppировка для providers)
- [ ] Кэширование preferences в Redis

**User Experience:**
- [ ] User preferences (channels, quiet hours, frequency cap)
- [ ] Delivery tracking (pending → sent → delivered → opened)
- [ ] Timezone-aware scheduling

**Операционность:**
- [ ] Мониторинг метрик (latency, success rate, queue depth)
- [ ] Структурированное логирование с трассировкой ID
- [ ] Алерты на критичные events (low delivery rate, provider issues)
- [ ] Health checks для всех компонентов

---

## См. также

- [Message Queues and Events](../00-go/11-message-queues-events.md) — работа с Kafka, RabbitMQ, паттерны асинхронной обработки
- [Event-Driven Architecture](../08-architecture/05-event-driven.md) — проектирование событийных систем
- [Rate Limiting](./02-rate-limiter.md) — глубокий разбор rate limiting алгоритмов
- [Chat Messenger](./04-chat-messenger.md) — похожая система с focus на real-time

---

[← Назад к списку тем](./README.md)
