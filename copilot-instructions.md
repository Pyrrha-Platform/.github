# Pyrrha Copilot Instructions

## Architecture Overview

Pyrrha is a distributed firefighter safety monitoring system built as a microservices architecture. The system processes real-time sensor data from IoT devices through multiple data transformation layers to provide safety alerts and analytics.

**Core Data Flow:**
1. **Prometeo devices** → **Device Simulator** → **VerneMQ MQTT broker** → **Database storage**
2. **MQTT Client** processes messages and stores in MariaDB with **device ID parsing**
3. **Rules Decision engine** calculates time-weighted averages (TWA) every minute
4. **WebSocket server** broadcasts real-time data to dashboard
5. **Dashboard** displays current readings and historical averages

**Key Services (9 containers total):**
- `Pyrrha-Dashboard`: React/Carbon UI with Python Flask + Node.js APIs
- `Pyrrha-Device-Simulator`: Node.js service simulating 4 IoT Prometeo devices
- `Pyrrha-MQTT-Client`: Node.js service bridging MQTT → Database → WebSocket
- `Pyrrha-Rules-Decision`: Python analytics service calculating exposure thresholds
- `Pyrrha-Database`: MariaDB 12.0-ubi with pre-seeded schema and MQTT auth
- `Pyrrha-WebSocket-Server`: Real-time data broadcasting to dashboard
- **VerneMQ MQTT Broker**: v2.1.1-alpine with MySQL authentication plugin
- **API Services**: Separate Flask (main) and Node.js (auth) backends

## Development Workflow

**Production-Ready Local Development Setup:**
```bash
# Complete system startup (recommended)
cd Pyrrha-Deployment-Configurations
docker compose up -d  # Starts all 9 services

# Clean rebuild (after changes)
docker compose down -v  # Removes volumes for fresh database
docker compose up -d

# Development with hot reload (dashboard only)
cd Pyrrha-Dashboard/pyrrha-dashboard
yarn install
yarn start-main-api  # Flask API on :5000
yarn start-auth-api  # Node.js auth on :4000  
yarn start-ui        # React dev server on :3000
```

**Critical Environment Files:**
- `Pyrrha-Dashboard/pyrrha-dashboard/.env` - Frontend config + Mapbox token
- `Pyrrha-Dashboard/pyrrha-dashboard/api-main/.env` - Database connection
- `Pyrrha-Dashboard/pyrrha-dashboard/api-auth/vcap-local.json` - IBM App ID credentials
- `Pyrrha-MQTT-Client/.env` - MQTT broker + database config
- `Pyrrha-Device-Simulator/devices.json` - Device credentials (standardized)

**Key Service URLs:**
- Dashboard: http://localhost:3000
- Flask API: http://localhost:5000
- Auth API: http://localhost:4000
- MQTT Broker: localhost:1883
- WebSocket: ws://localhost:8080
- MariaDB: localhost:3306

## Database Integration Patterns

**Connection Pattern:** All services use environment-based MariaDB configuration with connection pooling. See `firefighter_manager.py` for the standard pattern:

```python
conn = mariadb.connect(
    user=self._user, password=self._password,
    host=self._host, database=self._database, port=self._port
)
```

**Key Tables:**
- `firefighter_device_log` - Real-time sensor data with parsed device IDs (int)
- `averages_*` - Calculated TWA values (10min, 30min, 1hr, 4hr, 8hr)
- `firefighters` - Personnel records (uses `firefighter_code` as effective primary key)
- `devices` - Device registry (4 Prometeo devices with coordinates)
- `vmq_auth_acl` - **CRITICAL**: MQTT broker authentication with SHA2 passwords
  - 4 Prometeo device entries with publish permissions
  - 1 MQTT Client entry with publish + subscribe permissions

## API Architecture

**Multi-API Pattern:** Dashboard uses two separate backends:
- **api-main** (Flask/Python): CRUD operations, data queries, analytics
- **api-auth** (Node.js/Express): IBM App ID authentication, session management

**Proxy Configuration:** React dev server proxies API calls via `setupProxy.js`:
- `/api-main/v1/` → Flask service (:5000)
- `/api-auth/v1/` → Node.js service (:4000)

## Real-time Data Flow

**MQTT → WebSocket Pattern:**
1. Device Simulator publishes sensor data every minute to VerneMQ broker
2. MQTT Client subscribes to iot-2/# topic and receives all device messages
3. **Device ID Parsing**: Extracts numeric ID from "Prometeo:00:00:00:00:00:0X" → X
4. Stores parsed data in `firefighter_device_log` table
5. Forwards processed data to WebSocket Server for real-time broadcasting
6. Dashboard connects to WebSocket for live updates and alerts

**Authentication Requirements:**
- VerneMQ uses MySQL authentication plugin with `vmq_auth_acl` table
- Device simulator requires publish permissions for each Prometeo device
- MQTT Client requires **both** publish AND subscribe permissions to receive messages
- All passwords hashed using SHA2('password', 256) function

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

## MQTT Authentication System

**Critical Setup Requirements:**
- VerneMQ MQTT broker requires MySQL authentication plugin configuration
- Database seed file (`pyrrha.sql`) contains pre-configured authentication entries
- All device passwords use SHA2() hashing: `SHA2('password', 256)`

**Authentication Entries in vmq_auth_acl:**
- 4 Prometeo devices (client_id: "Prometeo:00:00:00:00:00:01" through "04")
  - Each has publish_acl with iot-2/# pattern
  - subscribe_acl: NULL (devices only publish)
- 1 MQTT Client entry (client_id: "someclientid", username: "username")
  - publish_acl with iot-2/# pattern
  - subscribe_acl with iot-2/# pattern (required to receive messages)

**Troubleshooting MQTT Issues:**
1. Check authentication table: `SELECT * FROM vmq_auth_acl;`
2. Verify MQTT Client has both publish AND subscribe permissions
3. Ensure device simulator credentials match database entries
4. Check VerneMQ logs for authentication failures

## Device ID Parsing Logic

**Data Transformation:** MQTT Client converts full device identifiers to numeric IDs:
- Input: "Prometeo:00:00:00:00:00:01" → Output: 1
- Input: "Prometeo:00:00:00:00:00:04" → Output: 4
- Database stores parsed numeric device_id as int(11) in firefighter_device_log

**Implementation:** See MQTT Client parsing logic that extracts the final digit from device strings.

## Repository Structure Updates

**Recent Changes:**
- Renamed "Pyrrha-Sensor-Simulator" → "Pyrrha-Device-Simulator"
- All documentation and configuration files updated to reflect new naming
- Docker Compose services use consistent naming convention

## Code Style Conventions

**Python:** Black formatting via `yarn format:black` in Dashboard
**JavaScript/React:** Prettier + ESLint configuration in `package.json`
**Database:** Uses parameterized queries exclusively (see `firefighter_manager.py`)

## Testing & Debugging

**Service Health:** Check `docker compose ps` for container status
**Database Access:** Connect directly to MariaDB container for debugging queries
**Log Aggregation:** Each service logs independently - use `docker compose logs [service]`

**End-to-End Verification:**
1. Check device simulator is publishing: `docker compose logs pyrrha-simulator`
2. Verify MQTT client connectivity: `docker compose logs pyrrha-mqttclient`
3. Confirm database entries: `SELECT COUNT(*) FROM firefighter_device_log WHERE timestamp_mins >= NOW() - INTERVAL 5 MINUTE;`
4. Test WebSocket broadcasting: `docker compose logs pyrrha-wss`

**Clean Environment Setup:**
```bash
# Complete reset and rebuild
docker compose down -v    # Removes all volumes and data
docker system prune -f    # Cleans up Docker resources
docker compose up -d      # Fresh rebuild with seed data

# System should be operational within 30 seconds
# Database will auto-populate with authentication entries from pyrrha.sql
```