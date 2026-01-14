# 11. Booking System (Hotel/Flights/Events)

[← Назад к списку тем](README.md)

---

## Требования

### Функциональные
- Search available inventory
- View details (hotel rooms, flight seats)
- Book/reserve items
- Cancel bookings
- Payment integration
- Notifications (confirmation, reminders)

### Нефункциональные
- No double booking (strong consistency)
- High availability for search
- Low latency (< 200ms for search)
- Handle high concurrency (flash sales)
- Support for overbooking rules

---

## Capacity Estimation

```
Hotel booking system:
- 500K hotels, 100 rooms each = 50M room-nights
- 10M searches/day
- 500K bookings/day
- Peak: 5x during holidays

Search QPS: 10M / 86400 ≈ 115 QPS (avg), 600 QPS (peak)
Booking QPS: 500K / 86400 ≈ 6 QPS (avg), 30 QPS (peak)

Storage:
- Hotel metadata: 500K × 10KB = 5GB
- Room inventory: 50M × 365 days × 100B = 1.8TB
- Bookings: 500K × 2KB × 365 = 365GB/year
```

---

## High-Level Design

```
┌──────────┐     ┌─────────────┐     ┌──────────────────┐
│  Client  │────▶│ API Gateway │────▶│   Search Service │
└──────────┘     └─────────────┘     └────────┬─────────┘
                       │                      │
                       │               ┌──────▼──────┐
                       │               │Elasticsearch│
                       │               │  (Search)   │
                       │               └─────────────┘
                       │
                       ▼
               ┌──────────────┐     ┌──────────────────┐
               │   Booking    │────▶│  Inventory       │
               │   Service    │     │  Service         │
               └──────┬───────┘     └────────┬─────────┘
                      │                      │
               ┌──────▼───────┐      ┌───────▼────────┐
               │   Booking    │      │   Inventory    │
               │      DB      │      │      DB        │
               │ (PostgreSQL) │      │  (PostgreSQL)  │
               └──────────────┘      └────────────────┘
```

---

## API Design

### Search
```json
GET /api/v1/hotels/search?
    location=NYC&
    checkin=2024-06-01&
    checkout=2024-06-03&
    guests=2&
    rooms=1

Response:
{
    "hotels": [
        {
            "hotel_id": "htl_123",
            "name": "Grand Hotel",
            "rating": 4.5,
            "location": {...},
            "rooms": [
                {
                    "room_type_id": "rt_456",
                    "name": "Deluxe King",
                    "price_per_night": 19900,
                    "available_rooms": 5
                }
            ]
        }
    ],
    "total_results": 150,
    "next_cursor": "..."
}
```

### Create Booking
```json
POST /api/v1/bookings
X-Idempotency-Key: {uuid}

Request:
{
    "hotel_id": "htl_123",
    "room_type_id": "rt_456",
    "checkin": "2024-06-01",
    "checkout": "2024-06-03",
    "guest": {
        "name": "John Doe",
        "email": "john@example.com",
        "phone": "+1234567890"
    },
    "payment_method_id": "pm_xxx"
}

Response:
{
    "booking_id": "bk_abc123",
    "status": "confirmed",
    "confirmation_code": "ABCD1234",
    "total_amount": 39800,
    "currency": "USD"
}
```

---

## Data Model

### Inventory (PostgreSQL)
```sql
CREATE TABLE hotels (
    id              UUID PRIMARY KEY,
    name            VARCHAR(255) NOT NULL,
    description     TEXT,
    location        GEOGRAPHY(POINT),
    address         JSONB,
    star_rating     DECIMAL(2,1),
    amenities       TEXT[],
    created_at      TIMESTAMP DEFAULT NOW()
);

CREATE TABLE room_types (
    id              UUID PRIMARY KEY,
    hotel_id        UUID REFERENCES hotels(id),
    name            VARCHAR(100),
    description     TEXT,
    max_occupancy   INT,
    amenities       TEXT[],
    base_price      INT  -- cents
);

-- Critical: Inventory per date
CREATE TABLE room_inventory (
    id              UUID PRIMARY KEY,
    hotel_id        UUID REFERENCES hotels(id),
    room_type_id    UUID REFERENCES room_types(id),
    date            DATE NOT NULL,
    total_rooms     INT NOT NULL,
    booked_rooms    INT DEFAULT 0,
    blocked_rooms   INT DEFAULT 0,  -- maintenance, etc.
    price           INT,  -- dynamic pricing

    UNIQUE (room_type_id, date),
    INDEX idx_hotel_date (hotel_id, date),

    CONSTRAINT available_check CHECK (booked_rooms + blocked_rooms <= total_rooms)
);
```

### Bookings
```sql
CREATE TABLE bookings (
    id                  UUID PRIMARY KEY,
    idempotency_key     VARCHAR(100) UNIQUE,
    confirmation_code   VARCHAR(20) UNIQUE,
    hotel_id            UUID REFERENCES hotels(id),
    room_type_id        UUID REFERENCES room_types(id),
    checkin_date        DATE NOT NULL,
    checkout_date       DATE NOT NULL,
    guest_name          VARCHAR(255),
    guest_email         VARCHAR(255),
    guest_phone         VARCHAR(50),
    status              VARCHAR(20),  -- pending, confirmed, cancelled, completed
    total_amount        INT,
    currency            VARCHAR(3),
    payment_id          UUID,
    created_at          TIMESTAMP DEFAULT NOW(),
    updated_at          TIMESTAMP,

    INDEX idx_confirmation (confirmation_code),
    INDEX idx_guest_email (guest_email),
    INDEX idx_hotel_dates (hotel_id, checkin_date, checkout_date)
);

-- Individual night reservations
CREATE TABLE booking_nights (
    id              UUID PRIMARY KEY,
    booking_id      UUID REFERENCES bookings(id),
    inventory_id    UUID REFERENCES room_inventory(id),
    date            DATE NOT NULL,
    price           INT,

    UNIQUE (booking_id, date)
);
```

---

## Deep Dive

### 1. Preventing Double Booking

**Approach 1: Pessimistic Locking**
```python
async def create_booking(request: BookingRequest):
    async with db.transaction(isolation_level='SERIALIZABLE'):
        # Lock inventory rows for the date range
        inventory_rows = await db.execute("""
            SELECT * FROM room_inventory
            WHERE room_type_id = $1
            AND date >= $2 AND date < $3
            FOR UPDATE
        """, request.room_type_id, request.checkin, request.checkout)

        # Check availability
        for row in inventory_rows:
            available = row.total_rooms - row.booked_rooms - row.blocked_rooms
            if available < 1:
                raise NoAvailabilityError(row.date)

        # Create booking
        booking = await create_booking_record(request)

        # Increment booked_rooms for each night
        for row in inventory_rows:
            await db.execute("""
                UPDATE room_inventory
                SET booked_rooms = booked_rooms + 1
                WHERE id = $1
            """, row.id)

        return booking
```

**Approach 2: Optimistic Locking with Version**
```python
async def create_booking(request: BookingRequest):
    max_retries = 3

    for attempt in range(max_retries):
        try:
            # Read current state
            inventory_rows = await get_inventory(
                request.room_type_id,
                request.checkin,
                request.checkout
            )

            # Check availability
            for row in inventory_rows:
                if row.available < 1:
                    raise NoAvailabilityError(row.date)

            async with db.transaction():
                # Create booking
                booking = await create_booking_record(request)

                # Update with version check
                for row in inventory_rows:
                    result = await db.execute("""
                        UPDATE room_inventory
                        SET booked_rooms = booked_rooms + 1,
                            version = version + 1
                        WHERE id = $1 AND version = $2
                        AND (total_rooms - booked_rooms - blocked_rooms) >= 1
                    """, row.id, row.version)

                    if result.rowcount == 0:
                        raise ConcurrentModificationError()

                return booking

        except ConcurrentModificationError:
            if attempt == max_retries - 1:
                raise
            await asyncio.sleep(0.1 * (attempt + 1))
```

**Approach 3: Distributed Lock (Redis)**
```python
async def create_booking(request: BookingRequest):
    # Acquire lock on room_type + dates
    lock_key = f"booking_lock:{request.room_type_id}:{request.checkin}:{request.checkout}"

    async with redis_lock(lock_key, timeout=30):
        # Check availability
        available = await check_availability(
            request.room_type_id,
            request.checkin,
            request.checkout
        )

        if not available:
            raise NoAvailabilityError()

        # Create booking atomically
        booking = await execute_booking(request)

        return booking
```

### 2. Handling Flash Sales / High Concurrency

```python
class FlashSaleBookingService:
    """
    For events with extremely high concurrency (concerts, flash sales)
    """

    async def handle_flash_sale(self, event_id: str, request: BookingRequest):
        # 1. Rate limiting per user
        if not await self.rate_limiter.allow(request.user_id):
            raise TooManyRequestsError()

        # 2. Virtual waiting room (queue)
        position = await self.join_queue(event_id, request.user_id)

        if position > self.queue_threshold:
            return QueuedResponse(position=position, estimated_wait=position * 2)

        # 3. Token bucket for processing
        token = await self.acquire_processing_token(event_id)

        try:
            # 4. Reserve inventory (temporary hold)
            reservation = await self.reserve_inventory(event_id, request, ttl=300)

            # 5. Process payment
            payment = await self.process_payment(request)

            # 6. Confirm booking
            booking = await self.confirm_booking(reservation, payment)

            return booking

        except Exception as e:
            # Release reservation on failure
            await self.release_reservation(reservation)
            raise

        finally:
            await self.release_processing_token(token)

    async def reserve_inventory(self, event_id: str, request, ttl: int):
        """
        Temporary hold using Redis with expiration
        """
        key = f"reservation:{event_id}:{request.seat_id}"

        # Try to acquire seat
        acquired = await redis.set(key, request.user_id, nx=True, ex=ttl)

        if not acquired:
            raise SeatUnavailableError()

        return Reservation(key=key, expires_at=datetime.utcnow() + timedelta(seconds=ttl))
```

### 3. Search & Availability

```python
class HotelSearchService:
    async def search(self, query: SearchQuery) -> list:
        # 1. Get candidate hotels from Elasticsearch
        candidates = await self.es.search({
            "query": {
                "bool": {
                    "filter": [
                        {"geo_distance": {
                            "distance": "10km",
                            "location": query.location
                        }},
                        {"range": {"star_rating": {"gte": query.min_rating}}},
                        {"terms": {"amenities": query.required_amenities}}
                    ]
                }
            },
            "size": 100
        })

        hotel_ids = [h['_id'] for h in candidates['hits']['hits']]

        # 2. Check real-time availability
        availability = await self.check_availability_batch(
            hotel_ids,
            query.checkin,
            query.checkout,
            query.rooms_needed
        )

        # 3. Get pricing
        hotels_with_availability = []
        for hotel_id, avail in availability.items():
            if avail['available']:
                hotels_with_availability.append({
                    'hotel_id': hotel_id,
                    'rooms': avail['rooms'],
                    'min_price': min(r['price'] for r in avail['rooms'])
                })

        # 4. Sort and return
        return sorted(hotels_with_availability, key=lambda x: x['min_price'])

    async def check_availability_batch(self, hotel_ids, checkin, checkout, rooms_needed):
        """
        Efficient batch availability check
        """
        query = """
            SELECT
                ri.hotel_id,
                ri.room_type_id,
                rt.name,
                MIN(ri.total_rooms - ri.booked_rooms - ri.blocked_rooms) as min_available,
                ARRAY_AGG(ri.price ORDER BY ri.date) as prices
            FROM room_inventory ri
            JOIN room_types rt ON ri.room_type_id = rt.id
            WHERE ri.hotel_id = ANY($1)
            AND ri.date >= $2 AND ri.date < $3
            GROUP BY ri.hotel_id, ri.room_type_id, rt.name
            HAVING MIN(ri.total_rooms - ri.booked_rooms - ri.blocked_rooms) >= $4
        """

        results = await db.fetch(query, hotel_ids, checkin, checkout, rooms_needed)

        # Group by hotel
        availability = {}
        for row in results:
            if row['hotel_id'] not in availability:
                availability[row['hotel_id']] = {'available': True, 'rooms': []}

            availability[row['hotel_id']]['rooms'].append({
                'room_type_id': row['room_type_id'],
                'name': row['name'],
                'available': row['min_available'],
                'price': sum(row['prices'])
            })

        return availability
```

### 4. Overbooking Strategy

```python
class OverbookingManager:
    """
    Airlines and hotels often overbook to compensate for no-shows
    """

    # Historical no-show rates by segment
    NO_SHOW_RATES = {
        'economy': 0.15,
        'business': 0.08,
        'first_class': 0.05
    }

    def calculate_overbooking_limit(self, flight_class: str, total_seats: int) -> int:
        base_rate = self.NO_SHOW_RATES.get(flight_class, 0.10)

        # Adjust based on factors
        adjustments = self.get_adjustments()  # day of week, route, etc.

        effective_rate = base_rate * adjustments

        # Calculate overbooking limit
        # Using simple formula: overbook by expected no-shows
        overbook_seats = int(total_seats * effective_rate)

        # Cap at maximum allowed
        max_overbook = int(total_seats * 0.10)  # Never more than 10%

        return min(overbook_seats, max_overbook)

    async def check_overbooking_availability(self, flight_id: str, flight_class: str):
        flight = await db.get_flight(flight_id)

        total_seats = flight.seats[flight_class]
        booked = await db.count_bookings(flight_id, flight_class)

        overbook_limit = self.calculate_overbooking_limit(flight_class, total_seats)

        effective_capacity = total_seats + overbook_limit

        return booked < effective_capacity
```

### 5. Cancellation & Refunds

```python
from datetime import date

class CancellationService:
    async def cancel_booking(self, booking_id: str, reason: str):
        async with db.transaction():
            # 1. Get booking with lock
            booking = await db.get_booking_for_update(booking_id)

            if booking.status not in ['confirmed', 'pending']:
                raise InvalidCancellationError()

            # 2. Calculate refund based on policy
            refund_amount = self.calculate_refund(booking)

            # 3. Release inventory
            await self.release_inventory(booking)

            # 4. Process refund
            if refund_amount > 0:
                await self.payment_service.refund(
                    booking.payment_id,
                    refund_amount
                )

            # 5. Update booking status
            booking.status = 'cancelled'
            booking.cancelled_at = datetime.utcnow()
            booking.cancellation_reason = reason
            booking.refund_amount = refund_amount
            await db.update_booking(booking)

            # 6. Send notification
            await self.notification_service.send_cancellation(booking)

            return CancellationResult(
                booking_id=booking_id,
                refund_amount=refund_amount
            )

    def calculate_refund(self, booking) -> int:
        days_until_checkin = (booking.checkin_date - date.today()).days

        policy = booking.cancellation_policy

        if days_until_checkin >= policy.free_cancellation_days:
            return booking.total_amount  # Full refund

        if days_until_checkin >= policy.partial_refund_days:
            return int(booking.total_amount * policy.partial_refund_rate)

        return 0  # No refund
```

---

## Bottlenecks & Solutions

| Проблема | Решение |
|----------|---------|
| Double booking | Pessimistic/optimistic locking, distributed locks |
| Flash sale traffic | Queue, rate limiting, temporary reservations |
| Search latency | Elasticsearch, caching, precomputed availability |
| Inventory updates | Eventual consistency for search, strong for booking |
| Cancellation cascades | Background processing, saga pattern |

---

## Trade-offs

### Consistency vs Availability

| Scenario | Consistency | Availability | Recommendation |
|----------|-------------|--------------|----------------|
| Booking | Strong | Medium | Pessimistic lock |
| Search | Eventual | High | Cache + async sync |
| Flash sale | Strong | Lower | Queue + reservation |

### Real-time vs Cached Availability

| Approach | Accuracy | Latency | Use Case |
|----------|----------|---------|----------|
| Real-time DB | 100% | Higher | Booking flow |
| Cached (5min) | 95%+ | Low | Search results |
| Pre-aggregated | 90%+ | Lowest | Browse pages |

---

## На интервью

### Ключевые моменты
1. **Preventing double booking** — locking strategies, constraints
2. **Handling high concurrency** — queues, reservations, rate limiting
3. **Search vs booking consistency** — different requirements
4. **Inventory management** — date-based, overbooking policies

### Типичные follow-up
- Как обработать групповое бронирование (10+ комнат)?
- Как реализовать waitlist при sold out?
- Как синхронизировать с внешними системами (GDS)?
- Как обрабатывать timezone при поиске?
- Как реализовать динамическое ценообразование?

---

## См. также

- [Транзакции в базах данных](../06-databases/03-transactions.md) — блокировки и уровни изоляции для предотвращения double booking
- [Rate Limiter](./02-rate-limiter.md) — контроль нагрузки при flash sales

---

[← Назад к списку тем](README.md)
