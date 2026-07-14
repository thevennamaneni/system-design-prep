# Day 26 - Full System Design: Uber (Ride-Sharing)

## Topic
**Design Uber/Lyft** — a location-based, real-time ride-sharing system. Covers geospatial indexing, real-time tracking, dispatching, and the map-reduce of spatial data.

---

## Why This Problem Matters
- It's a top-5 FAANG system design question (Uber, Lyft, DoorDash, Instacart all ask this).
- It tests **geospatial data** handling — a unique challenge not covered by other problems.
- It combines real-time tracking (WebSockets), spatial queries (nearby drivers), and dispatch algorithms.

---

## RESHADED Framework

### R — Requirements (5 min)

**Functional:**
1. Riders request a ride (pickup, dropoff location).
2. System finds nearby available drivers.
3. Driver accepts/rejects the ride.
4. Real-time tracking: rider sees driver approaching; driver sees pickup location.
5. Fare calculation (distance, time, surge).
6. Trip history and receipts.

**Non-functional:**
1. Low latency: < 3 seconds to match rider with driver.
2. Real-time updates: driver location refreshes every 3-5 seconds.
3. High availability: 99.99% (people rely on this for transportation).
4. 10M riders, 1M drivers, 1M trips/day.

**Clarifying questions:**
- "Pool/Shared rides?" → No, focus on single-rider rides.
- "Dynamic pricing (surge)?" → Yes, mention the concept.
- "Multiple vehicle types?" → Yes (UberX, UberXL, Black).
- "Global?" → Yes, multi-region.

### E — Estimation (3 min)

```
Riders: 10M MAU
Drivers: 1M (100K active at any time)
Trips/day: 1M
Trips/sec (avg): 1M / 86400 ≈ 12/sec
Trips/sec (peak): ~50/sec

Driver location updates:
  100K active drivers × 1 update / 3 sec = ~33K updates/sec

Rider location updates (during ride):
  ~500K riders in transit × 1 update / 3 sec = ~167K updates/sec

Total location updates: ~200K/sec (high write QPS)

Storage:
  Trip record: ~1KB (pickup, dropoff, fare, timestamps)
  Daily: 1M × 1KB = 1GB/day
  5 years: 1.8TB (small — trips are not the storage challenge)
  
  Location data: transient (in Redis, not persisted long-term)
```

### S — Storage Schema (5 min)

```
PostgreSQL: users, trips (relational, ACID)
  users (id, name, phone, email, role [rider/driver], rating)
  trips (id, rider_id, driver_id, pickup_lat, pickup_lng, dropoff_lat, dropoff_lng,
         status, fare, distance, created_at, completed_at)

Redis: real-time location + nearby search (GEOREDISS)
  driver:location:{driver_id} → [lat, lng] (TTL 30 seconds)
  driver:status:{driver_id} → "available" / "busy" / "offline"
  
  Geospatial index: Redis GEOADD
    Key: "available_drivers:{city}"
    Members: driver_ids with coordinates
    Query: GEORADIUS to find drivers within X km

Cassandra: trip history (high write volume, time-series)
  trips_by_user (user_id, trip_id, ...)

S3 + CDN: maps (map tiles served by Mapbox/Google Maps API)
```

### H — High-Level Design (10 min)

```
┌────────┐         ┌──────────┐
│ Rider   │────────→│ API       │
│ App     │←───────│ Gateway   │
└────┬───┘         └─────┬────┘
     │                   │
     │ WebSocket          │ REST
     ▼                   ▼
┌──────────┐      ┌───────────┐     ┌───────────┐
│WebSocket  │      │  Ride      │────→│  Dispatch │
│Server     │      │  Service   │     │  Service  │
│(location  │      │(request,   │     │(match     │
│ updates)  │      │ complete)  │     │ rider↔driver)│
└─────┬────┘      └─────┬─────┘     └─────┬─────┘
      │                  │                 │
      ▼                  ▼                 ▼
┌──────────┐      ┌───────────┐     ┌───────────┐
│  Redis   │      │PostgreSQL │     │  Redis    │
│(locations│      │(trips,    │     │(available │
│ presence)│      │ users)    │     │ drivers)  │
└──────────┘      └───────────┘     └───────────┘
```

### A — API Design (5 min)

```
=== REST API ===

POST /api/v1/rides
  Body: { "pickup": {"lat": 37.7, "lng": -122.4}, "dropoff": {"lat": 37.8, "lng": -122.5}, "ride_type": "uber_x" }
  Response: 201 { "ride_id": "r123", "status": "searching", "estimated_fare": 15.50, "eta": 300 }

POST /api/v1/rides/{ride_id}/accept    (driver accepts)
  Body: { "driver_id": "d456" }
  Response: 200 { "status": "driver_assigned", "rider": {...}, "pickup": {...} }

POST /api/v1/rides/{ride_id}/cancel
  Response: 200 { "status": "cancelled", "cancellation_fee": 0 }

POST /api/v1/rides/{ride_id}/complete
  Body: { "actual_distance": 5.2, "duration_sec": 900 }
  Response: 200 { "status": "completed", "fare": 16.20 }

GET /api/v1/rides/{ride_id}
  Response: 200 { "ride": {...}, "driver": {...}, "status": "ongoing" }

GET /api/v1/users/{user_id}/rides?cursor=abc  (trip history)

=== WebSocket Events ===

Driver → Server:
  { "type": "location_update", "lat": 37.7, "lng": -122.4, "heading": 180 }
  { "type": "status_update", "status": "available" }

Server → Driver:
  { "type": "ride_request", "ride_id": "r123", "pickup": {...}, "dropoff": {...}, "fare": 15.50 }
  { "type": "rider_location", "lat": ..., "lng": ... }

Server → Rider:
  { "type": "driver_assigned", "driver": {...}, "eta": 180 }
  { "type": "driver_location", "lat": ..., "lng": ..., "heading": ... }
  { "type": "driver_arrived" }
  { "type": "ride_started" }
  { "type": "ride_completed", "fare": 16.20 }
```

### D — Detailed Design (15 min)

#### Geospatial Indexing (THE core problem)

**The challenge:** "Find all available drivers within 3 km of the rider's pickup location, in < 1 second."

**Option 1: Redis GEO (Simplest)**
```
Redis has built-in geospatial commands:
  GEOADD available_drivers:SF -122.4 37.7 driver_123
  GEOADD available_drivers:SF -122.5 37.8 driver_456

  GEORADIUS available_drivers:SF -122.4 37.7 3 km ASC COUNT 10
  → Returns 10 nearest drivers within 3 km, sorted by distance.
```
**Pros:** Simple, fast, built-in. Good for up to ~100K drivers per city.
**Cons:** Limited to one Redis node per city (sharding by city is natural).

**Option 2: Geohash**
```
Geohash: encode lat/lng as a string that represents a grid cell.
  "dr5ru" = a cell in NYC (roughly 5km × 5km)

To find nearby drivers:
  1. Compute the geohash of the rider's location.
  2. Query the DB for drivers with the same geohash prefix.
  3. Also check neighboring cells (8 adjacent cells).
  4. Filter by actual distance (haversine formula).

Longer prefix = smaller cell = more precise but fewer drivers per cell.
"dr5ru" → ~5km cell
"dr5ruj" → ~600m cell
```

**Option 3: PostGIS (PostgreSQL extension)**
```sql
SELECT * FROM drivers 
WHERE status = 'available' 
  AND ST_DWithin(location, ST_Point(-122.4, 37.7)::geography, 3000)  -- within 3km
ORDER BY ST_Distance(location, ST_Point(-122.4, 37.7))
LIMIT 10;
```
**Pros:** Precise, SQL queries, supports complex spatial operations.
**Cons:** PostgreSQL can't handle 200K writes/sec (driver location updates). Use Redis for real-time, PostGIS for historical queries.

**Recommended for interview:** Redis GEO for real-time nearby search. It's purpose-built, fast, and simple.

#### Dispatch Algorithm
```
1. Rider requests ride → API sends to Dispatch Service.
2. Dispatch Service:
   a. GEORADIUS available_drivers:{city} {rider_lat} {rider_lng} 3 km ASC COUNT 10
      → Gets 10 nearest available drivers.
   b. Send ride request to each driver via WebSocket (first to accept wins).
   c. Set a timeout (5 seconds):
      - If a driver accepts → assign ride → notify rider.
      - If no one accepts → expand radius to 5 km → retry.
      - If still no one → "no drivers available" → notify rider.
3. Driver accepts → ride assigned → both parties get each other's info.
```

#### Real-Time Location Tracking
```
Driver app (every 3 seconds):
  1. Get GPS coordinates.
  2. Send via WebSocket: { type: "location_update", lat, lng, heading }
  3. WS Server:
     a. Update Redis: GEOADD driver_locations {city} lng lat {driver_id}
     b. If on a ride: publish to Redis Pub/Sub "rider:{rider_id}"
     c. Rider's WS Server receives → pushes to rider's app.

Rider sees driver moving on the map in real-time (3-second refresh).
```

#### Surge Pricing
```
Surge multiplier = f(supply, demand)

For a given area (geohash cell):
  Supply = count of available drivers
  Demand = count of active ride requests

If Demand/Supply > threshold (e.g., 2.0):
  Surge multiplier = min(Demand/Supply, max_surge)  // e.g., max 3.0x

Surge updates every 5 minutes per area.
Stored in Redis: surge:{geohash} → 1.5 (1.5x normal price)
```

#### Trip Completion & Fare
```
1. Driver marks ride as complete.
2. Ride Service:
   a. Calculate actual fare: base_fare + (distance × per_km) + (duration × per_min) × surge_multiplier
   b. Process payment (via Payment Service).
   c. Store trip in PostgreSQL + Cassandra (history).
   d. Update driver status to "available".
   e. Send receipt to rider (email/push).
   f. Update rider/driver ratings.
```

### E — Edge Cases (5 min)
- **No drivers available:** Expand radius. If still none, notify rider with estimated wait time.
- **Driver cancels after accepting:** Re-dispatch to next nearest driver. Penalize driver (lower rating).
- **Rider cancels:** If < 2 min after request → no fee. If driver already en route → cancellation fee.
- **GPS inaccuracy:** Use driver's heading + speed to interpolate between updates.
- **Network disconnection:** App uses last known location. On reconnect, sync missed updates.
- **Multiple riders requesting simultaneously:** Redis GEO queries are independent; no conflict.
- **Driver in multiple cities:** Shard by city; driver can only be in one city's index.

### D — Discussion (5 min)
- **Bottleneck:** Location update writes (200K/sec). Redis handles this well (single-threaded, in-memory).
- **Scaling Redis:** Shard by city (`available_drivers:SF`, `available_drivers:NYC`). Each city on a separate Redis node.
- **Alternative to Redis GEO:** Quadtree (used by Google Maps), R-tree, or PostGIS. Redis GEO is simplest for interviews.
- **Map rendering:** Use Mapbox/Google Maps SDK — don't build your own map.
- **Monitoring:** Driver match time (p99 < 3s), location update latency, WebSocket connection count.

---

## Exercise: Design Food Delivery (DoorDash/UberEats)

Adapt the ride-sharing design for food delivery:
1. Customer orders from a restaurant.
2. Driver picks up food from restaurant → delivers to customer.
3. Three parties: customer, restaurant, driver.

**Think about:**
- What changes from the Uber design?
- New entity: restaurant. New states: preparing, ready, picked up, delivered.
- How to estimate delivery time (restaurant prep + drive time)?

<details>
<summary>Key Differences</summary>

1. **Three-party system:** Customer, restaurant, driver (not just rider + driver).
2. **Order lifecycle:** Placed → Restaurant confirms → Preparing → Ready → Driver assigned → Picked up → En route → Delivered.
3. **Driver dispatch timing:** Don't assign driver immediately — assign when food is almost ready (minimize driver waiting at restaurant).
4. **ETA estimation:** Restaurant prep time (historical average) + drive time (routing API). More complex than Uber (where pickup is instant).
5. **Restaurant search:** "Restaurants near me" — geospatial query on restaurants, not drivers. Use PostGIS or Elasticsearch with geo filter.
6. **Menu management:** Restaurant updates menu/prices → must propagate to all customers (cache invalidation).
7. **Payment:** More complex — split between restaurant, driver, and platform (commission).
</details>

---

## Common Mistakes
- Not knowing how to do geospatial queries (Redis GEO, geohash, PostGIS).
- Trying to use PostgreSQL for real-time location updates (can't handle the write QPS).
- Not handling the "no drivers available" case.
- Forgetting that driver locations change every 3 seconds (high write QPS).
- Not discussing surge pricing (shows understanding of the business).

## Checklist Before Moving On
- [ ] Can explain 3 geospatial indexing approaches (Redis GEO, geohash, PostGIS)
- [ ] Understand the dispatch flow (find nearby → send requests → first accept wins)
- [ ] Can design real-time location tracking via WebSocket + Redis
- [ ] Know how to handle the "no drivers" edge case (expand radius, notify)
- [ ] Understand surge pricing (supply vs demand per area)
- [ ] Solved the Uber design using RESHADED
