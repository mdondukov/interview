# 12. Ride Sharing (Uber/Lyft)

**Навигация:** [Трек System Design](./README.md) · Предыдущая тема: [11-booking-system](./11-booking-system.md) · Следующая тема: [13-distributed-id](./13-distributed-id.md)

---

## Ключевые концепции

Прежде чем переходить к вопросам, разберёмся с базовыми терминами, которые будут встречаться в этом модуле.

**S2 Cells** — иерархическая система разделения земной поверхности на ячейки разного размера (от огромных до малюсеньких), которая позволяет быстро найти водителей в нужном районе. Вместо того чтобы проверять всех миллионов водителей в городе, система проверяет только водителей в ячейках рядом с пассажиром. S2 (разработана Google) даёт лучшие математические свойства для геопространственного поиска по сравнению с другими подходами. Это стандарт индустрии для ride-sharing компаний.

**Geohashing** — алгоритм кодирования географических координат (широта, долгота) в строку символов, которую легко индексировать в базе данных. Это преобразование 2D поиска в 1D поиск, что позволяет использовать обычные индексы базы данных вместо специализированных геопространственных индексов. Чем дольше строка хеша, тем более точное местоположение она представляет. Хотя проще в реализации, Geohashing имеет худшие свойства чем S2 и H3.

**QuadTree** — древовидная структура данных, которая разделяет 2D пространство на четыре квадранта рекурсивно, пока каждый лист не содержит нужное количество точек (например, водителей). Это альтернатива S2 и Geohashing, которая даёт полный контроль над реализацией и может быть оптимизирована под специфику приложения. QuadTree требует больше кода для реализации, но может быть более гибким для кастомных требований.

**ETA (Estimated Time of Arrival)** — предполагаемое время прибытия водителя до пассажира, рассчитанное на основе текущего положения, дорожного трафика и маршрута. Это критический критерий при выборе лучшего водителя для пассажира, так как пассажиры предпочитают водителей, которые приедут быстрее. ETA рассчитывается с помощью специального сервиса (часто интегрированного с Google Maps или Here) и обновляется по мере изменения трафика.

**Matching Algorithm** — алгоритм сопоставления водителя и пассажира, который балансирует три конкурирующих цели: скорость подбора (найти водителя как можно быстрее), справедливость к водителям (давать все равно возможности получить заказ) и качество услуги (выбирать лучших водителей по рейтингу). Различные стратегии включают назначение ближайшего свободного водителя, использование machine learning для предсказания принятия заказа, или комбинированные подходы.

**Surge Pricing** — динамическое повышение цены при высоком спросе относительно предложения водителей. Это экономический механизм, который балансирует спрос и предложение в пиковые часы (например, вечерние часы пик). Высокие цены стимулируют большего количества водителей выйти на дорогу, а также снижают спрос от потенциальных пассажиров. Система должна пересчитывать цены в реальном времени в зависимости от спроса.

**WebSocket** — двусторонний протокол коммуникации, который поддерживает постоянную связь между клиентом и сервером. Это необходимо для real-time обновления позиции водителя на карте у пассажира, а также для отправки уведомлений пассажиру о приближении водителя. WebSocket гораздо более эффективен чем polling, так как устраняет необходимость постоянных HTTP запросов.

**Location Tracking** — непрерывный процесс сбора и хранения географических координат (широта, долгота) водителей в реальном времени. Водители отправляют свои обновлённые координаты каждые несколько секунд через мобильное приложение, и эти данные сохраняются в быстром хранилище (обычно Redis или Cassandra). Это основа для всего геопространственного поиска и позволяет системе находить ближайших водителей.

**Dead Reckoning** — техника предсказания положения водителя между обновлениями координат на основе его скорости и направления. Вместо ожидания нового обновления координат, система может показать пассажиру примерное текущее положение водителя, которое экстраполировано из последнего известного положения. Это сокращает необходимое количество обновлений и улучшает пользовательский опыт, так как карта обновляется плавнее.

**Acceptance Rate** — процент заказов, которые водитель принял из всех предложенных ему заказов. Это метрика качества и надёжности водителя, которая используется при алгоритме матчинга: водители с высокой acceptance rate получают больше приоритета в подборе заказов. Очень низкая acceptance rate может указывать на проблемы с водителем (например, он находится в неправильном месте и часто отказывает).

---

## Q1. Как спроектировать систему поиска ближайших водителей при очень большой нагрузке?

### Зачем спрашивают.
На интервью проверяют понимание геопространственных индексов и способность решить классическую задачу масштабирования. Это наиболее сложная часть системы ride-sharing.

### Короткий ответ.
Используйте S2/H3 cells для разделения географической области на иерархические ячейки, сохраняйте водителей в Redis по ID ячейки, затем при поиске проверяйте все ячейки в радиусе запроса. Альтернатива — Redis Geo для небольших систем или QuadTree для кастомного контроля.

### Детальный разбор.

Проблема: 10M водителей обновляют местоположение каждые 3 секунды = 333K обновлений в секунду. Если хранить всё в одном индексе, произойдёт bottleneck.

Решение основано на трёх подходах:

**Подход 1: S2/H3 Hierarchical Cells**

```
┌─────────────────────────────────────┐
│   Entire Geographic Region (Level0) │
└────────┬────────┬────────┬──────────┘
         │        │        │
    ┌────▼──┐┌────▼──┐┌────▼──┐
    │Cell L1││Cell L1││Cell L1│
    └────┬──┘└────┬──┘└────┬──┘
         │        │        │
    ┌────▼──┐┌────▼──┐┌────▼──┐
    │Cell L2││Cell L2││Cell L2│
    └───────┘└───────┘└───────┘
     ~100m²   (varies with level)
```

Каждая ячейка содержит ID водителей, которые находятся в этом районе. При поиске ближайших водителей:
1. Берём координату поиска (lat, lng)
2. Вычисляем ID ячейки на нужном уровне детализации
3. Получаем водителей из этой ячейки и соседних
4. Отфильтровываем по расстоянию и доступности

**Подход 2: Redis Geo (для небольших систем)**

```
GEOADD drivers:nyc -74.0060 40.7128 driver_123
GEOADD drivers:nyc -74.0055 40.7135 driver_456
GEORADIUS drivers:nyc -74.0060 40.7128 5 km WITHDIST COUNT 10
```

Плюсы: просто, встроено в Redis, работает с любыми координатами.
Минусы: не масштабируется выше 10M+ записей, высокая CPU при больших радиусах.

**Подход 3: QuadTree (кастомное решение)**

```
        ┌─────────────────────┐
        │   Root QuadTree      │
        │  (whole map area)    │
        └──────────┬──────────┘
             ┌──┬──┬──┬──┐
           NW  NE  SW  SE
          ┌─┐ ┌─┐ ┌─┐ ┌─┐
          ├─┤ ├─┤ ├─┤ ├─┤  Level 2
          └─┘ └─┘ └─┘ └─┘
           │   │   │   │
          NW  NE  SW  SE
           (for each node)
```

Каждый узел может содержать до N водителей. Когда переполняется, делится на 4 подквадранта.

### Пример.

```python
import s2sphere

class GeospatialService:
    CELL_LEVEL = 14  # ~100m x 100m cells at this level
    
    def __init__(self, redis_client):
        self.redis = redis_client
    
    def get_cell_id(self, lat: float, lng: float) -> str:
        """Convert (lat, lng) to S2 cell ID"""
        ll = s2sphere.LatLng.from_degrees(lat, lng)
        cell = s2sphere.CellId.from_lat_lng(ll)
        return str(cell.parent(self.CELL_LEVEL).id())
    
    async def update_driver_location(self, driver_id: str, lat: float, lng: float):
        """Store driver location in S2 cell"""
        cell_id = self.get_cell_id(lat, lng)
        
        # Add driver to cell's set
        await self.redis.sadd(f"cell:{cell_id}:drivers", driver_id)
        
        # Store exact coordinates for later filtering
        await self.redis.hset(f"driver:{driver_id}", mapping={
            "lat": lat,
            "lng": lng,
            "cell_id": cell_id,
            "status": "available"
        })
        
        # TTL: drivers go offline after 5 min of no updates
        await self.redis.expire(f"driver:{driver_id}", 300)
    
    async def find_nearby_drivers(self, lat: float, lng: float, 
                                  radius_km: float = 5, 
                                  limit: int = 10) -> list:
        """Find drivers within radius using S2 cells"""
        
        # Get S2 region covering the search area
        center = s2sphere.LatLng.from_degrees(lat, lng)
        
        # Convert km to degrees (rough: 1 degree ≈ 111 km)
        radius_degrees = radius_km / 111.0
        
        cap = s2sphere.Cap.from_axis_angle(
            center.to_point(),
            s2sphere.Angle.from_degrees(radius_degrees)
        )
        
        # Get all cells covering this cap
        coverer = s2sphere.RegionCoverer()
        coverer.min_level = self.CELL_LEVEL
        coverer.max_level = self.CELL_LEVEL
        covering_cells = coverer.get_covering(cap)
        
        # Collect drivers from all cells
        all_drivers = []
        for cell_id in covering_cells:
            driver_ids = await self.redis.smembers(f"cell:{cell_id.id()}:drivers")
            all_drivers.extend(driver_ids)
        
        # Filter by exact distance and availability
        candidates = []
        for driver_id in all_drivers:
            data = await self.redis.hgetall(f"driver:{driver_id}")
            if data.get(b"status") == b"available":
                driver_lat = float(data.get(b"lat", 0))
                driver_lng = float(data.get(b"lng", 0))
                
                distance = self.haversine_distance(lat, lng, driver_lat, driver_lng)
                if distance <= radius_km:
                    candidates.append({
                        "driver_id": driver_id,
                        "distance_km": round(distance, 2),
                        "location": {"lat": driver_lat, "lng": driver_lng}
                    })
        
        # Sort by distance and return top N
        candidates.sort(key=lambda x: x["distance_km"])
        return candidates[:limit]
    
    def haversine_distance(self, lat1: float, lng1: float, 
                          lat2: float, lng2: float) -> float:
        """Calculate distance in km between two coordinates"""
        from math import radians, cos, sin, asin, sqrt
        
        lat1, lng1, lat2, lng2 = map(radians, [lat1, lng1, lat2, lng2])
        dlng = lng2 - lng1
        dlat = lat2 - lat1
        
        a = sin(dlat/2)**2 + cos(lat1) * cos(lat2) * sin(dlng/2)**2
        c = 2 * asin(sqrt(a))
        km = 6371 * c
        return km
```

### Типичные ошибки.

1. **Загружать все координаты в памяти** — неправильно масштабируется. Используйте индексы.

2. **Проверять расстояния в цикле без индекса** — O(N) на каждый запрос, где N = количество всех водителей.

3. **Не удалять старые записи о водителях** — Redis заполняется мёртвыми данными. Используйте TTL.

4. **Использовать Redis Geo на production с 10M+ записей** — будет медленным. S2/H3 лучше.

5. **Вычислять S2 cell ID неправильно** — убедитесь, что уровень детализации соответствует нужному размеру ячейки (~100-1000m).

### На интервью.

Объясните выбор: "S2 cells лучше всего, потому что:
- Иерархичны: можем масштабировать от города до района в одном индексе
- Поддерживают быстрый поиск ближайших соседей
- Меньше вычислений чем QuadTree
- Проще чем кастомная реализация

Для небольшой системы (< 100K водителей) Redis Geo достаточно. Для Uber-scale используем S2."

Follow-ups:
- Как обновить водителя с ячейки A в ячейку B? (Удалить из старой, добавить в новую)
- Что если водитель переместился быстро? (Обновляем TTL, стара запись удалится)
- Как работает покрытие S2 cap-ом? (Используем RegionCoverer для минимально необходимых ячеек)

---

## Q2. Как реализовать матчинг водителя и пассажира за < 10 секунд?

### Зачем спрашивают.
Это core функция ride-sharing. Интервьюер проверяет, знаете ли вы, как балансировать между:
- Speed (быстро найти водителя)
- Fairness (разумное распределение между водителями)
- Quality (высокие рейтинги, приемлемый ETA)

### Короткий ответ.
Параллельно ищите ближайших водителей (S2 cells), вычисляйте их ETA до пассажира, скорируйте по ETA + рейтинг + принятие заказов, выбираете топ. Весь процесс — асинхронно, занимает 1-5 сек.

### Детальный разбор.

```
Ride Request
     │
     ▼
┌─────────────────────────────┐
│ 1. Find Nearby Drivers      │  <- 500ms
│    (Geospatial index)       │
└─────────────────┬───────────┘
                  │ (20 drivers)
                  ▼
        ┌─────────────────────────────────┐
        │ 2. Fetch Driver Metadata        │  <- 200ms
        │    (Rating, acceptance_rate)    │
        └─────────────────┬───────────────┘
                          │
            ┌─────────────┬─────────────┐
            │             │             │
            ▼             ▼             ▼
        ┌──────────┐ ┌──────────┐ ┌──────────┐
        │ ETA Srv  │ │ ETA Srv  │ │ ETA Srv  │
        │ (calc)   │ │ (calc)   │ │ (calc)   │
        └────┬─────┘ └────┬─────┘ └────┬─────┘
             │            │            │
             └────────┬───┴──────┬─────┘
                      │ (3 sec) │
                      ▼
        ┌─────────────────────────────┐
        │ 3. Score Drivers            │  <- 100ms
        │    (ETA, rating, acceptance)│
        └─────────────┬───────────────┘
                      │
                      ▼
        ┌─────────────────────────────┐
        │ 4. Select Best Driver       │  <- Instant
        │    (Highest score)          │
        └─────────────┬───────────────┘
                      │
                      ▼
        ┌─────────────────────────────┐
        │ 5. Send Request to Driver   │  <- 1 sec
        │    (WebSocket push)         │
        └─────────────────────────────┘

Total: ~5 seconds
```

**Scoring Formula:**

```
Score = 100

// ETA (40% weight) — shorter is better
eta_penalty = min(eta_seconds / 600, 1.0) * 40  // 600s = 10 min max
score -= eta_penalty

// Driver Rating (20% weight)
rating_bonus = (rating - 3.5) * 20  // 3.5 baseline
score += rating_bonus

// Acceptance Rate (20% weight) — how often driver accepts
acceptance_bonus = acceptance_rate * 20
score += acceptance_bonus

// Vehicle Type Match (20% weight)
if driver.vehicle_type == request.ride_type:
    score += 20
```

Пример расчёта:
- Driver A: ETA=180s, rating=4.8, acceptance=95%
  - ETA penalty: (180/600) * 40 = 12
  - Rating bonus: (4.8-3.5)*20 = 26
  - Acceptance bonus: 0.95*20 = 19
  - Type match: 20
  - Score = 100 - 12 + 26 + 19 + 20 = 153

- Driver B: ETA=120s, rating=4.5, acceptance=85%
  - ETA penalty: (120/600)*40 = 8
  - Rating bonus: (4.5-3.5)*20 = 20
  - Acceptance bonus: 0.85*20 = 17
  - Type match: 0
  - Score = 100 - 8 + 20 + 17 = 129

Driver A wins (153 > 129) несмотря на более длинный ETA, потому что рейтинг выше.

### Пример.

```python
from dataclasses import dataclass
from typing import List
import asyncio

@dataclass
class MatchingRequest:
    ride_id: str
    pickup_lat: float
    pickup_lng: float
    destination_lat: float
    destination_lng: float
    ride_type: str  # "standard", "premium", "xl"

@dataclass
class Driver:
    driver_id: str
    lat: float
    lng: float
    rating: float
    acceptance_rate: float
    vehicle_type: str
    status: str  # "available", "busy"

class MatchingService:
    def __init__(self, location_service, eta_service, driver_repo):
        self.location_service = location_service
        self.eta_service = eta_service
        self.driver_repo = driver_repo
    
    async def match_request(self, request: MatchingRequest) -> str:
        """Match rider with best driver. Returns driver_id or None."""
        
        # Step 1: Find nearby drivers (parallelized)
        nearby_drivers = await self.location_service.find_nearby_drivers(
            lat=request.pickup_lat,
            lng=request.pickup_lng,
            radius_km=8,
            limit=30
        )
        
        if not nearby_drivers:
            return None
        
        # Step 2: Fetch full driver data
        driver_ids = [d["driver_id"] for d in nearby_drivers]
        drivers = await self.driver_repo.get_batch(driver_ids)
        
        # Step 3: Calculate ETAs in parallel
        eta_tasks = []
        for driver in drivers:
            task = self.eta_service.calculate_eta(
                origin={"lat": driver.lat, "lng": driver.lng},
                destination={"lat": request.pickup_lat, "lng": request.pickup_lng}
            )
            eta_tasks.append(task)
        
        etas = await asyncio.gather(*eta_tasks)
        
        # Step 4: Score each driver
        scored_drivers = []
        for driver, eta in zip(drivers, etas):
            score = self.calculate_score(driver, eta, request)
            scored_drivers.append((score, driver))
        
        # Step 5: Select best
        if not scored_drivers:
            return None
        
        scored_drivers.sort(reverse=True, key=lambda x: x[0])
        best_score, best_driver = scored_drivers[0]
        
        return best_driver.driver_id
    
    def calculate_score(self, driver: Driver, eta_seconds: int, 
                       request: MatchingRequest) -> float:
        """Calculate match score for a driver."""
        
        score = 100.0
        
        # Penalize longer ETA (40% weight)
        # Normalize to 10 min = max penalty
        eta_penalty = min(eta_seconds / 600.0, 1.0) * 40
        score -= eta_penalty
        
        # Reward higher rating (20% weight)
        # Baseline = 3.5 stars
        rating_bonus = max((driver.rating - 3.5) * 20, 0)
        score += rating_bonus
        
        # Reward higher acceptance rate (20% weight)
        acceptance_bonus = driver.acceptance_rate * 20
        score += acceptance_bonus
        
        # Bonus for matching vehicle type (20% weight)
        if driver.vehicle_type == request.ride_type:
            score += 20
        
        return score
```

### Типичные ошибки.

1. **Синхронные ETA запросы** — обращаетесь к maps API последовательно. Используйте asyncio.gather().

2. **Не фильтровать по доступности** — предлагаете заказ водителю, который едет. Проверьте статус.

3. **Одинаковые веса для всех метрик** — ETA должен весить 40%, а vehicle type 20%. Разные приоритеты.

4. **Отправлять запрос нескольким водителям одновременно** — конкурентные запросы, путаница. Сначала выбираем одного.

5. **Не обновлять score в реальном времени** — водитель может съехать за 5 сек. Используйте временные кэши.

### На интервью.

"Основная идея — асинхронное вычисление:
1. Параллельно ищем водителей (500ms, geo-index)
2. Параллельно считаем ETA для всех (3 sec, batch maps API)
3. Скорируем и выбираем лучшего (100ms)
4. Отправляем WebSocket запрос (1 sec)

Итого < 5 секунд вместо 10 требуемых."

Follow-ups:
- Что если все водители отклонят? (Используем следующего в очереди)
- Как балансировать fairness? (Round-robin между водителями или лотерея)
- Как обновлять рейтинги? (Кэш на 1 часу, затем re-fetch)

---

## Q3. Как реализовать эффективное предсказание ETA?

### Зачем спрашивают.
ETA — критическая компонента UX. Интервьюер проверяет знание:
- Кэширования результатов
- Работы с외부 API (maps)
- Адаптации к трафику в реальном времени

### Короткий ответ.
Разделите географию на сетку 100м x 100м, кэшируйте базовые времена проезда между ячейками, при запросе используйте кэш + корректировка на текущий трафик. Это дешевле чем вызывать maps API на каждый запрос.

### Детальный разбор.

ETA Prediction состоит из двух частей:

```
┌─────────────────────────────────────┐
│         ETA Calculation             │
├─────────────────────────────────────┤
│ 1. Base ETA (from Maps API)         │
│    ▼                                │
│    ┌──────────────────────────────┐ │
│    │ Is in cache (grid-based)?    │ │  <- 95% hit rate
│    │ TTL: 5 minutes               │ │
│    └──────────────────────────────┘ │
│                                     │
│ 2. Traffic Adjustment               │
│    ▼                                │
│    ┌──────────────────────────────┐ │
│    │ Get real-time traffic index  │ │
│    │ Multiply base ETA by factor  │ │
│    └──────────────────────────────┘ │
│                                     │
│ Result: Adjusted ETA (in seconds)   │
└─────────────────────────────────────┘

Example:
Base ETA: 300 seconds (5 min)
Traffic multiplier: 1.3x (heavy traffic)
Adjusted ETA: 300 * 1.3 = 390 seconds (6.5 min)
```

**Grid-based Caching:**

```
NYC Map divided into 100m x 100m cells:

┌──────┬──────┬──────┬──────┐
│(0,0) │(0,1) │(0,2) │(0,3) │
├──────┼──────┼──────┼──────┤
│(1,0) │(1,1) │(1,2) │(1,3) │
├──────┼──────┼──────┼──────┤
│(2,0) │(2,1) │(2,2) │(2,3) │
└──────┴──────┴──────┴──────┘

Cache key: "eta:{cell_from}:{cell_to}" = 350 seconds
Entry lifetime: 5 minutes
```

### Пример.

```python
import time
import hashlib
from math import radians, cos, sin, asin, sqrt

class ETAService:
    def __init__(self, maps_client, redis_client):
        self.maps = maps_client
        self.redis = redis_client
        self.grid_size_meters = 100  # 100m x 100m cells
        self.cache_ttl = 300  # 5 minutes
    
    def get_grid_cell(self, lat: float, lng: float) -> tuple:
        """Convert (lat, lng) to grid cell (row, col)."""
        # Use some reference point as origin (e.g., -74.0, 40.7 for NYC)
        origin_lat, origin_lng = 40.7, -74.0
        
        # Simple projection (not geodetically accurate, but good enough)
        cell_lat = int((lat - origin_lat) * 111000 / self.grid_size_meters)
        cell_lng = int((lng - origin_lng) * 111000 / self.grid_size_meters)
        
        return (cell_lat, cell_lng)
    
    def get_cache_key(self, origin: dict, destination: dict) -> str:
        """Grid-based cache key."""
        origin_cell = self.get_grid_cell(origin['lat'], origin['lng'])
        dest_cell = self.get_grid_cell(destination['lat'], destination['lng'])
        
        return f"eta:{origin_cell[0]}:{origin_cell[1]}:{dest_cell[0]}:{dest_cell[1]}"
    
    async def calculate_eta(self, origin: dict, destination: dict) -> int:
        """Calculate ETA in seconds."""
        
        cache_key = self.get_cache_key(origin, destination)
        
        # Try cache first
        cached_eta = await self.redis.get(cache_key)
        if cached_eta:
            base_eta = int(cached_eta)
        else:
            # Call maps API
            try:
                route = await self.maps.get_directions(origin, destination)
                base_eta = route['duration_seconds']
                
                # Cache result
                await self.redis.setex(
                    cache_key,
                    self.cache_ttl,
                    base_eta
                )
            except Exception as e:
                print(f"Maps API error: {e}")
                # Fallback: estimate based on distance
                distance_km = self.haversine_distance(
                    origin['lat'], origin['lng'],
                    destination['lat'], destination['lng']
                )
                base_eta = int(distance_km * 2.5 * 60)  # ~2.5 min per km
        
        # Adjust for traffic
        traffic_multiplier = await self.get_traffic_multiplier(
            origin['lat'], origin['lng']
        )
        
        adjusted_eta = int(base_eta * traffic_multiplier)
        
        # Clamp to reasonable bounds
        adjusted_eta = max(adjusted_eta, 60)  # min 1 min
        adjusted_eta = min(adjusted_eta, 3600)  # max 1 hour
        
        return adjusted_eta
    
    async def get_traffic_multiplier(self, lat: float, lng: float) -> float:
        """Get traffic multiplier for a location."""
        
        # Query traffic service (simplified)
        traffic_level = await self.get_traffic_level(lat, lng)
        
        # Map traffic level to multiplier
        multipliers = {
            "light": 0.9,      # 10% faster than average
            "normal": 1.0,     # baseline
            "moderate": 1.2,   # 20% slower
            "heavy": 1.5,      # 50% slower
            "congested": 2.0   # 2x slower
        }
        
        return multipliers.get(traffic_level, 1.0)
    
    async def get_traffic_level(self, lat: float, lng: float) -> str:
        """Get current traffic level. Could query external service."""
        # In real system, integrate with:
        # - Google Maps traffic API
        # - Waze traffic data
        # - Historical patterns
        # For now, simplified
        return "normal"
    
    def haversine_distance(self, lat1: float, lng1: float,
                          lat2: float, lng2: float) -> float:
        """Calculate distance in km."""
        lat1, lng1, lat2, lng2 = map(radians, [lat1, lng1, lat2, lng2])
        dlng = lng2 - lng1
        dlat = lat2 - lat1
        
        a = sin(dlat/2)**2 + cos(lat1) * cos(lat2) * sin(dlng/2)**2
        c = 2 * asin(sqrt(a))
        km = 6371 * c
        return km
```

### Типичные ошибки.

1. **Вызывать maps API на каждый запрос** — очень дорого ($$$). Кэшируйте!

2. **Cache с TTL = 1 час** — слишком долго, трафик меняется. Используйте 5 минут.

3. **Не учитывать трафик** — ETA = 5 минут в midnight, но 15 минут в rush hour.

4. **Кэш ключ по точным координатам** — пропускаете почти все попадания. Используйте сетку.

5. **Не обновлять ETA в процессе поездки** — сказали 10 минут, прошло 2 минуты, новый ETA всё ещё 10? Пересчитывайте.

### На интервью.

"ETA состоит из двух слоёв:

1. **Базовый ETA** (кэшированный)
   - Разбиваем город на сетку 100m x 100m
   - Предварительно вычисляем маршруты между ячейками (offline)
   - При запросе: cache lookup, TTL = 5 min
   - Hit rate: ~95%

2. **Корректировка на трафик** (реальное время)
   - Получаем traffic multiplier из Google/Waze API
   - Умножаем базовый ETA
   - Пример: base=300s, traffic_mult=1.3, final=390s

Стоимость:
- 100% от запросов — Redis lookup (10ms)
- 5% от запросов — Maps API (500ms)
- Экономия: 95% * 500ms = 475ms/100 запросов"

Follow-ups:
- Как обновлять ETA во время поездки? (Пересчитываем каждые 10 сек с текущей позиции)
- Что если водитель съехал с маршрута? (Перероутируем через maps API)
- Как учитывать пробки? (Traffic multiplier из real-time data)


---

## Q4. Как спроектировать систему динамического ценообразования (surge pricing)?

### Зачем спрашивают.
Surge pricing — это бизнес-критическая функция, которая балансирует спрос и предложение. Интервьюер проверяет:
- Понимание динамических систем
- Обработку распределённого состояния
- Предотвращение манипуляций

### Короткий ответ.
Разделите город на сетки (cell-based pricing), для каждой ячейки отслеживайте соотношение спрос/предложение в реальном времени, вычисляйте multiplier = базовая цена × (1 + (demand/supply - 1) × 0.5), применяйте smoothing для избежания скачков.

### Детальный разбор.

**Supply/Demand Model:**

```
        Demand (pending ride requests)
             /
            /
    Surge = ────────────────────────────
         Supply (available drivers)


Example:

Supply = 10 drivers available
Demand = 5 riders waiting
Ratio = 5/10 = 0.5 (oversupply)
Surge multiplier = 1.0 (no surge, normal price)

---

Supply = 10 drivers available
Demand = 25 riders waiting
Ratio = 25/10 = 2.5 (high demand)
Surge multiplier = 1.0 + (2.5 - 1.0) * 0.5 = 1.75 (75% markup)

---

Supply = 2 drivers available
Demand = 20 riders waiting
Ratio = 20/2 = 10 (extreme demand)
Surge multiplier = clamped to 3.0 (max 300% price)
```

**Cell-Based Grid:**

```
NYC divided into pricing cells (2km x 2km):

┌─────────┬─────────┬─────────┐
│ Cell A  │ Cell B  │ Cell C  │
│ surge:  │ surge:  │ surge:  │
│ 1.0x    │ 1.8x    │ 2.1x    │
├─────────┼─────────┼─────────┤
│ Cell D  │ Cell E  │ Cell F  │
│ surge:  │ surge:  │ surge:  │
│ 1.1x    │ 1.5x    │ 1.3x    │
├─────────┼─────────┼─────────┤
│ Cell G  │ Cell H  │ Cell I  │
│ surge:  │ surge:  │ surge:  │
│ 1.0x    │ 2.2x    │ 1.2x    │
└─────────┴─────────┴─────────┘

Each cell is independent. During peak hours,
busy areas (restaurants, airports) have higher surges.
```

**Smoothing to Prevent Shock:**

```
Without smoothing:
time  →
surge │     ╱╲
      │    ╱  ╲╱╲
      │   ╱      ╲
      │  ╱         ╲
      └──────────────────

With smoothing (α=0.3):
surge │    ╱ ─ ─ ─
      │   ╱ ╲ ─ ─
      │  ╱   ╲
      │ ╱     ╲
      └────────────────

new_surge = old_surge * 0.7 + calculated_surge * 0.3
```

### Пример.

```python
import time
from typing import Dict, Optional

class SurgePricingService:
    def __init__(self, redis_client):
        self.redis = redis_client
        self.min_multiplier = 1.0
        self.max_multiplier = 3.0
        self.smoothing_factor = 0.7  # old: new = 7:3
        self.cell_size_km = 2  # 2km x 2km cells
    
    def get_pricing_cell(self, lat: float, lng: float) -> str:
        """Convert (lat, lng) to pricing cell ID."""
        # NYC boundaries (rough)
        origin_lat, origin_lng = 40.7, -74.0
        cell_lat = int((lat - origin_lat) / (self.cell_size_km / 111))
        cell_lng = int((lng - origin_lng) / (self.cell_size_km / 111))
        return f"cell:{cell_lat}:{cell_lng}"
    
    async def get_available_drivers(self, cell_id: str, ride_type: str) -> int:
        """Count available drivers in cell."""
        key = f"{cell_id}:drivers:{ride_type}:available"
        count = await self.redis.scard(key)
        return count if count else 0
    
    async def get_pending_requests(self, cell_id: str, ride_type: str) -> int:
        """Count pending ride requests in cell."""
        key = f"{cell_id}:requests:{ride_type}:pending"
        count = await self.redis.scard(key)
        return count if count else 0
    
    async def calculate_surge(self, lat: float, lng: float, 
                             ride_type: str) -> float:
        """Calculate surge multiplier for location."""
        
        cell_id = self.get_pricing_cell(lat, lng)
        
        # Get supply and demand
        supply = await self.get_available_drivers(cell_id, ride_type)
        demand = await self.get_pending_requests(cell_id, ride_type)
        
        # Edge case: no supply
        if supply == 0:
            calculated = self.max_multiplier
        else:
            # Calculate demand/supply ratio
            ratio = demand / supply
            
            # Linear formula: multiplier = 1.0 + (ratio - 1.0) * 0.5
            # ratio=1.0 -> multiplier=1.0 (normal)
            # ratio=2.0 -> multiplier=1.5 (50% markup)
            # ratio=3.0 -> multiplier=2.0 (100% markup)
            
            calculated = self.min_multiplier + max(0, ratio - 1.0) * 0.5
            calculated = min(calculated, self.max_multiplier)
        
        # Get current surge (if exists)
        surge_key = f"{cell_id}:surge:{ride_type}"
        current_surge_str = await self.redis.get(surge_key)
        current_surge = float(current_surge_str) if current_surge_str else 1.0
        
        # Apply smoothing
        smoothed_surge = (current_surge * self.smoothing_factor + 
                         calculated * (1 - self.smoothing_factor))
        
        # Store new surge with 5 min TTL
        await self.redis.setex(
            surge_key,
            300,
            round(smoothed_surge, 2)
        )
        
        return smoothed_surge
    
    async def get_price_estimate(self, pickup: dict, destination: dict,
                                ride_type: str) -> dict:
        """Get price estimate with surge applied."""
        
        # Get base price (from maps/distance)
        # In real system: call maps API for route
        distance_km = self.estimate_distance(pickup, destination)
        duration_min = distance_km * 2.5  # ~2.5 min per km
        
        # Base pricing rules
        base_fares = {
            "standard": {"base": 250, "per_km": 10, "per_min": 0.50},
            "premium": {"base": 350, "per_km": 15, "per_min": 0.75},
            "xl": {"base": 400, "per_km": 20, "per_min": 1.00}
        }
        
        pricing = base_fares.get(ride_type, base_fares["standard"])
        
        # Calculate base price in cents
        base_price = (pricing["base"] + 
                     pricing["per_km"] * distance_km +
                     pricing["per_min"] * duration_min)
        
        # Apply surge
        surge = await self.calculate_surge(pickup['lat'], pickup['lng'], ride_type)
        surged_price = base_price * surge
        
        # Apply min/max
        min_price = max(surged_price * 0.9, pricing["base"])
        max_price = surged_price * 1.1
        
        return {
            "base_price": int(base_price),
            "min_price": int(min_price),
            "max_price": int(max_price),
            "surge_multiplier": round(surge, 2),
            "breakdown": {
                "base_fare": pricing["base"],
                "distance_charge": int(pricing["per_km"] * distance_km),
                "time_charge": int(pricing["per_min"] * duration_min),
                "surge_markup": int(surged_price - base_price)
            }
        }
    
    def estimate_distance(self, origin: dict, destination: dict) -> float:
        """Rough distance estimate. In production: call maps API."""
        from math import radians, cos, sin, asin, sqrt
        
        lat1, lng1 = origin['lat'], origin['lng']
        lat2, lng2 = destination['lat'], destination['lng']
        
        lat1, lng1, lat2, lng2 = map(radians, [lat1, lng1, lat2, lng2])
        dlng = lng2 - lng1
        dlat = lat2 - lat1
        
        a = sin(dlat/2)**2 + cos(lat1) * cos(lat2) * sin(dlng/2)**2
        c = 2 * asin(sqrt(a))
        return 6371 * c
    
    async def on_driver_available(self, driver_id: str, lat: float, 
                                  lng: float, ride_type: str):
        """Called when driver comes online."""
        cell_id = self.get_pricing_cell(lat, lng)
        key = f"{cell_id}:drivers:{ride_type}:available"
        await self.redis.sadd(key, driver_id)
    
    async def on_driver_busy(self, driver_id: str, lat: float, 
                            lng: float, ride_type: str):
        """Called when driver accepts a ride."""
        cell_id = self.get_pricing_cell(lat, lng)
        key = f"{cell_id}:drivers:{ride_type}:available"
        await self.redis.srem(key, driver_id)
    
    async def on_ride_requested(self, ride_id: str, lat: float, 
                               lng: float, ride_type: str):
        """Called when rider requests a ride."""
        cell_id = self.get_pricing_cell(lat, lng)
        key = f"{cell_id}:requests:{ride_type}:pending"
        await self.redis.sadd(key, ride_id)
    
    async def on_ride_matched(self, ride_id: str, lat: float, 
                             lng: float, ride_type: str):
        """Called when ride is matched."""
        cell_id = self.get_pricing_cell(lat, lng)
        key = f"{cell_id}:requests:{ride_type}:pending"
        await self.redis.srem(key, ride_id)
```

### Типичные ошибки.

1. **Слишком агрессивный surge** — multiplier прыгает с 1.0 на 2.5. Используйте smoothing.

2. **Не использовать cell-based подход** — вычисляете surge для города целиком. Должны быть локальные surges.

3. **Обновлять surge слишком часто** — каждую секунду. Обновляйте раз в 1-2 минуты.

4. **Не ограничивать максимум surge** — drivers бойкотируют города с surge > 3.0. Установите max.

5. **Манипулирование supply/demand** — водители отключаются перед peak, demand лжёт. Используйте ML-модели для предсказания.

### На интервью.

"Surge pricing использует cell-based подход:
1. Разбиваем город на ячейки 2km x 2km
2. На каждой ячейке отслеживаем demand (pending requests) и supply (available drivers)
3. Вычисляем multiplier = 1.0 + (demand/supply - 1) * 0.5
4. Применяем smoothing: new = old*0.7 + calculated*0.3
5. Зажимаем между min=1.0x и max=3.0x

Это обеспечивает:
- Инцентивы для водителей приехать в busy районы
- Справедливость (клиенты платят за спрос)
- Стабильность (не скачут цены)"

Follow-ups:
- Как обновлять surge в реальном времени? (Event-driven: водитель онлайн/офлайн, запрос создан/matched)
- Как предотвращать манипуляции? (ML для предсказания fake demand, detection patterns)
- Что если surge очень высокий? (Показываем альтернативы, может быть ждать 5 min)

---

## Q5. Как реализовать real-time tracking и location updates при 500K обновлений в секунду?

### Зачем спрашивают.
Это bottleneck для ride-sharing. Нужно обработать огромный объём данных и доставить их с низкой латенцией. Интервьюер проверяет:
- Батчинг и buffering
- WebSocket vs HTTP long-polling
- Шардирование по ride_id

### Короткий ответ.
Водители отправляют лок-апдейты батчами через HTTP (не каждые 200ms, а раз в 1-2 сек), сохраняются в Redis, пассажирам доставляются через WebSocket с батчингом (100-200 обновлений за раз). Шардируйте по ride_id для масштабирования.

### Детальный разбор.

```
                    Location Update Flow

Driver Location Updates:
├─ Batched: 1-2 seconds
├─ Contains: lat, lng, heading, speed
└─ Sent via HTTP POST

                    ▼
           ┌─────────────────┐
           │ Location Router │
           │ (API Gateway)   │
           └────────┬────────┘
                    │
                    ▼
           ┌─────────────────────────────┐
           │ Redis Stream (per ride_id)  │
           │ ride:123 → [               │
           │   {lat, lng, ts}           │
           │   {lat, lng, ts}           │
           │   {lat, lng, ts}           │
           │ ]                           │
           └────────┬────────────────────┘
                    │
        ┌───────────┴───────────┐
        ▼                       ▼
   ┌────────────┐         ┌──────────────┐
   │ WebSocket  │         │ Event Stream │
   │ Server A   │         │ Processor    │
   │ (push to   │         │ (analytics)  │
   │ riders)    │         └──────────────┘
   └────────────┘

Latency breakdown:
- HTTP POST: 50ms
- Redis write: 10ms
- WebSocket broadcast: 20ms
- Client receive: 100ms
---
Total: ~180ms end-to-end
```

**Batching Strategy:**

```
Driver sends updates every 1 second (not 200ms):

Time:
  t=0s:   Location: (40.700, -74.000)
  t=1s:   Location: (40.701, -74.001)  <- send batch with prev + current
  t=2s:   Location: (40.702, -74.002)  <- send batch
  ...

Rider receives:
  t=0.2s: Batch {
    ride_id: "123",
    locations: [
      {lat: 40.700, lng: -74.000, ts: 1000},
      {lat: 40.701, lng: -74.001, ts: 2000}
    ]
  }
```

**WebSocket Connection Pooling:**

```
Each WebSocket server handles ~10K concurrent connections:

WS Server 1 → Riders 1-10K
WS Server 2 → Riders 10K-20K
WS Server 3 → Riders 20K-30K
...

Broadcast for ride_id="123":
Redis Pub/Sub: publish("ride:123:location", {...})
All WS servers subscribe to "ride:*:location"
Each WS server sends to riders watching that ride
```

### Пример.

```python
import asyncio
import json
from typing import Dict, Set
from datetime import datetime

class LocationService:
    def __init__(self, redis_client, websocket_manager):
        self.redis = redis_client
        self.ws = websocket_manager
        self.batch_timeout = 1.0  # seconds
        self.update_batches: Dict[str, list] = {}  # ride_id -> [updates]
    
    async def batch_driver_location(self, ride_id: str, driver_id: str,
                                    lat: float, lng: float, 
                                    heading: int, speed: int):
        """Accumulate location update for batching."""
        
        if ride_id not in self.update_batches:
            self.update_batches[ride_id] = []
            # Schedule batch send
            asyncio.create_task(self._send_batch_after_delay(ride_id))
        
        self.update_batches[ride_id].append({
            "driver_id": driver_id,
            "lat": lat,
            "lng": lng,
            "heading": heading,
            "speed": speed,
            "timestamp": int(datetime.now().timestamp() * 1000)  # ms
        })
    
    async def _send_batch_after_delay(self, ride_id: str):
        """Send batch of location updates after delay."""
        await asyncio.sleep(self.batch_timeout)
        
        if ride_id in self.update_batches and self.update_batches[ride_id]:
            batch = self.update_batches[ride_id]
            await self._persist_and_broadcast(ride_id, batch)
            del self.update_batches[ride_id]
    
    async def _persist_and_broadcast(self, ride_id: str, updates: list):
        """Persist to Redis and broadcast via WebSocket."""
        
        # 1. Store in Redis Stream for history
        for update in updates:
            await self.redis.xadd(
                f"ride:{ride_id}:locations",
                {
                    "driver_id": update["driver_id"],
                    "lat": update["lat"],
                    "lng": update["lng"],
                    "heading": update["heading"],
                    "speed": update["speed"],
                    "timestamp": update["timestamp"]
                }
            )
        
        # 2. Keep latest location in Redis hash for quick lookup
        latest = updates[-1]
        await self.redis.hset(
            f"ride:{ride_id}:latest",
            mapping={
                "driver_id": latest["driver_id"],
                "lat": latest["lat"],
                "lng": latest["lng"],
                "heading": latest["heading"],
                "speed": latest["speed"],
                "timestamp": latest["timestamp"]
            }
        )
        
        # 3. Set TTL (ride expires after 4 hours)
        await self.redis.expire(f"ride:{ride_id}:latest", 14400)
        
        # 4. Broadcast via WebSocket to all subscribers
        message = {
            "type": "driver_location",
            "ride_id": ride_id,
            "updates": updates
        }
        
        await self.ws.publish_to_ride(ride_id, json.dumps(message))
    
    async def subscribe_to_ride(self, user_id: str, ride_id: str,
                               websocket):
        """Subscribe user to ride location updates."""
        
        # Send initial location
        initial = await self.redis.hgetall(f"ride:{ride_id}:latest")
        if initial:
            await websocket.send(json.dumps({
                "type": "driver_location",
                "ride_id": ride_id,
                "updates": [{
                    "driver_id": initial.get(b"driver_id", b"").decode(),
                    "lat": float(initial.get(b"lat", 0)),
                    "lng": float(initial.get(b"lng", 0)),
                    "heading": int(initial.get(b"heading", 0)),
                    "speed": int(initial.get(b"speed", 0)),
                    "timestamp": int(initial.get(b"timestamp", 0))
                }]
            }))
        
        # Subscribe to updates
        await self.ws.subscribe(user_id, ride_id, websocket)


class WebSocketManager:
    def __init__(self, redis_pubsub):
        self.pubsub = redis_pubsub
        self.subscriptions: Dict[str, Set[str]] = {}  # ride_id -> {user_ids}
        self.connections: Dict[str, list] = {}  # user_id -> [websockets]
    
    async def subscribe(self, user_id: str, ride_id: str, websocket):
        """Subscribe user to ride location channel."""
        
        if ride_id not in self.subscriptions:
            self.subscriptions[ride_id] = set()
            # Subscribe to Redis channel for this ride
            await self.pubsub.subscribe(
                f"ride:{ride_id}:location",
                lambda msg: self._broadcast_to_users(ride_id, msg)
            )
        
        self.subscriptions[ride_id].add(user_id)
        
        if user_id not in self.connections:
            self.connections[user_id] = []
        self.connections[user_id].append(websocket)
    
    async def publish_to_ride(self, ride_id: str, message: str):
        """Publish location update to Redis channel."""
        await self.pubsub.publish(f"ride:{ride_id}:location", message)
    
    async def _broadcast_to_users(self, ride_id: str, message: str):
        """Broadcast message to all users subscribed to ride."""
        
        if ride_id in self.subscriptions:
            for user_id in self.subscriptions[ride_id]:
                if user_id in self.connections:
                    for ws in self.connections[user_id]:
                        try:
                            await ws.send(message)
                        except Exception as e:
                            print(f"WebSocket send error: {e}")
```

### Типичные ошибки.

1. **Отправлять обновление на каждый location event** — 333K/sec обновлений = мёртвая сеть. Батчируйте.

2. **HTTP long-polling вместо WebSocket** — примерно 1 MB/s трафика. Используйте WebSocket.

3. **Отправлять историю всем пользователям** — каждый ride может иметь 1000+ обновлений. Отправьте только последнее.

4. **Хранить всю историю в памяти** — Будет 100 GB+ для 1M rides. Используйте Redis Streams с TTL.

5. **Не шардировать по ride_id** — один Redis instance переполнится. Используйте consistent hashing.

### На интервью.

"Location tracking масштабируется через батчинг и WebSocket:

1. **Batching** — водитель отправляет обновления раз в 1-2 сек, а не каждые 200ms
   - Reduces: 333K/sec → 50K/sec (6.6x)
   
2. **WebSocket** — bi-directional connection, идеально для pushes
   - Faster чем HTTP long-polling
   - Lower latency (~100ms vs ~500ms)
   
3. **Redis Streams** — сохраняем историю с TTL
   - Stream хранит последние N обновлений
   - Компактно, быстро, с TTL автоудаление
   
4. **Sharding** — каждый ride_id шардирован
   - ride:123 → Stream instance 1
   - ride:456 → Stream instance 2
   - Распределяет нагрузку

Latency:
- Old: 500ms (HTTP polling cycle)
- New: 100ms (WebSocket batch)
- 5x faster UX"

Follow-ups:
- Что если connection потеряется? (Reconnect и fetch от последнего timestamp)
- Как масштабировать WebSocket? (Multiple servers + Redis pub/sub)
- Как обрабатывать out-of-order updates? (Используйте timestamp для сортировки)


---

## Q6. Как реализовать ride pooling (несколько пассажиров в одной машине)?

### Зачем спрашивают.
Ride pooling — это оптимизационная задача. Интервьюер проверяет способность:
- Группировать рейды в реальном времени
- Балансировать доступность и эффективность
- Управлять сложными государствами системы

### Короткий ответ.
Когда пассажир делает запрос, ищем других рейдов в направлении (+/- 10% от маршрута), создаём пулед-рейд если находим матч, отправляем одного водителя на обе точки пикапа, скидываем цену на 20-30% за пулинг.

### Детальный разбор.

**Ride Pooling Flow:**

```
Rider A requests:
  Pickup: (40.700, -74.000)
  Destination: (40.730, -74.010)
  ▼
┌──────────────────────────────────┐
│ Find matching pending requests   │
│ in similar direction             │
│ (±10% detour, same area)        │
└────────────┬─────────────────────┘
             │
             ▼
   Rider B found:
   Pickup: (40.705, -74.001)
   Destination: (40.735, -74.012)
   │
   ├─ Distance A→B pickup: 0.8km (acceptable)
   ├─ Shared path percentage: 85% (good)
   └─ Both can save 25% (good for both)
             │
             ▼
┌──────────────────────────────────┐
│ Create pooled ride               │
│ - One driver                     │
│ - Two pickup points (A first)    │
│ - Two destinations (A first)     │
└────────────┬─────────────────────┘
             │
             ▼
   Estimate detour:
   - Baseline A→destination: 10 min
   - With pickup B: 11 min (+1 min)
   - Acceptable detour: < 5 min
             │
             ▼
   Send offers to both:
   - Rider A: $8.50 (30% savings)
   - Rider B: $7.20 (25% savings)
```

**Matching Algorithm:**

```
For each new request:

1. Get nearby pending requests (same pickup area within 1km)
2. For each candidate:
   - Calculate detour time (pickup + dropoff both)
   - If detour < 5 min and passenger agrees to pooling
   - Calculate price split
   - Offer to both riders

3. If both accept within 30 seconds:
   - Create pooled ride
   - One driver
   - Two sequential pickups/dropoffs

4. If one rejects:
   - Create single ride for the one who accepted
   - Re-match the other
```

### Пример.

```python
import asyncio
from typing import List, Optional
from dataclasses import dataclass
from datetime import datetime, timedelta

@dataclass
class RideRequest:
    rider_id: str
    pickup_lat: float
    pickup_lng: float
    destination_lat: float
    destination_lng: float
    ride_type: str
    pooling_allowed: bool

@dataclass
class PooledRide:
    ride_id: str
    driver_id: str
    riders: List[str]  # [rider_a, rider_b]
    pickups: List[tuple]  # [(lat1, lng1), (lat2, lng2)]
    dropoffs: List[tuple]
    detour_minutes: int
    price_split: dict  # {rider_a: price, rider_b: price}

class RidePoolingService:
    def __init__(self, matching_service, eta_service):
        self.matching_service = matching_service
        self.eta_service = eta_service
        self.pending_requests = {}  # rider_id -> RideRequest
        self.max_detour_minutes = 5
        self.pooling_discount = 0.25  # 25% discount
    
    async def try_pool_request(self, request: RideRequest) -> Optional[str]:
        """Try to find a pooling match for request.
        Returns: ride_id if pooled or created, None if waiting."""
        
        if not request.pooling_allowed:
            # Create solo ride
            return await self._create_solo_ride(request)
        
        # Find nearby pending requests
        similar_requests = self._find_similar_requests(request)
        
        if not similar_requests:
            # No matches, add to pending
            self.pending_requests[request.rider_id] = request
            return None
        
        # Try pooling with each candidate
        for candidate in similar_requests:
            pooled = await self._try_pool_with_candidate(request, candidate)
            if pooled:
                return pooled.ride_id
        
        # No successful pooling, add to pending
        self.pending_requests[request.rider_id] = request
        return None
    
    def _find_similar_requests(self, request: RideRequest) -> List[RideRequest]:
        """Find pending requests in similar direction."""
        
        candidates = []
        for rider_id, pending in self.pending_requests.items():
            # Must allow pooling
            if not pending.pooling_allowed:
                continue
            
            # Must be same ride type
            if pending.ride_type != request.ride_type:
                continue
            
            # Pickup must be close (within 1km)
            pickup_distance = self._haversine(
                request.pickup_lat, request.pickup_lng,
                pending.pickup_lat, pending.pickup_lng
            )
            if pickup_distance > 1.0:
                continue
            
            # Must be going in similar direction
            if self._is_similar_direction(request, pending):
                candidates.append(pending)
        
        return candidates
    
    def _is_similar_direction(self, req1: RideRequest, req2: RideRequest) -> bool:
        """Check if two requests are in similar direction."""
        
        # Simple heuristic: check if routes overlap
        # More sophisticated: use actual routing from maps API
        
        # Calculate approximate destination distance
        dest_distance = self._haversine(
            req1.destination_lat, req1.destination_lng,
            req2.destination_lat, req2.destination_lng
        )
        
        # If destinations are close (< 2km), probably same direction
        if dest_distance < 2.0:
            return True
        
        # Calculate shared path percentage
        req1_distance = self._haversine(
            req1.pickup_lat, req1.pickup_lng,
            req1.destination_lat, req1.destination_lng
        )
        
        # Rough heuristic: if destinations are close, consider similar
        shared_percentage = 1.0 - (dest_distance / req1_distance)
        if shared_percentage > 0.7:  # 70% overlap
            return True
        
        return False
    
    async def _try_pool_with_candidate(
        self, request: RideRequest, candidate: RideRequest
    ) -> Optional[PooledRide]:
        """Try to pool request with candidate."""
        
        # Calculate detour time
        # Route: driver → pickupA → pickupB → destA → destB
        
        # Get baseline time for request A
        eta_a = await self.eta_service.calculate_eta(
            {"lat": request.pickup_lat, "lng": request.pickup_lng},
            {"lat": request.destination_lat, "lng": request.destination_lng}
        )
        
        # Get baseline time for candidate B
        eta_b = await self.eta_service.calculate_eta(
            {"lat": candidate.pickup_lat, "lng": candidate.pickup_lng},
            {"lat": candidate.destination_lat, "lng": candidate.destination_lng}
        )
        
        # Estimate detour time
        # Pickup A → Pickup B → Destination A → Destination B
        detour_time = await self.eta_service.calculate_eta(
            {"lat": request.pickup_lat, "lng": request.pickup_lng},
            {"lat": candidate.pickup_lat, "lng": candidate.pickup_lng}
        ) + await self.eta_service.calculate_eta(
            {"lat": candidate.pickup_lat, "lng": candidate.pickup_lng},
            {"lat": request.destination_lat, "lng": request.destination_lng}
        ) + await self.eta_service.calculate_eta(
            {"lat": request.destination_lat, "lng": request.destination_lng},
            {"lat": candidate.destination_lat, "lng": candidate.destination_lng}
        )
        
        # Detour for A: how much extra time?
        detour_a = (detour_time - eta_a) / 60  # minutes
        
        # Detour for B: similar calculation
        detour_b = (detour_time - eta_b) / 60
        
        if detour_a > self.max_detour_minutes or detour_b > self.max_detour_minutes:
            return None  # Too much detour
        
        # Calculate pricing
        price_a = await self._calculate_price(request)
        price_b = await self._calculate_price(candidate)
        
        # Apply discount for pooling
        pooled_price_a = price_a * (1 - self.pooling_discount)
        pooled_price_b = price_b * (1 - self.pooling_discount)
        
        # Get driver
        driver_id = await self.matching_service.find_best_driver(
            pickup_lat=request.pickup_lat,
            pickup_lng=request.pickup_lng,
            num_stops=2  # Two pickups
        )
        
        if not driver_id:
            return None
        
        # Create pooled ride
        pooled = PooledRide(
            ride_id=f"pool_{request.rider_id}_{candidate.rider_id}",
            driver_id=driver_id,
            riders=[request.rider_id, candidate.rider_id],
            pickups=[
                (request.pickup_lat, request.pickup_lng),
                (candidate.pickup_lat, candidate.pickup_lng)
            ],
            dropoffs=[
                (request.destination_lat, request.destination_lng),
                (candidate.destination_lat, candidate.destination_lng)
            ],
            detour_minutes=int(max(detour_a, detour_b)),
            price_split={
                request.rider_id: int(pooled_price_a),
                candidate.rider_id: int(pooled_price_b)
            }
        )
        
        # Send offers to both riders
        await asyncio.gather(
            self._send_pool_offer(request.rider_id, pooled),
            self._send_pool_offer(candidate.rider_id, pooled)
        )
        
        # Wait for both to accept (30s timeout)
        try:
            await asyncio.wait_for(
                self._wait_for_both_confirmations(pooled),
                timeout=30.0
            )
        except asyncio.TimeoutError:
            # One or both rejected/timed out
            return None
        
        # Both accepted!
        del self.pending_requests[request.rider_id]
        del self.pending_requests[candidate.rider_id]
        
        return pooled
    
    async def _send_pool_offer(self, rider_id: str, pooled: PooledRide):
        """Send pooling offer to rider."""
        # In real system: send via WebSocket/push notification
        print(f"Offer pooled ride to {rider_id}: "
              f"${pooled.price_split[rider_id]/100:.2f}, "
              f"detour: {pooled.detour_minutes} min")
    
    async def _wait_for_both_confirmations(self, pooled: PooledRide):
        """Wait for both riders to confirm pooling."""
        # In real system: wait for user taps "Accept" button
        # Simplified: auto-accept for demo
        await asyncio.sleep(2)
    
    async def _create_solo_ride(self, request: RideRequest) -> str:
        """Create single ride (no pooling)."""
        # Call matching service to find driver
        driver_id = await self.matching_service.find_best_driver(
            request.pickup_lat, request.pickup_lng
        )
        return f"ride_{request.rider_id}_{driver_id}"
    
    async def _calculate_price(self, request: RideRequest) -> int:
        """Calculate price for request in cents."""
        # Simplified; in production call pricing service
        return 2500  # $25.00
    
    def _haversine(self, lat1: float, lng1: float,
                   lat2: float, lng2: float) -> float:
        """Distance in km."""
        from math import radians, cos, sin, asin, sqrt
        
        lat1, lng1, lat2, lng2 = map(radians, [lat1, lng1, lat2, lng2])
        dlng = lng2 - lng1
        dlat = lat2 - lat1
        
        a = sin(dlat/2)**2 + cos(lat1) * cos(lat2) * sin(dlng/2)**2
        c = 2 * asin(sqrt(a))
        return 6371 * c
```

### Типичные ошибки.

1. **Не ограничивать detour** — водитель едет 20 минут вместо 5. Установите max 5 min.

2. **Не спрашивать пассажира** — сразу создаёте пулед-рейд. Всегда спросите с предложением цены.

3. **Одинаковые скидки для всех** — может быть 10% или 50% в зависимости от detour. Вычисляйте динамически.

4. **Не отменять pooling если отклонили** — один пассажир отклонил, но second попал в "pooled ride". Создайте solo ride.

5. **Долгое ожидание ответа** — ждёте 2 минуты, пока пассажир ответит. Timeout 30 сек max.

### На интервью.

"Ride pooling работает так:
1. Когда пассажир запрашивает рейд → ищем похожих в очереди
2. Similar = близко по pickup (< 1km) + похожее направление (70%+ overlap)
3. Проверяем detour: max 5 минут для обоих
4. Расчитываем new price с 20-30% скидкой за pooling
5. Отправляем обоим предложение, ждём 30 сек
6. Если оба accept → создаём pooled ride с одним водителем
7. Если rejected → создаём solo rides для обоих

Сбережения:
- Пассажиры: 20-30% скидка
- Система: 40-50% экономия на ride (один водитель вместо двух)
- Окружение: меньше машин на дорогах"

Follow-ups:
- Что если пассажиры не согласны на pooling? (Respect their preference, solo ride)
- Как оптимизировать маршрут для 3+ пассажиров? (Travelling salesman problem, может быть slow)
- Как платить водителю за pooled ride? (Бо́льшая база, но один trip вместо двух)

---

## Q7. Как реализовать dispatch систему для масштабирования на новый город?

### Зачем спрашивают.
Масштабирование — практический вопрос, который сталкивается с реальностью: каждый город имеет разные паттерны трафика, постоянное время на дороги, пиковые часы. Интервьюер проверяет архитектурное мышление.

### Короткий ответ.
Используйте multi-region архитектуру: каждый город имеет свою базу данных, кэши и воркеры, но share одного control plane для конфигурации и аналитики. Для нового города: создайте новый Redis instance, скопируйте city-specific параметры (surge pricing rules, geofences, peak hours), запустите воркеры.

### Детальный разбор.

**Multi-Region Architecture:**

```
┌────────────────────────────────────────────────────────┐
│              Control Plane (global)                    │
│  ┌──────────────────────────────────────────────────┐  │
│  │ Configuration: city parameters, algorithms      │  │
│  │ Analytics: ride patterns, driver metrics        │  │
│  │ Billing: payment processing                     │  │
│  └──────────────────────────────────────────────────┘  │
└────────────────────────────────────────────────────────┘
           │              │              │
           ▼              ▼              ▼
┌─────────────────┐ ┌──────────────┐ ┌──────────────┐
│    NYC Region   │ │ LA Region    │ │ SF Region    │
├─────────────────┤ ├──────────────┤ ├──────────────┤
│ Database: PG    │ │ Database: PG │ │ Database: PG │
│ Cache: Redis    │ │ Cache: Redis │ │ Cache: Redis │
│ Workers: 100    │ │ Workers: 80  │ │ Workers: 50  │
│ Max surge: 3.0x │ │ Max surge:3.5│ │ Max surge:2.5│
└─────────────────┘ └──────────────┘ └──────────────┘
```

**New City Onboarding:**

```
1. Create Infrastructure (5 hours)
   ├─ PostgreSQL read-replicas (multi-az)
   ├─ Redis cluster (for geolocation, surge)
   ├─ Kafka topics for events
   └─ Monitoring + alerting

2. Load Configuration (1 hour)
   ├─ City boundaries (geofences)
   ├─ Base pricing ($/km, $/min, base fare)
   ├─ Peak hours (rush hour = 8-10am, 5-7pm)
   ├─ Surge limits (min=1.0x, max=3.0x)
   └─ Driver requirements (licenses, insurance)

3. Seed Initial Data (30 min)
   ├─ ~100 test drivers (from recruitment)
   ├─ Historical weather/traffic data
   ├─ Maps service coverage check
   └─ Payment processor (stripe/adyen) setup

4. Launch Beta (2 weeks)
   ├─ Limited riders (5K-10K)
   ├─ Monitor metrics (P95 ETA error, match rate)
   ├─ A/B test pricing rules
   └─ Collect feedback

5. Full Launch (ongoing)
   ├─ Scale drivers as demand increases
   ├─ Optimize algorithms with real data
   └─ Monitor SLA compliance
```

**City-specific Parameters:**

```python
CITY_CONFIG = {
    "nyc": {
        "bounds": {
            "min_lat": 40.5,
            "max_lat": 40.9,
            "min_lng": -74.3,
            "max_lng": -73.7
        },
        "pricing": {
            "base_fare": 250,  # cents
            "per_km": 10,
            "per_min": 0.50,
            "minimum": 500,
            "surge_multiplier_max": 3.0,
            "surge_smoothing": 0.7
        },
        "peak_hours": {
            "monday-friday": [(8, 10), (17, 19)],  # rush hours
            "saturday": [(20, 23)],  # night
            "sunday": [(18, 21)]
        },
        "matching": {
            "search_radius_km": 5,
            "max_eta_minutes": 15,
            "acceptance_rate_threshold": 0.7
        },
        "dispatch": {
            "workers": 100,
            "batch_size": 1000,
            "processing_interval_ms": 100
        }
    },
    "la": {
        # Similar structure but different values
        "bounds": {...},
        "pricing": {"base_fare": 300, ...},  # higher base in LA
        ...
    }
}
```

### Пример.

```python
class CityDispatchService:
    def __init__(self, config_manager):
        self.config = config_manager
        self.services = {}  # city_id -> service instance
    
    async def onboard_new_city(self, city_id: str, config: dict):
        """Onboard a new city."""
        
        # 1. Validate configuration
        self._validate_config(city_id, config)
        
        # 2. Create infrastructure
        await self._create_infrastructure(city_id)
        
        # 3. Load configuration
        await self.config.store_city_config(city_id, config)
        
        # 4. Initialize services
        await self._initialize_services(city_id)
        
        # 5. Start dispatch workers
        await self._start_workers(city_id, config["dispatch"]["workers"])
    
    def _validate_config(self, city_id: str, config: dict):
        """Validate city configuration."""
        
        required_keys = ["bounds", "pricing", "peak_hours", "matching", "dispatch"]
        for key in required_keys:
            if key not in config:
                raise ValueError(f"Missing required key: {key}")
        
        # Validate bounds
        bounds = config["bounds"]
        assert bounds["min_lat"] < bounds["max_lat"]
        assert bounds["min_lng"] < bounds["max_lng"]
        
        # Validate pricing
        pricing = config["pricing"]
        assert pricing["base_fare"] > 0
        assert pricing["surge_multiplier_max"] > 1.0
    
    async def _create_infrastructure(self, city_id: str):
        """Create new database and cache instances."""
        
        # Create PostgreSQL read replicas
        db_name = f"rides_{city_id}"
        # In real system: Terraform/CloudFormation
        print(f"Creating PostgreSQL database: {db_name}")
        
        # Create Redis cluster for geolocation
        redis_name = f"geo_{city_id}"
        print(f"Creating Redis cluster: {redis_name}")
        
        # Create Kafka topics for events
        kafka_topic = f"rides-{city_id}"
        print(f"Creating Kafka topic: {kafka_topic}")
    
    async def _initialize_services(self, city_id: str):
        """Initialize city-specific services."""
        
        config = await self.config.get_city_config(city_id)
        
        # Create matching service for this city
        matching_service = MatchingService(
            location_service=LocationService(city_id),
            eta_service=ETAService(city_id, config["matching"]),
            config=config["matching"]
        )
        
        # Create pricing service
        pricing_service = SurgePricingService(
            redis_client=self._get_redis(city_id),
            config=config["pricing"]
        )
        
        # Create dispatch service
        dispatch_service = DispatchWorker(
            city_id=city_id,
            matching_service=matching_service,
            pricing_service=pricing_service,
            config=config["dispatch"]
        )
        
        self.services[city_id] = {
            "matching": matching_service,
            "pricing": pricing_service,
            "dispatch": dispatch_service
        }
    
    async def _start_workers(self, city_id: str, num_workers: int):
        """Start dispatch worker threads for a city."""
        
        workers = []
        for i in range(num_workers):
            worker = DispatchWorker(
                city_id=city_id,
                worker_id=i,
                services=self.services[city_id]
            )
            workers.append(worker)
            asyncio.create_task(worker.run())
        
        print(f"Started {num_workers} workers for {city_id}")
    
    def _get_redis(self, city_id: str):
        """Get Redis instance for city."""
        # In real system: use Redis Cluster API
        return RedisClient(f"geo_{city_id}")


class DispatchWorker:
    """Worker that processes ride requests for a city."""
    
    def __init__(self, city_id: str, worker_id: int, services: dict, config: dict):
        self.city_id = city_id
        self.worker_id = worker_id
        self.services = services
        self.config = config
        self.kafka = KafkaConsumer(f"rides-{city_id}")
    
    async def run(self):
        """Main worker loop."""
        
        while True:
            # Get batch of ride requests
            requests = await self.kafka.consume_batch(
                batch_size=self.config["batch_size"],
                timeout_ms=self.config["processing_interval_ms"]
            )
            
            if not requests:
                continue
            
            # Process each request
            for request in requests:
                try:
                    await self._process_request(request)
                except Exception as e:
                    print(f"Error processing request: {e}")
                    # Log to monitoring system
    
    async def _process_request(self, request: dict):
        """Process single ride request."""
        
        ride_id = request["ride_id"]
        
        # 1. Find best driver
        driver_id = await self.services["matching"].find_best_driver(request)
        
        if not driver_id:
            # No drivers available, try again in 10 seconds
            await self.kafka.requeue(request, delay_ms=10000)
            return
        
        # 2. Calculate surge pricing
        surge = await self.services["pricing"].calculate_surge(
            request["pickup_lat"],
            request["pickup_lng"],
            request["ride_type"]
        )
        
        # 3. Create and persist ride
        ride = {
            "ride_id": ride_id,
            "driver_id": driver_id,
            "surge_multiplier": surge,
            "status": "matched"
        }
        await self._persist_ride(ride)
        
        # 4. Notify driver and rider
        await self._notify_driver_and_rider(ride)
    
    async def _persist_ride(self, ride: dict):
        """Save ride to database."""
        # In real system: SQL insert
        pass
    
    async def _notify_driver_and_rider(self, ride: dict):
        """Send notifications via WebSocket."""
        # In real system: WebSocket push
        pass
```

### Типичные ошибки.

1. **Общая база для всех городов** — город A перегружен, медленный для города B. Используйте региональные БД.

2. **Не регулировать параметры** — surge max для маленького города = 2.0x, для большого = 3.0x. Customize.

3. **Одни воркеры для всех городов** — невозможно масштабировать. Каждый город имеет собственных workers.

4. **Не тестировать перед launch** — оранжевые ошибки в production. Beta launch с 5K riders первую неделю.

5. **Игнорировать локальные паттерны** — NYC пик в 8-10am, но в LA пик в 12pm. Load city-specific peak hours.

### На интервью.

"Масштабирование на новый город:

1. **Архитектура**
   - Каждый город = своя DB + Redis + workers
   - Share: control plane (config, analytics, billing)
   
2. **Инфраструктура** (5 часов)
   - PostgreSQL read-replicas
   - Redis для geo + cache
   - Kafka для events
   
3. **Конфигурация** (1 час)
   - City bounds (geofences)
   - Pricing rules (base fare, per-km)
   - Peak hours (когда surge?
   - Surge limits (max 2-3x)
   
4. **Beta Launch** (2 недели)
   - 5-10K riders
   - Собираем metrics (match rate, ETA accuracy)
   - A/B test параметры
   
5. **Полный Launch**
   - Scale drivers, optimize
   - Monitor SLA (P95 pickup time, etc)

Одна компания смогла запустить в 50+ городах за год, используя этот подход."

Follow-ups:
- Как обновить параметры без downtime? (Feature flags, gradual rollout)
- Что если surge не работает в новом городе? (Проверить data quality, может быть fake demand)
- Как обрабатывать события города (праздники, события)? (Load special configs before)


---

## Q8. Как спроектировать систему платежей и расчётов водителям?

### Зачем спрашивают.
Платежи — это критический путь. Интервьюер проверяет:
- Обработку финансовых трансакций безопасно
- Идемпотентность (одна операция = один результат)
- Согласованность между ride price и водителем payment

### Короткий ответ.
Используйте Payment Gateway (Stripe/Adyen), сохраняйте payment intent с уникальным ключом идемпотентности, обрабатывайте webhook для подтверждения, рассчитывайте что водителю получит (ride price - commission), платите батчем один раз в день.

### Детальный разбор.

**Payment Flow:**

```
Ride Completion
     │
     ▼
┌─────────────────────────┐
│ Calculate final price   │
│ - Base fare             │
│ - Distance + time       │
│ - Surge multiplier      │
│ - Tolls/extras          │
└────────┬────────────────┘
         │
         ▼
┌─────────────────────────┐
│ Charge rider via        │  Payment Intent
│ Stripe/Adyen (idempotent)
└────────┬────────────────┘
         │ (webhook response)
         ▼
    ✓ Success → record payment
    ✗ Failure → retry 3x
    
         │
         ▼
┌─────────────────────────┐
│ Calculate driver payout │
│ - Ride price: $25       │
│ - Commission: 20%       │
│ - Driver gets: $20      │
└────────┬────────────────┘
         │
         ▼
┌─────────────────────────┐
│ Accrue to driver wallet │
│ (daily payout batch)    │
└─────────────────────────┘
```

**Idempotency:**

```
Scenario: payment webhook lost

Request 1:
  POST /charge
  {idempotency_key: "ride_123", amount: 2500}
  → Creates payment intent
  → Charges card
  → Returns 200 OK
  → But response lost!

Request 2 (retry):
  POST /charge
  {idempotency_key: "ride_123", amount: 2500}
  → Sees same idempotency_key
  → Returns cached result from Request 1
  → No double charge!

Key: Store (idempotency_key → response) in database
```

**Commission Structure:**

```
Different commission rates:

Ride Type    Base Rate  Peak Rate  Minimum
─────────────────────────────────────────
Standard     20%        25%        $1.50
Premium      15%        20%        $2.00
XL           10%        15%        $2.50
Pooled       10%        10%        $1.00

Example:
Ride price: $25.00
Ride type: Standard
Is peak: No

Commission: 25.00 * 0.20 = $5.00
Driver payout: 25.00 - 5.00 = $20.00
```

### Пример.

```python
from enum import Enum
import uuid
from dataclasses import dataclass
from datetime import datetime, timedelta

class PaymentStatus(Enum):
    PENDING = "pending"
    COMPLETED = "completed"
    FAILED = "failed"
    REFUNDED = "refunded"

@dataclass
class Payment:
    payment_id: str
    ride_id: str
    rider_id: str
    amount_cents: int  # in cents: 2500 = $25.00
    status: PaymentStatus
    idempotency_key: str  # for deduplication
    created_at: datetime
    completed_at: Optional[datetime] = None
    error: Optional[str] = None

class PaymentService:
    def __init__(self, stripe_client, db_client):
        self.stripe = stripe_client
        self.db = db_client
    
    async def charge_rider(self, ride_id: str, rider_id: str,
                          amount_cents: int) -> Payment:
        """Charge rider for completed ride."""
        
        # Create idempotency key
        idempotency_key = f"ride_{ride_id}_{uuid.uuid4()}"
        
        # Check if this payment already exists (retry)
        existing = await self.db.get_payment_by_idempotency_key(idempotency_key)
        if existing:
            return existing
        
        # Create payment intent
        try:
            payment_intent = await self.stripe.create_payment_intent(
                amount=amount_cents,
                currency="USD",
                customer_id=rider_id,
                metadata={
                    "ride_id": ride_id,
                    "idempotency_key": idempotency_key
                },
                idempotency_key=idempotency_key  # Stripe deduplication
            )
        except Exception as e:
            payment = Payment(
                payment_id=str(uuid.uuid4()),
                ride_id=ride_id,
                rider_id=rider_id,
                amount_cents=amount_cents,
                status=PaymentStatus.FAILED,
                idempotency_key=idempotency_key,
                created_at=datetime.now(),
                error=str(e)
            )
            await self.db.save_payment(payment)
            return payment
        
        # Create local payment record
        payment = Payment(
            payment_id=payment_intent["id"],
            ride_id=ride_id,
            rider_id=rider_id,
            amount_cents=amount_cents,
            status=PaymentStatus.PENDING,
            idempotency_key=idempotency_key,
            created_at=datetime.now()
        )
        await self.db.save_payment(payment)
        
        return payment
    
    async def on_payment_webhook(self, webhook_data: dict):
        """Handle payment confirmation webhook from Stripe."""
        
        event_type = webhook_data["type"]
        
        if event_type == "payment_intent.succeeded":
            payment_intent_id = webhook_data["data"]["object"]["id"]
            
            # Find local payment record
            payment = await self.db.get_payment(payment_intent_id)
            if not payment:
                return
            
            # Update status
            payment.status = PaymentStatus.COMPLETED
            payment.completed_at = datetime.now()
            await self.db.save_payment(payment)
            
            # Accrue payout to driver
            await self._accrue_driver_payout(payment)
        
        elif event_type == "payment_intent.payment_failed":
            payment_intent_id = webhook_data["data"]["object"]["id"]
            payment = await self.db.get_payment(payment_intent_id)
            if payment:
                payment.status = PaymentStatus.FAILED
                await self.db.save_payment(payment)
                # Retry logic
                await self._retry_payment(payment)
    
    async def _accrue_driver_payout(self, payment: Payment):
        """Add payment to driver's daily payout batch."""
        
        ride = await self.db.get_ride(payment.ride_id)
        
        # Calculate commission
        commission_rate = self._get_commission_rate(ride)
        commission = int(payment.amount_cents * commission_rate)
        driver_payout = payment.amount_cents - commission
        
        # Get or create daily batch
        batch = await self.db.get_or_create_payout_batch(
            driver_id=ride.driver_id,
            date=datetime.now().date()
        )
        
        # Add to batch
        batch.rides.append({
            "ride_id": payment.ride_id,
            "amount": driver_payout,
            "commission": commission
        })
        batch.total_amount += driver_payout
        batch.ride_count += 1
        
        await self.db.save_payout_batch(batch)
    
    def _get_commission_rate(self, ride) -> float:
        """Get commission rate for ride."""
        
        rates = {
            "standard": 0.20,
            "premium": 0.15,
            "xl": 0.10,
            "pooled": 0.10
        }
        
        base_rate = rates.get(ride.ride_type, 0.20)
        
        # Increase during peak hours
        if self._is_peak_hour():
            base_rate += 0.05
        
        return base_rate
    
    def _is_peak_hour(self) -> bool:
        """Check if current time is peak."""
        now = datetime.now()
        hour = now.hour
        weekday = now.weekday()  # 0=Monday, 6=Sunday
        
        # Peak: 8-10am and 5-7pm weekdays
        if weekday < 5:  # Weekday
            return (8 <= hour < 10) or (17 <= hour < 19)
        
        return False
    
    async def _retry_payment(self, payment: Payment):
        """Retry failed payment."""
        
        max_retries = 3
        retry_count = await self.db.get_retry_count(payment.payment_id)
        
        if retry_count >= max_retries:
            # Give up, notify support
            print(f"Payment {payment.payment_id} failed after {max_retries} retries")
            return
        
        # Schedule retry after 1 hour
        await self.db.schedule_retry(payment.payment_id, delay_ms=3600000)


class PayoutService:
    """Daily payout batch processor."""
    
    def __init__(self, stripe_client, db_client):
        self.stripe = stripe_client
        self.db = db_client
    
    async def process_daily_payouts(self):
        """Called daily (e.g., 2am UTC) to payout drivers."""
        
        # Get all pending payout batches from yesterday
        yesterday = (datetime.now() - timedelta(days=1)).date()
        batches = await self.db.get_payout_batches_by_date(yesterday)
        
        for batch in batches:
            try:
                await self._payout_batch(batch)
            except Exception as e:
                print(f"Payout failed for {batch.driver_id}: {e}")
                # Log to monitoring, will retry next cycle
    
    async def _payout_batch(self, batch):
        """Payout single driver batch."""
        
        driver = await self.db.get_driver(batch.driver_id)
        
        # Create transfer to driver's bank account
        transfer = await self.stripe.create_transfer(
            amount=batch.total_amount,
            destination=driver.bank_account_id,
            metadata={
                "batch_id": batch.batch_id,
                "ride_count": batch.ride_count
            }
        )
        
        # Mark batch as processed
        batch.payout_id = transfer["id"]
        batch.status = "completed"
        batch.completed_at = datetime.now()
        
        await self.db.save_payout_batch(batch)
        
        # Send notification to driver
        await self._notify_driver(driver, batch)
    
    async def _notify_driver(self, driver, batch):
        """Send payout notification to driver."""
        
        amount = batch.total_amount / 100  # Convert to dollars
        message = f"Payout of ${amount:.2f} for {batch.ride_count} rides"
        
        # Send via email/SMS
        print(f"Notify {driver.driver_id}: {message}")


@dataclass
class PayoutBatch:
    batch_id: str
    driver_id: str
    date: datetime.date
    rides: list  # [{ride_id, amount, commission}, ...]
    total_amount: int  # cents
    ride_count: int
    status: str  # "pending", "completed"
    payout_id: Optional[str] = None
    completed_at: Optional[datetime] = None
```

### Типичные ошибки.

1. **Без идемпотентности** — webhook потерялся, водитель нажал retry, двойной платёж! Используйте idempotency_key.

2. **Платить сразу** — обработка payments по одному очень медленно. Батчируйте один раз в день.

3. **Забыть комиссию** — водитель видит $25, ждёт $25, но получает $20. Вычисляйте и показывайте комиссию.

4. **Не обрабатывать失败** — payment failed, но ride отмечена как completed. Retry с exponential backoff.

5. **Хранить payment info** — ни когда не сохраняйте карточные данные. Используйте payment gateway (PCI-DSS compliant).

### На интервью.

"Payment system использует:

1. **Idempotency** — каждый charge имеет уникальный ключ
   - Stripe тоже дедублирует по idempotency_key
   - Если webhook потеряется, retry вернёт тот же результат
   
2. **Webhooks** — асинхронное подтверждение платежа
   - Stripe отправляет payment_intent.succeeded
   - Мы обновляем статус, аккумулируем к водителю
   
3. **Batching** — один раз в день платим водителям
   - Считаем total_amount за день
   - Single transfer to bank account
   - Экономим на fees ($0.30 per transaction)
   
4. **Commission** — применяем во время пayout
   - Ride type: standard 20%, premium 15%, xl 10%
   - Peak multiplier: +5% surge
   - Minimum commission: $1-2 depending on type

Стоимость:
- Payment processing: 2.9% + $0.30
- Payout fee: $0.50 per transfer
- Total: ~3.5% of revenue"

Follow-ups:
- Что если водитель не согласен с комиссией? (Transparent breakdown in app, disputable)
- Как обрабатывать chargebacks? (Stripe handles, may withhold future payouts)
- Как платить международным водителям? (Wise/Remitly для cross-border transfers)

---

## Q9. Как масштабировать систему с 1M на 10M concurrent rides?

### Зачем спрашивают.
Это тест архитектурного мышления о bottlenecks и horizontal scaling. Интервьюер проверяет понимание:
- Когда добавлять machines vs optimize code
- Database scaling strategies
- Cache coherency в distributed system

### Короткий ответ.
При 10x нагрузке: (1) шардируйте PostgreSQL по ride_id, (2) Redis по city_id, (3) добавьте message queue (Kafka) для async processing, (4) используйте read replicas для analytics, (5) кэшируйте агрессивнее (maps ETA, driver ratings).

### Детальный разбор.

**Current Architecture (1M concurrent):**

```
         API Gateway
              │
     ┌────────┼────────┐
     ▼        ▼        ▼
┌────────┐ ┌────────┐ ┌────────┐
│Service1│ │Service2│ │Service3│
└────┬───┘ └────┬───┘ └────┬───┘
     │          │          │
     └──────────┼──────────┘
                │
        ┌───────┴────────┐
        ▼                ▼
    ┌────────┐      ┌────────┐
    │PostgreSQL     │Redis   │
    │(main)         │(cache) │
    └────────┘      └────────┘
```

**Bottlenecks at 10M:**

```
Current TPS (1M concurrent):
- Ride requests: 1000 QPS
- Location updates: 500K/sec
- Payment charges: 100 QPS

Problem: Single PostgreSQL instance:
- Write capacity: ~5K TPS
- We're doing: 1000 QPS rides + 100 QPS payments
- Latency: OK (100ms)

At 10M concurrent:
- Ride requests: 10K QPS
- Location updates: 5M/sec
- Payment charges: 1K QPS

Single PG can't handle 10K+ QPS → sharding needed
Location updates don't need persistence, only cache
Payments are critical, need dedicated DB
```

**Scaling Strategy:**

```
┌──────────────────────────────────┐
│        API Gateway (LB)          │
│    10 load balancer instances    │
└──────────────────────────────────┘
           │
     ┌─────┼──────┬──────────┐
     ▼     ▼      ▼          ▼
   Service Pods (50 instances, auto-scaling)
     │
     ├─ Location writes → Kafka (async)
     ├─ Ride requests → PostgreSQL (sync)
     ├─ ETA queries → Cache (Redis)
     └─ Payments → Payment Service (sync)
     
Databases:
┌─────────────────────────────────┐
│  PostgreSQL Sharded (by city)   │
│  Shard 1 (NYC): 3TB             │
│  Shard 2 (LA):  2TB             │
│  Shard 3 (SF):  1.5TB           │
│  Read replicas: 2x per shard    │
└─────────────────────────────────┘

┌─────────────────────────────────┐
│  Redis Cluster (by city)        │
│  NYC: 200GB (hot)               │
│  LA:  150GB                     │
│  SF:  100GB                     │
└─────────────────────────────────┘

┌─────────────────────────────────┐
│  Kafka Topics                   │
│  - rides-requests (10K msg/s)   │
│  - location-updates (5M msg/s)  │
│  - payments (1K msg/s)          │
│  - analytics (all events)       │
└─────────────────────────────────┘
```

### Пример.

```python
import hashlib
from typing import Optional

class ShardingService:
    """Distribute data across PostgreSQL shards."""
    
    # NYC, LA, SF, Austin, Denver, etc.
    CITIES = ["nyc", "la", "sf", "aut", "den", "hou", "dal", "atl", "chi"]
    
    def get_ride_shard(self, ride_id: str) -> str:
        """Determine which shard stores this ride."""
        # Hash-based sharding: ride_id → shard
        hash_val = int(hashlib.md5(ride_id.encode()).hexdigest(), 16)
        shard_idx = hash_val % len(self.CITIES)
        return self.CITIES[shard_idx]
    
    def get_driver_shard(self, driver_id: str, city: str) -> str:
        """Drivers live in their city."""
        return city
    
    def get_payment_shard(self, payment_id: str) -> str:
        """Payments can be sharded by hash."""
        hash_val = int(hashlib.md5(payment_id.encode()).hexdigest(), 16)
        shard_idx = hash_val % len(self.CITIES)
        return self.CITIES[shard_idx]


class DatabasePool:
    """Connection pool manager for sharded databases."""
    
    def __init__(self):
        self.shards = {}  # city → connection pool
    
    async def get_shard_connection(self, shard: str):
        """Get connection to specific shard."""
        if shard not in self.shards:
            # Lazy init
            self.shards[shard] = await self._create_shard(shard)
        return self.shards[shard]
    
    async def _create_shard(self, shard: str):
        """Create connection pool for city."""
        
        # In production: use pgBouncer for connection pooling
        # Each shard: 100 connections max
        db_url = f"postgresql://user:pass@{shard}-db.internal/rides"
        pool = await asyncpg.create_pool(db_url, min_size=10, max_size=100)
        return pool
    
    async def query_ride(self, ride_id: str):
        """Query ride from appropriate shard."""
        
        shard = self.sharding.get_ride_shard(ride_id)
        conn = await self.get_shard_connection(shard)
        
        return await conn.fetchrow(
            "SELECT * FROM rides WHERE id = $1",
            ride_id
        )


class KafkaLocationBuffer:
    """Buffer location updates, flush to Kafka."""
    
    def __init__(self, kafka_producer):
        self.producer = kafka_producer
        self.buffer = {}  # ride_id → [updates]
        self.flush_interval = 1.0  # seconds
        self.batch_size = 1000
    
    async def add_location(self, ride_id: str, lat: float, lng: float):
        """Add location to buffer."""
        
        if ride_id not in self.buffer:
            self.buffer[ride_id] = []
            asyncio.create_task(self._flush_after_delay(ride_id))
        
        self.buffer[ride_id].append({"lat": lat, "lng": lng})
        
        # Flush if batch full
        if len(self.buffer[ride_id]) >= self.batch_size:
            await self._flush(ride_id)
    
    async def _flush_after_delay(self, ride_id: str):
        """Flush batch after delay."""
        await asyncio.sleep(self.flush_interval)
        if ride_id in self.buffer:
            await self._flush(ride_id)
    
    async def _flush(self, ride_id: str):
        """Send batch to Kafka."""
        updates = self.buffer[ride_id]
        
        # Find city from ride_id (need lookup)
        ride = await self.db.query_ride(ride_id)
        city = ride["city"]
        
        topic = f"location-updates-{city}"
        
        # Send batch
        await self.producer.send(
            topic,
            {
                "ride_id": ride_id,
                "updates": updates,
                "timestamp": time.time()
            }
        )
        
        del self.buffer[ride_id]


class CacheWarming:
    """Pre-load frequently accessed data."""
    
    def __init__(self, redis_client):
        self.redis = redis_client
    
    async def warm_driver_ratings(self, city: str):
        """Pre-load top driver ratings into cache."""
        
        # Get top 10K drivers by rating
        drivers = await self.db.get_top_drivers(city, limit=10000)
        
        pipe = self.redis.pipeline()
        for driver in drivers:
            key = f"driver:{driver['id']}:rating"
            pipe.setex(key, 86400, driver['rating'])  # 24h TTL
        
        await pipe.execute()
    
    async def warm_eta_cache(self, city: str):
        """Pre-compute and cache common ETAs."""
        
        # Get top 1000 routes in city
        routes = await self.analytics.get_top_routes(city, limit=1000)
        
        for route in routes:
            cache_key = self._get_eta_cache_key(
                route["from_lat"], route["from_lng"],
                route["to_lat"], route["to_lng"]
            )
            
            # Pre-compute ETA
            eta = await self.maps.get_eta(route)
            
            await self.redis.setex(
                cache_key,
                300,  # 5 min TTL
                eta
            )
```

### Типичные ошибки.

1. **Вертикальное масштабирование** — купить большую машину вместо шардирования. Неправильно масштабируется.

2. **Одна база для всех** — при 10x нагрузке сломается. Шардируйте по city/region.

3. **Все в памяти** — Redis заполнится. Используйте TTL и батчинг для location updates.

4. **Нет read replicas** — analytics queries перегружают main DB. Используйте replicas.

5. **Кэширование не стратегическое** — кэшируйте ETA, ratings, но не ride status (меняется часто).

### На интервью.

"Масштабирование с 1M на 10M concurrent:

1. **Database Sharding** (город-базированный)
   - NYC → Shard 1 (replicas in 2 zones)
   - LA → Shard 2
   - SF → Shard 3
   - Reduced per-shard QPS: 10K → 1K-2K

2. **Message Queue** (for async work)
   - Location updates → Kafka (no DB write)
   - Payment processing → separate queue
   - Decouples front-end from back-end

3. **Caching** (aggressive)
   - ETA: grid-based, 5 min TTL
   - Driver ratings: 1 day TTL, warm-up
   - City configs: immutable, cache forever

4. **Read Replicas** (for analytics)
   - Main DB: writes only
   - Replica 1: real-time queries
   - Replica 2: analytics (lagged OK)

5. **Connection Pooling** (pgBouncer)
   - 100 max connections per shard
   - Prevents connection exhaustion

Результат:
- Old: 1 DB, 5K QPS limit → latency issues at 1M
- New: 9 shards, 2K QPS each → can handle 10M+ rides
- Cost: ~3x more infrastructure, but 10x capacity"

Follow-ups:
- Как распределять новый шард при росте? (Consistent hashing, double sharding)
- Что если один sharde станет hot? (Split that shard, rebalance)
- Как делать cross-shard queries? (Fanout to all shards, merge results)

---

## Q10. Как обеспечить высокую доступность (99.99% uptime)?

### Зачем спрашивают.
Ride-sharing требует высокой доступности. Интервьюер проверяет:
- Redundancy на всех уровнях
- Graceful degradation (лучше 50% function чем 0%)
- Monitoring и alerting

### Короткий ответ.
Мульти-зонная развёртка (3+ AZs), active-active services, БД реплики с failover, circuit breakers для external APIs, graceful degradation (matching работает со старыми ratings если свежие недоступны).

### Детальный разбор.

**Redundancy Architecture:**

```
┌─────────────────────────────────────────────────────────┐
│               DNS (GeoDNS)                              │
│  nyc.uber.com → US-East-1a, US-East-1b, US-East-1c    │
└─────────────────────────────────────────────────────────┘
             │                 │                 │
        AZ 1a             AZ 1b             AZ 1c
             │                 │                 │
    ┌────────┼─────────┐  ┌────────┬─────────┐  ┌──────────┐
    │                  │  │        │         │  │          │
┌───▼──┐         ┌────▼──┘  ┌───┬──┴──┐    ┌──┴──┐   ┌────▼───┐
│ API  │         │ API      │ API    │    │ API │   │ API    │
│Pod 1 │         │Pod 2     │Pod 3   │    │Pod 4│   │Pod 5   │
└──────┘         └────┬─────┘   └────┬─┘    └─┬──┘   └────┬───┘
     │                │              │        │            │
     └────────────────┼──────────────┼────────┼────────────┘
                      ▼
            PostgreSQL Primary (1a)
                   │
        ┌──────────┴──────────┐
        ▼                     ▼
    Replica (1b)         Replica (1c)
    (sync, 0-lag)        (async, <1s lag)
```

**Graceful Degradation:**

```
Normal operation:
  Fresh driver ratings → Calculate score → Match

When rating service down:
  Try fresh ratings (fail)
  → Fall back to cached ratings (1h old)
  → Match with stale data

When location service down:
  Try to find nearby drivers (fail)
  → Fall back to "any available drivers in city"
  → Offer to everyone, let them decide

When ETA service down:
  Try to calculate ETA (fail)
  → Use estimated ETA (distance / avg speed)
  → Show "estimated time" instead of accurate
```

**Circuit Breaker Pattern:**

```
State: CLOSED (normal)
  └─ Requests pass through
  └─ Failure count = 0

After 5 consecutive failures:
State: OPEN (failing)
  └─ Requests fail immediately
  └─ No call to external service
  └─ Prevent cascading failures

After 30 seconds (timeout):
State: HALF_OPEN (testing)
  └─ Allow 1 request to test service
  └─ If succeeds → back to CLOSED
  └─ If fails → back to OPEN
```

### Пример.

```python
from enum import Enum
import time
import asyncio
from typing import Optional

class CircuitBreakerState(Enum):
    CLOSED = "closed"      # Normal
    OPEN = "open"          # Failing
    HALF_OPEN = "half_open"  # Testing

class CircuitBreaker:
    def __init__(self, failure_threshold: int = 5, 
                 timeout_seconds: int = 30,
                 half_open_max_calls: int = 1):
        self.failure_threshold = failure_threshold
        self.timeout_seconds = timeout_seconds
        self.half_open_max_calls = half_open_max_calls
        
        self.state = CircuitBreakerState.CLOSED
        self.failure_count = 0
        self.success_count = 0
        self.last_failure_time = None
    
    async def call(self, func, *args, **kwargs):
        """Execute function with circuit breaker protection."""
        
        if self.state == CircuitBreakerState.OPEN:
            # Check if we should transition to HALF_OPEN
            if time.time() - self.last_failure_time > self.timeout_seconds:
                self.state = CircuitBreakerState.HALF_OPEN
                self.success_count = 0
            else:
                raise Exception("Circuit breaker is OPEN")
        
        try:
            result = await func(*args, **kwargs)
            
            if self.state == CircuitBreakerState.HALF_OPEN:
                self.success_count += 1
                if self.success_count >= self.half_open_max_calls:
                    # Service recovered
                    self.state = CircuitBreakerState.CLOSED
                    self.failure_count = 0
            
            return result
        
        except Exception as e:
            self.failure_count += 1
            self.last_failure_time = time.time()
            
            if self.failure_count >= self.failure_threshold:
                self.state = CircuitBreakerState.OPEN
            
            raise


class ResilientMatchingService:
    def __init__(self, location_service, rating_service):
        self.location_service = location_service
        self.rating_service = rating_service
        
        self.location_breaker = CircuitBreaker()
        self.rating_breaker = CircuitBreaker()
        
        self.cache = {}  # Fallback cache
    
    async def find_best_driver(self, pickup_lat: float, pickup_lng: float):
        """Find best driver with graceful degradation."""
        
        # Try fresh driver ratings with circuit breaker
        driver_ratings = None
        try:
            driver_ratings = await self.rating_breaker.call(
                self.rating_service.get_fresh_ratings
            )
        except Exception as e:
            print(f"Fresh ratings unavailable: {e}")
            # Fall back to cached
            driver_ratings = await self.rating_service.get_cached_ratings()
        
        # Try location service
        try:
            nearby = await self.location_breaker.call(
                self.location_service.find_nearby,
                pickup_lat, pickup_lng
            )
        except Exception as e:
            print(f"Location service unavailable: {e}")
            # Fall back to city-wide search
            nearby = await self.location_service.find_any_available_in_city(
                city=self._get_city(pickup_lat, pickup_lng)
            )
        
        if not nearby or not driver_ratings:
            raise Exception("Cannot find driver")
        
        # Score drivers (with fallback for missing ratings)
        scored = []
        for driver in nearby:
            rating = driver_ratings.get(driver["id"], 4.0)  # Default 4.0
            score = self._calculate_score(driver, rating)
            scored.append((score, driver))
        
        scored.sort(reverse=True)
        return scored[0][1]["id"]
    
    def _calculate_score(self, driver, rating):
        """Calculate driver score."""
        score = rating * 100
        # Add other factors
        return score


class FailoverManager:
    """Manage database failover."""
    
    def __init__(self, primary_url: str, replicas: list):
        self.primary_url = primary_url
        self.replica_urls = replicas
        self.current_url = primary_url
        self.connection = None
    
    async def get_connection(self):
        """Get active database connection."""
        
        if self.connection is None or not await self._is_healthy():
            # Current connection is bad, try next
            await self._failover()
        
        return self.connection
    
    async def _is_healthy(self) -> bool:
        """Check if current connection is healthy."""
        try:
            await asyncio.wait_for(
                self.connection.fetchone("SELECT 1"),
                timeout=5.0
            )
            return True
        except:
            return False
    
    async def _failover(self):
        """Switch to next available connection."""
        
        candidates = [self.primary_url] + self.replica_urls
        
        for url in candidates:
            try:
                conn = await asyncpg.connect(url, timeout=5)
                await conn.fetchone("SELECT 1")  # Test
                
                self.connection = conn
                self.current_url = url
                print(f"Failover to {url}")
                return
            except:
                continue
        
        # All connections failed
        raise Exception("All databases unavailable")


class HealthCheck:
    """Liveness and readiness probes."""
    
    def __init__(self, services: dict):
        self.services = services  # {name: service_instance}
    
    async def liveness(self) -> bool:
        """Process is alive (can be restarted)?"""
        # Usually just check if we're running
        return True
    
    async def readiness(self) -> bool:
        """Process can accept traffic?"""
        
        # Check critical dependencies
        checks = {
            "database": await self._check_database(),
            "redis": await self._check_redis(),
            "kafka": await self._check_kafka()
        }
        
        # Need 2/3 healthy
        healthy_count = sum(1 for v in checks.values() if v)
        return healthy_count >= 2
    
    async def _check_database(self) -> bool:
        try:
            conn = await self.services["db"].get_connection()
            await asyncio.wait_for(conn.fetchone("SELECT 1"), timeout=2)
            return True
        except:
            return False
    
    async def _check_redis(self) -> bool:
        try:
            await asyncio.wait_for(
                self.services["redis"].ping(),
                timeout=2
            )
            return True
        except:
            return False
    
    async def _check_kafka(self) -> bool:
        try:
            # Kafka: check if we can list topics
            topics = await asyncio.wait_for(
                self.services["kafka"].list_topics(),
                timeout=2
            )
            return len(topics) > 0
        except:
            return False
```

### Типичные ошибки.

1. **Одна AZ** — AZ падает, сервис недоступен. Используйте 3+ AZs.

2. **Синхронная репликация с большой lag** — если primary падает, replica отстаёт на 10 минут. Используйте sync replicas.

3. **Нет graceful degradation** — rating сервис down = entire matching down. Fall back to stale data.

4. **Нет circuit breakers** — external service медленный, timeout 30 сек, requests скапливаются. Circuit breaker fails fast.

5. **Не мониторить readiness** — K8s doesn't know pod is unhealthy, still sends traffic. Implement readiness probe.

### На интервью.

"99.99% uptime (52 минуты downtime в год) требует:

1. **Мультизонная развёртка**
   - 3+ Availability Zones
   - Active-active (не active-passive)
   - Any AZ can be lost, system works

2. **Database Failover**
   - Primary в AZ-1a
   - Sync replica в AZ-1b (zero RPO)
   - Async replica в AZ-1c (backup)
   - Auto-failover: <1 minute detection + failover

3. **Graceful Degradation**
   - Matching без fresh ratings? Use 1-hour-old cache
   - Location service down? Search all drivers in city
   - ETA service down? Use estimated ETA
   - Never fail if we can serve degraded

4. **Circuit Breakers**
   - External API fails 5x → OPEN circuit
   - Return cached/default response immediately
   - After 30 sec, try again (HALF_OPEN)
   - Prevents cascading failures

5. **Health Checks**
   - Liveness: process running?
   - Readiness: can accept traffic?
   - K8s uses these for traffic routing
   - If not ready, remove from LB

Результат:
- Primary fails: <1 min to detect + failover
- AZ fails: instant (active in other AZs)
- External API fails: graceful degradation (cached)
- Overall uptime: 99.99%"

Follow-ups:
- Как протестировать failover? (Chaos engineering: kill random pods/AZs)
- Что если данные рассинхронизировались? (Event log для recovery)
- Как обновить без downtime? (Blue-green deployment, canary)

---

## Итоговая таблица сложности

| Компонент | Сложность | Масштабирование |
|-----------|-----------|-----------------|
| Geospatial indexing | Высокая | S2/H3 cells, 10M+ водителей |
| Matching algorithm | Средняя | Параллельные ETA, scoring |
| ETA prediction | Средняя | Grid-based cache, traffic factors |
| Surge pricing | Средняя | Cell-based, smoothing |
| Real-time tracking | Высокая | WebSocket, Redis Streams, 500K/sec |
| Ride pooling | Высокая | Matching + TSP optimization |
| Dispatch system | Средняя | Multi-region, workers per city |
| Payment processing | Высокая | Idempotency, batching, webhook |
| Scaling | Очень высокая | Sharding, circuit breakers, degradation |
| High availability | Очень высокая | Multi-AZ, failover, SLA monitoring |

---

## Резюме по интервью

### Ключевые компоненты системы:
1. **Geospatial**: S2/H3 для индексирования водителей
2. **Matching**: Параллельные ETA + мульти-критериальное скоринг
3. **ETA**: Grid cache + traffic adjustment
4. **Pricing**: Cell-based surge с smoothing
5. **Tracking**: Batched WebSocket updates
6. **Payments**: Idempotent charges + daily payout batches
7. **Scaling**: Sharding by city, circuit breakers, graceful degradation
8. **HA**: Multi-AZ, active-active, readiness probes

### Типичный ход интервью:
- Start: "Как спроектировать ride-sharing?"
- Диаграмма: API → Services → DB/Cache
- Deep dive: Geospatial + matching (собирают 15 мин)
- Scale it: 1M → 10M (sharding, messaging)
- HA: 99.99% uptime (failover, degradation)
- Trade-offs: consistency vs availability, latency vs cost

---

**Навигация:** [Трек System Design](./README.md) · Предыдущая тема: [11-booking-system](./11-booking-system.md) · Следующая тема: [13-distributed-id](./13-distributed-id.md)

