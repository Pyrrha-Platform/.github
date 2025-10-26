# Pyrrha Copilot Instructions

## Architecture Overview

Pyrrha is a distributed firefighter safety monitoring system built as a microservices architecture. The system processes real-time sensor data from IoT devices through multiple data transformation layers to provide safety alerts and analytics.

**Core Data Flow:**
1. Hardware sensors → Mobile app (Bluetooth) → MQTT broker → Database storage
2. Rules engine calculates time-weighted averages (TWA) every minute
3. WebSocket server broadcasts real-time data to dashboard
4. Dashboard displays current readings and historical averages

**Key Services:**
- `Pyrrha-Dashboard`: React/Carbon UI with Python Flask + Node.js APIs
- `Pyrrha-MQTT-Client`: Node.js service bridging IoT → Database → WebSocket
- `Pyrrha-Rules-Decision`: Python analytics service calculating exposure thresholds
- `Pyrrha-Database`: MariaDB with pre-seeded schema in `data-seed/pyrrha.sql`
- `Pyrrha-WebSocket-Server`: Real-time data broadcasting to dashboard

## Development Workflow

**Local Development Setup:**
```bash
# Start all services via Docker Compose
cd Pyrrha-Deployment-Configurations
docker-compose up

# Dashboard development (runs on :3000)
cd Pyrrha-Dashboard/pyrrha-dashboard
yarn install
yarn start-main-api  # Flask API on :5000
yarn start-auth-api  # Node.js auth on :4000  
yarn start-ui        # React dev server
```

**Critical Environment Files:**
- `Pyrrha-Dashboard/pyrrha-dashboard/.env` - Frontend config + Mapbox token
- `Pyrrha-Dashboard/pyrrha-dashboard/api-main/.env` - Database connection
- `Pyrrha-Dashboard/pyrrha-dashboard/api-auth/vcap-local.json` - IBM App ID credentials
- `Pyrrha-MQTT-Client/.env` - MQTT broker + database config

## Database Integration Patterns

**Connection Pattern:** All services use environment-based MariaDB configuration with connection pooling. See `firefighter_manager.py` for the standard pattern:

```python
conn = mariadb.connect(
    user=self._user, password=self._password,
    host=self._host, database=self._database, port=self._port
)
```

**Key Tables:**
- `sensor_data` - Raw IoT readings
- `averages_*` - Calculated TWA values (10min, 30min, 1hr, 4hr, 8hr)
- `firefighters` - Personnel records (uses `firefighter_code` as effective primary key)
- `vmq_auth_acl` - MQTT broker authentication

## API Architecture

**Multi-API Pattern:** Dashboard uses two separate backends:
- **api-main** (Flask/Python): CRUD operations, data queries, analytics
- **api-auth** (Node.js/Express): IBM App ID authentication, session management

**Proxy Configuration:** React dev server proxies API calls via `setupProxy.js`:
- `/api-main/v1/` → Flask service (:5000)
- `/api-auth/v1/` → Node.js service (:4000)

## Real-time Data Flow

**MQTT → WebSocket Pattern:**
1. MQTT Client receives sensor data from IoT platform
2. Stores in database + forwards to WebSocket Server
3. Dashboard connects to WebSocket for live updates
4. Real-time alerts trigger based on threshold comparisons

**WebSocket Endpoint:** Update `src/Utils/Constants.js` for WebSocket server URL

## Exposure Calculation Logic

**Time-Weighted Averages (TWA):** Rules-Decision service calculates occupational exposure limits:
- Short-term (10min, 30min): For immediate safety alerts
- Medium-term (1hr, 4hr): Regulatory compliance monitoring  
- Long-term (8hr): Full shift exposure tracking

**Gas Thresholds:** CO, NO2 limits based on OSHA/NIOSH standards. Configuration in Rules-Decision service.

## Deployment Considerations

**Docker Compose Stack:** All services orchestrated via single `docker-compose.yaml`. Critical for local development and testing integration points.

**Authentication Flow:** IBM App ID integration requires service credentials in `vcap-local.json`. Security is authentication-only (no authorization tiers currently).

**Database Dependencies:** Services start order matters - MariaDB must be ready before dependent services. Use health checks in production deployments.

## Code Style Conventions

**Python:** Black formatting via `yarn format:black` in Dashboard
**JavaScript/React:** Prettier + ESLint configuration in `package.json`
**Database:** Uses parameterized queries exclusively (see `firefighter_manager.py`)

## Testing & Debugging

**Service Health:** Check `docker compose ps` for container status
**Database Access:** Connect directly to MariaDB container for debugging queries
**Log Aggregation:** Each service logs independently - use `docker compose logs [service]`