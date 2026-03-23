# Designing a Ride-Sharing System (Uber/Lyft)

**Phase 7 — Tradeoffs + Interview Problems | Topic 8 of 8**

---

## Overview

This is a capstone-level question that combines nearly every concept from the system design curriculum: real-time location tracking, geospatial indexing, matching algorithms, distributed state management, pricing, and consistency under high load.

---

## Phase 1: Requirements

### Functional Requirements

```
Core flow:
  1. Rider requests a ride (origin, destination)
  2. System finds nearby available drivers
  3. System matches rider to best driver
  4. Driver accepts (or system auto-assigns)
  5. Real-time location tracking during ride
  6. Ride completes → fare calculated → payment charged

Questions to ask:
  "Is this standard rides only or also pool/carpool?"
  "Do drivers have to manually accept or is it auto-assigned?"
  "What's the matching criterion — closest driver or best ETA?"
  "Do we need surge pricing?"
  "Real-time tracking: how frequently does location update?"
  "Do we need trip history, ratings, payments?"

For this walkthrough:
  Standard rides only (no pooling)
  Driver manually accepts within 15s window
  Match by lowest ETA (not just proximity)
  Surge pricing: yes
  Location updates: every 5 seconds
  Trip history + ratings: yes, basic
  Payments: integration point only (not deep dive)
```

### Non-Functional Requirements

```
Scale:
  10M daily rides
  1M active drivers globally
  5M active riders globally

  Rides: 10M/day = 116 rides/sec
  Location updates: 1M drivers × 1 update/5s = 200K updates/sec

Latency:
  Match driver to rider: <5 seconds from request
  Location update processing: <1 second

Availability:
  99.99% — riders and drivers cannot be unable to use the app

Consistency:
  Driver can only be matched to ONE rider at a time (critical)
  No double-booking under any circumstances

Geography:
  Global: every city operates as mostly independent unit
  Data stays regional (EU data in EU)
```

---

## Phase 2: Capacity Estimation

```
Location updates:
  1M active drivers × 1 update/5s = 200K writes/sec
  Each update: driverId (8B) + lat (8B) + lng (8B) + timestamp (8B) = 32 bytes
  200K × 32 bytes = 6.4 MB/sec write bandwidth → manageable

Active driver index:
  Need to query "drivers within X km of point"
  1M active drivers × 32 bytes = 32 MB in memory
  Tiny — entire driver location table fits in one Redis instance

Ride records:
  10M rides/day × 500 bytes/record = 5 GB/day
  Retain 3 years: 5.5 TB
  → Sharded PostgreSQL by rideId or userId

Matching requests:
  116 ride requests/sec = 116 matching queries/sec
  Each query: geospatial search within 5km radius
  → Redis with geospatial index handles easily

Trip data (during ride):
  116 new rides/sec × ~600s average = ~70K concurrent rides
  70K riders + 70K drivers = 140K devices sending updates
  140K × 1 update/5s = 28K updates/sec
  → This is the hot path to optimize
```

---

## Phase 3: High-Level Design

### Key APIs

```
Rider API:
  POST /rides/request
    Body: {pickupLat, pickupLng, dropoffLat, dropoffLng, rideType}
    Response: {rideId, estimatedPrice, estimatedWait}

  GET /rides/{rideId}/status
    Response: {status, driver, location, eta}

  DELETE /rides/{rideId}  (cancel)

Driver API:
  PUT /drivers/location
    Body: {lat, lng, heading, speed}
    Response: {status: "updated"}

  PUT /drivers/availability
    Body: {available: true/false}

  POST /rides/{rideId}/accept
  POST /rides/{rideId}/complete
    Body: {finalLat, finalLng}

Internal:
  GET /drivers/nearby?lat=X&lng=Y&radius=5000
    Response: {drivers: [{driverId, lat, lng, eta, rating}]}
```

### System Architecture

```
Driver location pipeline:
[Driver App]
  → [Location Service] ← high-write optimized
  → [Redis Geo Index]  ← real-time driver positions
  → [Kafka: location-updates] ← for analytics/tracking

Ride request flow:
[Rider App]
  → [Ride Request Service]
       → [Pricing Service]        ← estimate + surge
       → [Matching Service]       ← find best driver
             → [Redis Geo Index]  ← nearby drivers
             → [ETA Service]      ← calculate ETAs
       → [Offer Service]          ← send offer to driver
       → [Trip Service]           ← manage ride lifecycle

During ride:
[Driver App] → [Location Service] → [Redis]
                                  → [Websocket Gateway] → [Rider App]

Payment (on completion):
[Trip Service] → [Fare Calculator] → [Payment Service]
```

---

## Phase 4: Deep Dives

### Deep Dive 1: Real-Time Driver Location

The highest write volume component: 200K location updates/sec.

```
Storage requirements:
  Must answer: "Which drivers are within 5km of this point?"
  Must be updated every 5 seconds
  Must handle 200K writes/sec

Redis GEOADD:
  GEOADD drivers:active {longitude} {latitude} {driverId}

  Internally: stores as sorted set with geohash score
  Supports: GEOSEARCH drivers:active FROMMEMBER/FROMLONLAT
            BYRADIUS 5 km ASC COUNT 20

  GEOSEARCH drivers:active
    FROMLONLAT -87.6298 41.8781  ← Chicago coordinates
    BYRADIUS 5 km
    ASC            ← sorted by distance
    COUNT 20       ← top 20 nearest
    WITHCOORD      ← include coordinates
    WITHDIST       ← include distance

At 200K updates/sec:
  Redis single instance: handles ~100K ops/sec
  Solution: geo-shard by city or region

  City-based sharding:
    drivers:chicago → Redis node 1
    drivers:nyc     → Redis node 2
    drivers:london  → Redis node 3

  Query always hits one city shard (rides are local)
  200K/sec across 50 major cities = 4K/sec per city → easy
```

```java
@Service
public class DriverLocationService {

    private final RedisTemplate<String, String> redis;

    public void updateLocation(String driverId, double lat, double lng, String city) {
        String key = "drivers:" + city;

        // Update Redis geo index
        redis.opsForGeo().add(key,
            new Point(lng, lat),  // GeoOperations uses (longitude, latitude)
            driverId);

        // Separate TTL key to mark driver as active
        // (can't set TTL on individual geo members directly)
        redis.opsForValue().set("driver:active:" + driverId, "1",
            Duration.ofSeconds(60));
    }

    public List<NearbyDriver> findNearbyDrivers(
            double lat, double lng, String city, double radiusKm) {

        String key = "drivers:" + city;

        GeoResults<GeoLocation<String>> results = redis.opsForGeo()
            .search(key,
                new GeoReference.GeoCoordinateReference<>(lng, lat),
                new Distance(radiusKm, Metrics.KILOMETERS),
                GeoSearchCommandArgs.newGeoSearchArgs()
                    .includeCoordinates()
                    .includeDistance()
                    .sortAscending()
                    .limit(20));

        return results.getContent().stream()
            .filter(r -> isDriverActive(r.getContent().getName()))
            .map(r -> new NearbyDriver(
                r.getContent().getName(),
                r.getContent().getPoint().getY(),   // lat
                r.getContent().getPoint().getX(),   // lng
                r.getDistance().getValue()))
            .collect(toList());
    }

    private boolean isDriverActive(String driverId) {
        return redis.hasKey("driver:active:" + driverId);
    }
}
```

**Alternative: Geohash for sharding:**
```
Geohash: encode lat/lng as a string prefix
  Geohash precision 4 = ~40km × 20km cell
  Geohash precision 5 = ~5km × 5km cell

  Driver at (41.8781, -87.6298) in Chicago → geohash = "dp3wjzp"

  Index by geohash cell:
  drivers:dp3w → all drivers in that geohash cell

  Query nearby: get 9 adjacent cells (3×3 grid)
  SUNIONSTORE nearby drivers:dp3w drivers:dp3x drivers:dp3q ...

  Used by: real production systems (Uber's approach)
           More complex but better for arbitrary precision
```

---

### Deep Dive 2: Matching Algorithm

```
Goal: minimize average ETA (not just proximity)
  Closest driver ≠ best ETA (traffic, road network matters)

Matching steps:

1. Find nearby available drivers (Redis GEOSEARCH, 5km radius)
   Returns: top 20 drivers by crow-fly distance

2. Calculate ETA for each candidate
   Crow-fly distance is approximate
   Real ETA requires road-network calculation

   Options:
   a) Google Maps / Mapbox Distance Matrix API
      Expensive at scale, external dependency
   b) Internal routing engine (OSRM, Valhalla)
      Self-hosted, faster, customizable
   c) Pre-computed ETA via geohash cells
      Each driver geohash → rider geohash → cached ETA
      Fast but less accurate

   At 116 requests/sec × 20 candidates = 2,320 ETA calls/sec
   Batch the Distance Matrix call (20 destinations at once)
   OSRM internally: ~10ms for 20-destination batch → acceptable

3. Score candidates
   Score = ETA to pickup (weighted)
           + driver rating (minor weight)
           + acceptance rate (prefer reliable drivers)

   Sort ascending → best match = lowest score

4. Offer to top candidate
   Sequential offers (not broadcast):
   Offer driver 1 (15s window)
   If declined/timeout → offer driver 2
   If declined/timeout → offer driver 3
   ...
   If all declined → expand radius → repeat

   Why sequential, not broadcast to all?
   Broadcast causes: driver 1 accepts, drivers 2-5 also try to accept
   → Race condition on ride assignment
   → Users see "driver found" then "reassigned"
   → Bad UX
```

---

### Deep Dive 3: Preventing Double-Booking (Critical)

```
Problem:
  Driver is AVAILABLE
  Rider A and Rider B both request at same millisecond
  Matching Service A: finds Driver X → assigns to Rider A
  Matching Service B: finds Driver X → assigns to Rider B

  Driver X assigned to two rides simultaneously → catastrophe

Solutions:

Option 1: Distributed Lock (Redis SETNX)
  Before assigning driver, acquire lock:
    SET driver:lock:{driverId} {rideId} NX EX 30
    NX: only set if not exists
    EX 30: auto-expire in 30s (in case lock never released)

  If returns OK: lock acquired → assignment proceeds
  If returns nil: another ride is being assigned → skip driver, try next

  ✅ Prevents double-booking
  ❌ Redis single point of failure (use Redis Sentinel or Cluster)
  ❌ Lock expiry must exceed max assignment time

Option 2: Database Optimistic Locking
  drivers table:
    driverId, status, currentRideId, version

  UPDATE drivers
  SET status='ON_TRIP', currentRideId='{rideId}', version=version+1
  WHERE driverId='{driverId}' AND status='AVAILABLE' AND version={expected}

  Rows affected = 1: success (version matched → no concurrent update)
  Rows affected = 0: concurrent update → retry with next driver

  ✅ ACID guarantee via DB transaction
  ❌ Extra DB round trip on every match attempt
  ❌ Higher latency than Redis lock

Option 3: Single-Writer per Driver (Partition by driverId)
  All assignment decisions for a given driver → same instance
  Kafka partition key = driverId
  One consumer handles all operations for driver X
  No concurrent assignment possible (single thread)

  ✅ No locks needed (serial processing per driver)
  ✅ Natural ordering
  ❌ Complex routing
  ❌ Hotspot if popular driver ID handles many assignments

Production recommendation:
  Redis distributed lock (primary path, fast)
  + Database optimistic lock (safety net, confirmation)
  Both before confirming assignment to driver/rider
```

```java
@Service
public class DriverAssignmentService {

    @Transactional
    public boolean tryAssignDriver(String driverId, String rideId) {

        // Step 1: Acquire distributed lock
        String lockKey = "driver:lock:" + driverId;
        Boolean locked = redis.opsForValue()
            .setIfAbsent(lockKey, rideId, Duration.ofSeconds(30));

        if (!Boolean.TRUE.equals(locked)) {
            return false; // driver being assigned elsewhere
        }

        try {
            // Step 2: DB optimistic lock (confirm driver still available)
            int updated = driverRepo.assignIfAvailable(driverId, rideId);

            if (updated == 0) {
                // Driver already assigned (race condition caught)
                redis.delete(lockKey);
                return false;
            }

            // Step 3: Update ride record
            rideRepo.assignDriver(rideId, driverId);

            // Step 4: Notify driver via WebSocket/push
            notifyDriver(driverId, rideId);

            return true;

        } catch (Exception e) {
            redis.delete(lockKey); // release lock on failure
            throw e;
        }
    }
}

// Driver table - DB side
@Modifying
@Query("""
    UPDATE Driver d
    SET d.status = 'OFFERED', d.currentRideId = :rideId, d.version = d.version + 1
    WHERE d.id = :driverId
    AND d.status = 'AVAILABLE'
    AND d.version = :version
    """)
int assignIfAvailable(@Param("driverId") String driverId,
                       @Param("rideId") String rideId,
                       @Param("version") long version);
```

---

### Deep Dive 4: Surge Pricing

```
Supply/demand imbalance → increase price to:
  1. Attract more drivers to area
  2. Reduce demand (price-sensitive riders wait)
  3. Ensure drivers get sufficient compensation

Calculation:
  Surge multiplier = f(demand / supply)

  Supply: available drivers in area (from Redis geo index)
  Demand: ride requests per minute in area (from Kafka stream)

  Simple surge formula:
  demand_rate = ride requests in last 5 minutes in geohash cell
  supply_count = available drivers in same cell

  demand_to_supply = demand_rate / supply_count

  if d/s < 0.5: multiplier = 1.0 (no surge)
  if d/s < 1.0: multiplier = 1.2
  if d/s < 1.5: multiplier = 1.5
  if d/s < 2.0: multiplier = 2.0
  else:         multiplier = 2.5 (cap to avoid PR disaster)

Architecture:
  Kafka stream of ride requests
  → Flink window: count requests per geohash per 5-minute window
  → Compare to driver count from Redis
  → Write surge multiplier to Redis:
      SET surge:{geohash} 1.5 EX 300  ← recomputed every 5 minutes

  Rider request → lookup surge for pickup geohash → apply to base price

User experience:
  Show surge multiplier to rider before confirming
  "Prices are 1.5x due to high demand. Confirm?"
  Require explicit confirmation for surge > 1.5x
  Timer: multiplier may change, lock price for 2 minutes
```

---

### Deep Dive 5: Real-Time Tracking During Ride

```
During ride:
  Driver app → sends GPS every 5 seconds
  Rider app → shows live driver position on map

Location update flow:
  Driver → WebSocket → Location Service → Redis
                                        → Kafka: location-updates

  Rider's WebSocket connection:
  Subscribes to updates for their rideId
  Location Service pushes update: "driver X is now at lat/lng"

  Rider map updates in real-time

Implementation:
  Location Service maintains:
    - WebSocket connections to all active drivers
    - WebSocket connections to all active riders
    - Mapping: rideId → (driverId, riderId)

  On driver location update:
    Update Redis: SET driver:location:{driverId} "{lat},{lng}" EX 30
    Look up rider for this driver: rideId → riderId
    Push to rider's WebSocket: {lat, lng, heading, eta}

Scale:
  70K concurrent rides × 2 WebSocket connections = 140K connections
  Per WebSocket server: handle ~50K connections (50KB RAM each = 2.5GB)
  3 servers needed for 140K connections → very manageable

Path recording for fare calculation:
  All location updates for active rides → Kafka
  Trip path service consumes → builds polyline
  On trip end: calculate actual distance from polyline
  Base fare + per-mile × actual miles + per-minute × duration
  Apply surge multiplier from ride start time
```

---

### Deep Dive 6: Trip State Machine

```
Ride progresses through states:

REQUESTED → MATCHING → DRIVER_ASSIGNED → DRIVER_ARRIVING
→ DRIVER_ARRIVED → IN_PROGRESS → COMPLETED / CANCELLED

State transitions:
  REQUESTED:       rider submits request
  MATCHING:        system searching for driver (timeout: 3 minutes)
  DRIVER_ASSIGNED: driver accepted offer
  DRIVER_ARRIVING: driver en route to pickup
  DRIVER_ARRIVED:  driver at pickup location (app detects via geofence)
  IN_PROGRESS:     rider in car, en route to destination
  COMPLETED:       driver marks trip complete
  CANCELLED:       rider or driver cancels

Storage:
  rides table:
    rideId, riderId, driverId, status,
    pickupLat, pickupLng, dropoffLat, dropoffLng,
    requestedAt, matchedAt, pickedUpAt, completedAt,
    baseFare, surgeMult, totalFare,
    driverRating, riderRating

  Shard by rideId (UUIDv4, random distribution)
  Hot path: current rides → Redis cache (small, fast)
  Cold path: historical rides → sharded PostgreSQL

State transitions via Kafka:
  Each transition → event published
  DRIVER_ARRIVED event → start timer for rider (2 min free wait)
  COMPLETED event → trigger fare calculation → payment
  CANCELLED event → determine cancellation fee
    (driver arrived → charge rider)
```

---

### Deep Dive 7: Regional Architecture

```
Rides are local (not cross-city)
Different cities → different loads, regulations, pricing

Architecture:
  Each major city/region has dedicated infrastructure:

  [Chicago Region]
    Redis: drivers:chicago
    Matching Service instances
    Trip DB shard
    WebSocket gateway

  [NYC Region]
    Redis: drivers:nyc
    Matching Service instances
    Trip DB shard
    WebSocket gateway

  [Global Control Plane]
    User accounts DB (global)
    Payment Service (global)
    Analytics pipeline (global, reads from regional Kafka)

User routing:
  Mobile app → nearest regional endpoint
  Driver in Chicago → Chicago matching service
  Rider in Chicago → Chicago matching service
  No cross-region coordination needed for a ride

Inter-region: only for user account operations
  Driver moves from Chicago to NYC:
    App detects city change
    Re-registers with NYC matching service
    Driver location moves to drivers:nyc index
```

---

## Phase 5: Tradeoffs

```
1. Sequential driver offers vs broadcast:
   Sequential: simpler state, no double-accept race conditions
   Cost: slightly longer wait if first drivers decline
   Mitigation: 15s window per driver, expand radius after 3 declines

2. Redis for driver location vs in-memory service:
   Redis: persistent, survives crashes, shared across instances
   In-memory: faster but no persistence, can't share across servers
   Choice: Redis (durability + sharing essential)

3. ETA via external API vs internal routing:
   External (Google Maps): accurate, high cost at 2320 calls/sec
   Internal (OSRM): less accurate but controllable cost
   Choice: internal OSRM with Google as fallback

4. Single-region vs global driver index:
   Global: can match across cities (not needed for rides)
   Regional: faster, simpler, legal compliance
   Choice: regional (rides are always local)

5. WebSocket vs polling for live tracking:
   WebSocket: real-time, server push, complex to scale
   Polling: simpler but delayed updates, more DB load
   Choice: WebSocket (rider map must be real-time, not 5s stale)
```

---

## End-to-End Flow Summary

```
1. Driver app sends location every 5s
   → Location Service → Redis GEOADD drivers:chicago
   → Driver active TTL refreshed

2. Rider requests ride (pickup, dropoff)
   → Ride Request Service creates ride (status: MATCHING)
   → Pricing Service: base fare + surge multiplier
   → Matching Service: GEOSEARCH nearby drivers (5km)
   → ETA Service: calculate ETAs for top 20 candidates
   → Score and rank candidates

3. Offer to best driver
   → WebSocket push to driver app: "New ride request"
   → 15-second accept window
   → Driver accepts:
       Redis lock + DB optimistic lock → assign driver
       Ride status: DRIVER_ASSIGNED
   → Driver declines/timeout: offer next candidate

4. During ride
   → Driver location updates → WebSocket → rider map
   → Trip path recorded in Kafka → stored for fare calculation

5. Ride completes
   → Driver marks complete
   → Fare calculator: base + surge + distance + time
   → Payment Service charges rider
   → Ratings prompted on both apps
   → Driver released back to available pool
```

---

## Key Takeaways

```
Ride-sharing combines almost every distributed systems concept:

Real-time location:
  200K updates/sec → Redis GEOADD with city-based sharding
  GEOSEARCH by radius → top 20 nearest drivers instantly
  Active driver TTL (60s) for stale driver cleanup

Matching:
  Not just proximity → ETA-based scoring
  Batch ETA calculation (OSRM, 20 destinations per call)
  Sequential offers (not broadcast) → avoid race conditions

Double-booking prevention:
  Redis SETNX distributed lock (fast path)
  DB optimistic locking (safety net)
  Both required together for correctness

Surge pricing:
  Flink stream: demand/supply ratio per geohash cell
  Surge multiplier stored in Redis (5-min TTL)
  Cap multiplier + explicit user confirmation

Real-time tracking:
  WebSocket connections: 140K concurrent during rides
  Location push: driver update → rider map
  Path recording via Kafka for fare calculation

State machine:
  REQUESTED → MATCHING → ASSIGNED → ARRIVING →
  IN_PROGRESS → COMPLETED
  Each transition = Kafka event + DB update

Regional architecture:
  Rides are local → per-city Redis, matching service, DB
  Global: user accounts, payments, analytics
  No cross-region coordination for matching
```
