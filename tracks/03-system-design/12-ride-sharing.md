# 12. Ride Sharing (Uber/Lyft)

[← Назад к списку тем](README.md)

---

## Требования

### Функциональные
- Request ride (pickup → destination)
- Match with nearby drivers
- Real-time tracking
- ETA calculation
- Pricing (surge pricing)
- Payment processing
- Rating system

### Нефункциональные
- Low latency matching (< 10s)
- Real-time location updates
- High availability (99.99%)
- Scale: millions of concurrent rides
- Geographic distribution

---

## Capacity Estimation

```
Users:
- 100M riders, 10M drivers
- 10M rides/day
- Peak: 1M concurrent rides

Location Updates:
- Drivers update location every 3 seconds
- 1M active drivers × (1/3) = 333K updates/sec
- Peak: 500K updates/sec

Ride Requests:
- 10M / 86400 ≈ 115 QPS average
- Peak: 1000 QPS

Storage:
- Ride record: 2KB
- 10M × 2KB × 365 = 7.3TB/year
- Location history: 100B points/day (if stored)
```

---

## High-Level Design

```
┌──────────┐     ┌─────────────┐     ┌──────────────────┐
│  Rider   │────▶│ API Gateway │────▶│   Ride Service   │
│   App    │     └─────────────┘     └────────┬─────────┘
└──────────┘                                  │
                                   ┌──────────┼──────────┐
                                   │          │          │
                            ┌──────▼───┐ ┌────▼─────┐ ┌──▼──────┐
                            │ Matching │ │  Pricing │ │   ETA   │
                            │ Service  │ │  Service │ │ Service │
                            └────┬─────┘ └──────────┘ └────┬────┘
                                 │                        │
                            ┌────▼─────┐            ┌─────▼────┐
                            │ Location │            │  Maps    │
                            │ Service  │            │  Service │
                            └────┬─────┘            └──────────┘
                                 │
┌──────────┐     ┌───────────────▼───────────────┐
│  Driver  │────▶│      Location Database        │
│   App    │     │  (Redis Geo / QuadTree)       │
└──────────┘     └───────────────────────────────┘
```

---

## API Design

### Rider APIs
```json
// Request ride
POST /api/v1/rides/request
{
    "pickup": {"lat": 40.7128, "lng": -74.0060},
    "destination": {"lat": 40.7580, "lng": -73.9855},
    "ride_type": "standard"  // standard, premium, shared
}

Response:
{
    "ride_id": "ride_abc123",
    "status": "matching",
    "estimated_price": {"min": 2500, "max": 3200, "currency": "USD"},
    "eta_pickup": 180  // seconds
}

// Get ride status
GET /api/v1/rides/{ride_id}
{
    "ride_id": "ride_abc123",
    "status": "driver_assigned",
    "driver": {
        "id": "drv_456",
        "name": "John",
        "rating": 4.8,
        "vehicle": {"make": "Toyota", "model": "Camry", "plate": "ABC123"}
    },
    "driver_location": {"lat": 40.7135, "lng": -74.0055},
    "eta_pickup": 120
}
```

### Driver APIs
```json
// Update location
PUT /api/v1/drivers/location
{
    "lat": 40.7135,
    "lng": -74.0055,
    "heading": 45,
    "speed": 15
}

// Accept ride
POST /api/v1/rides/{ride_id}/accept
{
    "driver_id": "drv_456"
}

// Update ride status
PUT /api/v1/rides/{ride_id}/status
{
    "status": "arrived_pickup"  // arrived_pickup, trip_started, trip_completed
}
```

---

## Data Model

### Location Service (Redis Geo)
```redis
# Store driver locations
GEOADD drivers:nyc -74.0060 40.7128 driver_123
GEOADD drivers:nyc -74.0055 40.7135 driver_456

# Find nearby drivers (within 5km)
GEORADIUS drivers:nyc -74.0060 40.7128 5 km WITHDIST WITHCOORD COUNT 10

# Get driver location
GEOPOS drivers:nyc driver_123
```

### Rides (PostgreSQL)
```sql
CREATE TABLE rides (
    id                  UUID PRIMARY KEY,
    rider_id            UUID NOT NULL,
    driver_id           UUID,
    ride_type           VARCHAR(20),
    status              VARCHAR(20),  -- requested, matching, driver_assigned,
                                      -- arrived, in_progress, completed, cancelled
    pickup_location     GEOGRAPHY(POINT),
    pickup_address      TEXT,
    destination_location GEOGRAPHY(POINT),
    destination_address TEXT,
    estimated_price     INT,
    final_price         INT,
    surge_multiplier    DECIMAL(3,2),
    distance_meters     INT,
    duration_seconds    INT,
    requested_at        TIMESTAMP,
    pickup_at           TIMESTAMP,
    dropoff_at          TIMESTAMP,
    created_at          TIMESTAMP DEFAULT NOW(),

    INDEX idx_rider (rider_id),
    INDEX idx_driver (driver_id),
    INDEX idx_status (status)
);

CREATE TABLE ride_locations (
    id          UUID PRIMARY KEY,
    ride_id     UUID REFERENCES rides(id),
    location    GEOGRAPHY(POINT),
    timestamp   TIMESTAMP,
    is_driver   BOOLEAN,

    INDEX idx_ride_time (ride_id, timestamp)
);
```

### Drivers (PostgreSQL + Redis)
```sql
CREATE TABLE drivers (
    id              UUID PRIMARY KEY,
    name            VARCHAR(255),
    phone           VARCHAR(50),
    email           VARCHAR(255),
    vehicle_make    VARCHAR(100),
    vehicle_model   VARCHAR(100),
    vehicle_plate   VARCHAR(20),
    vehicle_type    VARCHAR(20),  -- standard, premium, xl
    rating          DECIMAL(2,1),
    total_rides     INT DEFAULT 0,
    status          VARCHAR(20),  -- online, offline, busy
    created_at      TIMESTAMP DEFAULT NOW()
);
```

---

## Deep Dive

### 1. Geospatial Indexing

**Option A: Redis Geo (Simple)**
```python
class RedisLocationService:
    def __init__(self, redis_client):
        self.redis = redis_client

    async def update_driver_location(self, driver_id: str, lat: float, lng: float, city: str):
        # Update location
        await self.redis.geoadd(f"drivers:{city}", lng, lat, driver_id)

        # Store metadata
        await self.redis.hset(f"driver:{driver_id}", {
            "lat": lat,
            "lng": lng,
            "updated_at": time.time(),
            "status": "available"
        })

        # Set TTL for cleanup
        await self.redis.expire(f"driver:{driver_id}", 300)

    async def find_nearby_drivers(self, lat: float, lng: float, city: str,
                                  radius_km: float = 5, limit: int = 10) -> list:
        # Find drivers within radius
        results = await self.redis.georadius(
            f"drivers:{city}",
            lng, lat,
            radius_km, unit='km',
            withdist=True,
            withcoord=True,
            count=limit * 2,  # Get more, filter later
            sort='ASC'
        )

        # Filter by availability
        available = []
        for driver_id, distance, (lng, lat) in results:
            metadata = await self.redis.hgetall(f"driver:{driver_id}")
            if metadata.get('status') == 'available':
                available.append({
                    'driver_id': driver_id,
                    'distance': distance,
                    'location': {'lat': lat, 'lng': lng},
                    'metadata': metadata
                })

            if len(available) >= limit:
                break

        return available
```

**Option B: QuadTree (Custom, More Control)**
```python
class QuadTree:
    def __init__(self, boundary, capacity=4):
        self.boundary = boundary  # (x, y, width, height)
        self.capacity = capacity
        self.points = []
        self.divided = False
        self.nw = self.ne = self.sw = self.se = None

    def insert(self, point):
        if not self.contains(point):
            return False

        if len(self.points) < self.capacity:
            self.points.append(point)
            return True

        if not self.divided:
            self.subdivide()

        return (self.nw.insert(point) or self.ne.insert(point) or
                self.sw.insert(point) or self.se.insert(point))

    def query_range(self, range_boundary):
        """Find all points within a range"""
        found = []

        if not self.intersects(range_boundary):
            return found

        for point in self.points:
            if self.point_in_range(point, range_boundary):
                found.append(point)

        if self.divided:
            found.extend(self.nw.query_range(range_boundary))
            found.extend(self.ne.query_range(range_boundary))
            found.extend(self.sw.query_range(range_boundary))
            found.extend(self.se.query_range(range_boundary))

        return found
```

**Option C: S2/H3 Cells (Production Grade)**
```python
import s2sphere

class S2LocationService:
    CELL_LEVEL = 14  # ~100m x 100m cells

    def get_cell_id(self, lat: float, lng: float) -> str:
        ll = s2sphere.LatLng.from_degrees(lat, lng)
        cell = s2sphere.CellId.from_lat_lng(ll)
        return str(cell.parent(self.CELL_LEVEL).id())

    async def update_driver_location(self, driver_id: str, lat: float, lng: float):
        cell_id = self.get_cell_id(lat, lng)

        # Store in Redis set for the cell
        await self.redis.sadd(f"cell:{cell_id}", driver_id)

        # Store exact location
        await self.redis.hset(f"driver:{driver_id}", {
            "lat": lat,
            "lng": lng,
            "cell_id": cell_id
        })

    async def find_nearby_drivers(self, lat: float, lng: float, radius_km: float):
        # Get covering cells for the search area
        center = s2sphere.LatLng.from_degrees(lat, lng)
        cap = s2sphere.Cap.from_axis_angle(
            center.to_point(),
            s2sphere.Angle.from_degrees(radius_km / 111)  # ~111km per degree
        )

        coverer = s2sphere.RegionCoverer()
        coverer.min_level = self.CELL_LEVEL
        coverer.max_level = self.CELL_LEVEL
        covering = coverer.get_covering(cap)

        # Query all covering cells
        drivers = []
        for cell_id in covering:
            cell_drivers = await self.redis.smembers(f"cell:{cell_id.id()}")
            drivers.extend(cell_drivers)

        return drivers
```

### 2. Matching Algorithm

```python
class MatchingService:
    async def find_best_driver(self, ride_request) -> Driver:
        # 1. Get nearby available drivers
        nearby = await self.location_service.find_nearby_drivers(
            ride_request.pickup_lat,
            ride_request.pickup_lng,
            radius_km=5,
            limit=20
        )

        if not nearby:
            return None

        # 2. Get ETAs for all candidates
        etas = await asyncio.gather(*[
            self.eta_service.calculate_eta(
                driver['location'],
                {'lat': ride_request.pickup_lat, 'lng': ride_request.pickup_lng}
            )
            for driver in nearby
        ])

        # 3. Score each driver
        scored_drivers = []
        for driver, eta in zip(nearby, etas):
            score = self.calculate_match_score(driver, eta, ride_request)
            scored_drivers.append((score, driver, eta))

        # 4. Sort by score and get best
        scored_drivers.sort(reverse=True)

        return scored_drivers[0] if scored_drivers else None

    def calculate_match_score(self, driver: dict, eta: int, request) -> float:
        score = 100.0

        # Penalize longer ETA (weight: 40%)
        eta_penalty = min(eta / 600, 1.0) * 40  # Max 10 min
        score -= eta_penalty

        # Reward higher rating (weight: 20%)
        rating_bonus = (driver['rating'] - 4.0) * 20  # 4.0 as baseline
        score += rating_bonus

        # Reward acceptance rate (weight: 20%)
        acceptance_bonus = driver['acceptance_rate'] * 20
        score += acceptance_bonus

        # Match vehicle type preference (weight: 20%)
        if driver['vehicle_type'] == request.ride_type:
            score += 20

        return score
```

### 3. ETA Calculation

```python
class ETAService:
    def __init__(self, maps_client):
        self.maps = maps_client
        self.cache = {}

    async def calculate_eta(self, origin: dict, destination: dict) -> int:
        """Calculate ETA in seconds"""

        # Try cache first (grid-based)
        cache_key = self.get_cache_key(origin, destination)
        cached = self.cache.get(cache_key)
        if cached and time.time() - cached['time'] < 300:  # 5 min TTL
            return self.adjust_for_traffic(cached['base_eta'])

        # Call maps API
        route = await self.maps.get_directions(origin, destination)

        base_eta = route['duration_seconds']

        # Cache base ETA
        self.cache[cache_key] = {
            'base_eta': base_eta,
            'time': time.time()
        }

        return self.adjust_for_traffic(base_eta)

    def adjust_for_traffic(self, base_eta: int) -> int:
        """Adjust ETA based on real-time traffic"""
        traffic_multiplier = self.get_traffic_multiplier()
        return int(base_eta * traffic_multiplier)

    def get_cache_key(self, origin: dict, destination: dict) -> str:
        """Grid-based caching (round to ~100m)"""
        o_lat = round(origin['lat'], 3)
        o_lng = round(origin['lng'], 3)
        d_lat = round(destination['lat'], 3)
        d_lng = round(destination['lng'], 3)
        return f"{o_lat},{o_lng}:{d_lat},{d_lng}"
```

### 4. Surge Pricing

```python
class SurgePricingService:
    def __init__(self):
        self.base_multiplier = 1.0
        self.max_multiplier = 3.0

    async def calculate_surge(self, location: dict, ride_type: str) -> float:
        cell_id = self.get_pricing_cell(location)

        # Get supply/demand ratio
        supply = await self.get_available_drivers(cell_id, ride_type)
        demand = await self.get_pending_requests(cell_id, ride_type)

        if supply == 0:
            return self.max_multiplier

        ratio = demand / supply

        # Calculate surge multiplier
        if ratio <= 1.0:
            return self.base_multiplier

        # Linear increase
        multiplier = self.base_multiplier + (ratio - 1.0) * 0.5

        # Apply smoothing
        current_surge = await self.get_current_surge(cell_id)
        smoothed = current_surge * 0.7 + multiplier * 0.3

        # Clamp to max
        return min(smoothed, self.max_multiplier)

    async def get_price_estimate(self, pickup: dict, destination: dict,
                                  ride_type: str) -> dict:
        # Get route info
        route = await self.maps.get_directions(pickup, destination)
        distance_km = route['distance_meters'] / 1000
        duration_min = route['duration_seconds'] / 60

        # Base pricing
        pricing = self.get_base_pricing(ride_type)
        base_price = (
            pricing['base_fare'] +
            pricing['per_km'] * distance_km +
            pricing['per_min'] * duration_min
        )

        # Apply surge
        surge = await self.calculate_surge(pickup, ride_type)
        surged_price = base_price * surge

        # Apply min/max
        final_price = max(surged_price, pricing['minimum'])

        return {
            'min_price': int(final_price * 0.9),
            'max_price': int(final_price * 1.1),
            'surge_multiplier': surge,
            'currency': 'USD'
        }
```

### 5. Real-time Tracking

```python
class TrackingService:
    def __init__(self, websocket_manager, redis_pubsub):
        self.ws = websocket_manager
        self.pubsub = redis_pubsub

    async def start_tracking(self, ride_id: str, rider_id: str, driver_id: str):
        # Subscribe rider to driver location updates
        channel = f"ride:{ride_id}:location"

        await self.pubsub.subscribe(channel, lambda msg: self.ws.send_to_user(
            rider_id,
            {
                "type": "driver_location",
                "ride_id": ride_id,
                "location": msg['location'],
                "eta": msg['eta']
            }
        ))

    async def update_driver_location(self, driver_id: str, ride_id: str,
                                     lat: float, lng: float):
        # Calculate new ETA
        ride = await self.get_ride(ride_id)
        eta = await self.eta_service.calculate_eta(
            {'lat': lat, 'lng': lng},
            ride.destination if ride.status == 'in_progress' else ride.pickup
        )

        # Publish update
        await self.pubsub.publish(f"ride:{ride_id}:location", {
            "location": {"lat": lat, "lng": lng},
            "eta": eta,
            "timestamp": time.time()
        })

        # Store for history
        await self.store_location(ride_id, driver_id, lat, lng)
```

---

## Bottlenecks & Solutions

| Проблема | Решение |
|----------|---------|
| Location update volume | Batching, sampling, regional sharding |
| Matching latency | Pre-computed indexes, caching ETAs |
| Hot areas | Cell-based load balancing, regional scaling |
| Real-time tracking | WebSocket, pub/sub, CDN for static |
| Maps API costs | Caching, batch requests, own routing |

---

## Trade-offs

### Location Storage

| Approach | Accuracy | Latency | Scale |
|----------|----------|---------|-------|
| Redis Geo | High | Very Low | Medium |
| PostGIS | High | Medium | Medium |
| S2/H3 Cells | Good | Low | High |
| QuadTree | High | Low | Medium |

### Matching Strategy

| Strategy | Fairness | Speed | User Experience |
|----------|----------|-------|-----------------|
| Nearest driver | Low | Fast | Best ETA |
| Queue-based | High | Medium | Variable ETA |
| Auction | Medium | Slow | Best price |

---

## На интервью

### Ключевые моменты
1. **Geospatial indexing** — Redis Geo, S2/H3, QuadTree
2. **Real-time updates** — WebSocket, pub/sub, батчинг
3. **Matching algorithm** — баланс ETA, fairness, acceptance rate
4. **Surge pricing** — supply/demand, smoothing, regional cells

### Типичные follow-up
- Как масштабировать на новый город?
- Как обработать пиковую нагрузку (New Year's Eve)?
- Как реализовать ride sharing (несколько пассажиров)?
- Как оптимизировать маршрут для нескольких точек?
- Как обработать потерю связи у водителя?

---

[← Назад к списку тем](README.md)
