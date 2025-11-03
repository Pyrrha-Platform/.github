# Pyrrha Copilot Instructions

## Architecture Overview

Pyrrha is a distributed firefighter safety monitoring system built as a microservices architecture with **enterprise-grade code quality standards** implemented across all repositories. The system processes real-time sensor data from IoT devices through multiple data transformation layers to provide safety alerts and analytics.

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

**Mobile & Watch Applications:**
- `Pyrrha-Mobile-App`: Android 14 (API 34) app with Samsung Accessory Protocol provider
- `Pyrrha-Watch-App`: Tizen 5.5 Galaxy Watch 3 app with Samsung Accessory Protocol consumer
- **Samsung Integration**: Real-time sensor data transmission mobile → watch via Bluetooth/WiFi
- **BLE Integration**: Mobile app connects to Prometeo devices and forwards data to watch
- **Modernization**: Both apps updated for Samsung Galaxy A51 + Galaxy Watch 3 platform

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

## Code Quality Improvements & Metrics

**Dashboard Quality Transformation:**
- **Before:** 1,622 linting problems (1,496 errors, 126 warnings)  
- **After:** 363 problems (21 errors, 342 warnings)
- **Achievement:** 78% reduction in code quality issues
- **Key Fixes:** Prettier formatting applied to 57 files, ESLint configuration enhanced for mixed environments

**Rules-Decision Formatting Success:**
- **Black Reformatting:** 5 Python files successfully standardized
- **Configuration:** Line-length 127 applied consistently across all Python code
- **Import Sorting:** isort applied with matching Black configuration
- **Security Integration:** bandit + safety scanning automated in CI/CD

**JavaScript/React Standardization:**
- **Prettier Configuration:** Centralized .prettierrc.js with proven Dashboard settings
  - singleQuote: true, bracketSameLine: true (updated from deprecated jsxBracketSameLine)
  - Applied across all JavaScript repositories (MQTT-Client, WebSocket-Server, Website, Device-Simulator)
- **ESLint Enhancement:** Updated for mixed Node.js/browser environments with proper global handling

**C/C++ Arduino Quality:**
- **clang-format Integration:** Google style with 2-space indent, 100-character limit
- **Arduino Validation:** Automatic detection and validation of setup()/loop() functions
- **Performance Analysis:** Code statistics and large file warnings for maintainability

**Cross-Repository Benefits:**
- **Dependency Caching:** npm/yarn caching implemented across all Node.js workflows
- **Security Automation:** Python repositories automatically generate bandit + safety reports
- **Configuration Inheritance:** All repositories can reference centralized Pyrrha-Development-Tools configs
- **Graceful Degradation:** Workflows handle missing configurations without failing builds

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

## GitHub Actions Workflow Standards

**Standardized CI/CD Patterns:** All repositories now follow consistent GitHub Actions workflows with technology-appropriate templates:

**Dashboard Repository (`github-workflow-dashboard.yml`):**
- Multi-stack linting: React (ESLint + Prettier) + Flask (Black) + Node.js Auth
- Test execution with coverage reporting
- TypeScript validation (when applicable)
- Dockerfile linting for all containerized components

**Python Repositories (`github-workflow-python.yml`):**
- Black formatting with line-length 127 (proven Rules-Decision configuration)
- Import sorting (isort), linting (flake8), type checking (mypy)
- Security scanning: bandit (static analysis) + safety (dependency vulnerabilities)
- Automated artifact upload for security reports

**Node.js Services (`github-workflow-nodejs.yml`):**
- Centralized Prettier configuration from Development Tools repository
- ESLint validation (when configured)
- npm test execution with dependency caching
- Dockerfile linting for containerized services

**C/C++ Arduino (`github-workflow-cpp.yml`):**
- clang-format with Google style guidelines
- Arduino project structure validation (setup/loop functions)
- Code quality analysis with file size warnings
- Performance optimization suggestions

**Infrastructure (`github-workflow-infrastructure.yml`):**
- Dockerfile linting with dockerfilelint
- Documentation formatting (Markdown, YAML, JSON)
- SQL file validation with syntax checking
- Infrastructure-specific quality gates

**Consistent Features Across All Workflows:**
- **Triggers:** Push to all branches + PR to main branch
- **Node.js Version:** Standardized on Node.js 20 with dependency caching
- **Python Version:** Python 3.11 for all Python-based workflows  
- **Artifact Management:** Security reports and build artifacts properly uploaded
- **Error Handling:** Graceful degradation when optional tools/configs missing

## Repository Structure & Recent Improvements

**Centralized Development Tools (NEW):**
- **Pyrrha-Development-Tools**: Complete linting system with shared configurations
  - `configs/`: Shared .prettierrc.js, pyproject.toml, .clang-format files
  - `scripts/`: Technology-specific linting orchestration (lint-all.js, lint-*.sh)
  - `templates/`: Standardized GitHub Actions workflows for all repository types

**Repository Naming Updates:**
- Renamed "Pyrrha-Sensor-Simulator" → "Pyrrha-Device-Simulator"
- All documentation and configuration files updated to reflect new naming
- Docker Compose services use consistent naming convention

**Code Quality Standardization (October 2025):**
- **9 repositories** now use standardized GitHub Actions workflows
- **4 technology templates** covering all platform needs:
  - `github-workflow-dashboard.yml`: Multi-stack React + Python + Node.js
  - `github-workflow-python.yml`: Comprehensive Python with security
  - `github-workflow-nodejs.yml`: Node.js/JavaScript services  
  - `github-workflow-cpp.yml`: C/C++/Arduino firmware development
- **100% consistency** in CI/CD triggers, naming, and tooling approaches
- **Proven configurations** extracted from existing successful workflows

## Centralized Code Quality System

**📁 Pyrrha-Development-Tools Repository:** Single source of truth for all code quality standards across the platform.

**Centralized Linting System:**
```bash
# Central linting orchestration
cd Pyrrha-Development-Tools
node scripts/lint-all.js --repo=Pyrrha-Dashboard --check  # Check specific repo
node scripts/lint-all.js --fix                             # Fix all repositories

# Technology-specific scripts
scripts/lint-dashboard.sh --fix    # Multi-stack React + Flask + Node.js
scripts/lint-python.sh --check     # Python with security scanning
scripts/lint-nodejs.sh --fix       # Node.js services
scripts/lint-cpp.sh --check        # C/C++/Arduino firmware
```

**Proven Configuration Standards:**
- **JavaScript/React:** Prettier (singleQuote: true, bracketSameLine: true) + ESLint with React hooks
- **Python:** Black + isort + flake8 + mypy (line-length: 127, matching Rules-Decision proven config)
- **C/C++/Arduino:** clang-format (Google style, 2-space indent, 100-char limit)
- **Infrastructure:** dockerfilelint + documentation formatting
- **Security:** bandit (Python) + safety (dependency scanning) with automated report generation

**Repository Auto-Detection:** System intelligently detects technology stack and applies appropriate tooling:
- **Dashboard:** Multi-technology (React + Python Flask + Node.js Auth)
- **Rules-Decision:** Python analytics service  
- **MQTT-Client/WebSocket-Server/Device-Simulator:** Node.js services
- **Firmware:** C/C++/Arduino development
- **Database:** Infrastructure with SQL validation

## Code Style Conventions

**Python:** Black formatting (line-length 127) + isort + flake8 + mypy + security scanning
**JavaScript/React:** Prettier + ESLint with proven Dashboard configuration, React hooks validation
**C/C++/Arduino:** clang-format with Google style, Arduino project validation  
**Node.js Services:** Prettier + ESLint (when configured) + dependency caching
**Database:** Parameterized queries exclusively + SQL file validation
**Infrastructure:** Dockerfile linting + documentation formatting consistency

## Testing & Debugging

**Service Health:** Check `docker compose ps` for container status
**Database Access:** Connect directly to MariaDB container for debugging queries
**Log Aggregation:** Each service logs independently - use `docker compose logs [service]`

**End-to-End Verification:**
1. Check device simulator is publishing: `docker compose logs pyrrha-simulator`
2. Verify MQTT client connectivity: `docker compose logs pyrrha-mqttclient`
3. Confirm database entries: `SELECT COUNT(*) FROM firefighter_device_log WHERE timestamp_mins >= NOW() - INTERVAL 5 MINUTE;`
4. Test WebSocket broadcasting: `docker compose logs pyrrha-wss`

**Quality Assurance Workflow:**
```bash
# Test centralized linting system
cd Pyrrha-Development-Tools
node scripts/lint-all.js --repo=Pyrrha-Dashboard --check    # Dashboard multi-stack
node scripts/lint-all.js --repo=Pyrrha-Rules-Decision --fix # Python formatting
node scripts/lint-all.js --repo=Pyrrha-Firmware --check     # C++ Arduino validation

# Validate specific technology stacks
scripts/lint-python.sh --check    # Python repositories
scripts/lint-nodejs.sh --check    # Node.js services  
scripts/lint-cpp.sh --check       # Firmware development
scripts/lint-dashboard.sh --check # Multi-technology dashboard

# Verify system after changes
cd ../Pyrrha-Deployment-Configurations
docker compose down -v && docker compose up -d  # Full system rebuild test
curl http://localhost:5000/api-main/v1/health   # API health check
open http://localhost:3000                       # Dashboard verification
```

**Clean Environment Setup:**
```bash
# Complete reset and rebuild
docker compose down -v    # Removes all volumes and data
docker system prune -f    # Cleans up Docker resources
docker compose up -d      # Fresh rebuild with seed data

# System should be operational within 30 seconds
# Database will auto-populate with authentication entries from pyrrha.sql
```

## Current System Status (October 2025)

**✅ Validated Production Environment:**
- All 9 microservices successfully running after code quality improvements
- Dashboard (React): 78% reduction in linting issues (1,622 → 363 problems)
- Rules-Decision (Python): 5 files successfully reformatted with proven Black configuration
- No breaking changes from formatting standardization
- Complete end-to-end system functionality verified

**🔧 Enterprise Code Quality Implementation:**
- **Centralized Linting:** Single source of truth in Pyrrha-Development-Tools
- **Multi-language Support:** JavaScript, Python, C/C++, Infrastructure
- **Proven Configurations:** Extracted from existing successful repository workflows
- **Security Integration:** Automated bandit + safety scanning for Python repositories
- **CI/CD Standardization:** All repositories use consistent GitHub Actions patterns

**🎯 Platform Maintainability:**
- **Repository Auto-Detection:** Intelligent technology stack identification
- **Configuration Inheritance:** Centralized configs applied across repositories  
- **Fix Mode Support:** Automated formatting fixes via centralized scripts
- **Documentation Consistency:** Standardized README and contributing guidelines
- **Dependency Management:** Coordinated package versions across services

**📊 Quality Metrics Achieved:**
- **9 repositories** with standardized linting workflows
- **4 technology stacks** properly configured and validated
- **1 centralized system** managing all code quality standards
- **0 breaking changes** from quality improvements
- **100% operational** system after complete rebuild verification

## Samsung Accessory Protocol Integration (November 2025)

**📱 Mobile-Watch Communication Pipeline:**
- **Architecture**: Mobile App (Provider) → Samsung Accessory Protocol → Galaxy Watch (Consumer)
- **Data Flow**: Prometeo Device (BLE) → Mobile App → ProviderService → Galaxy Watch Display
- **Platform Support**: Samsung Galaxy A51 (Android 14/API 34) + Galaxy Watch 3 (Tizen 5.5)

**🔧 Technical Implementation:**
- **ProviderService.java**: Samsung Accessory Protocol provider extending SAAgent
- **Service Configuration**: `accessoryservices.xml` with PyrrhaMobileProvider profile
- **Communication Channel**: Channel ID 104 with high priority, reliable transmission
- **JSON Protocol**: Structured sensor data with timestamps, status, and alert thresholds
- **Connection Management**: Automatic watch discovery, multi-device support, reconnection logic

**📊 Integration Status (November 2025):**
- **Mobile App Modernization**: ✅ Complete - Android 14 compatibility, Samsung Accessory SDK integration
- **Watch App Enhancement**: ✅ Complete - Tizen 5.5 optimization, circular UI redesign, ES6+ JavaScript
- **Samsung Protocol**: ✅ Complete - Real-time sensor data transmission pipeline implemented
- **Build Status**: ✅ Complete - Zero compilation errors, comprehensive documentation created
- **Testing Phase**: 🔄 Ready for end-to-end validation with physical Samsung devices

**🎯 Data Transmission Features:**
- **Update Frequency**: 3-second intervals for optimal battery life and real-time monitoring
- **Sensor Validation**: CO readings (0-1000 ppm), NO2 readings (0-10 ppm) with error filtering  
- **Alert System**: Status calculation (normal/warning/alert) based on safety thresholds
- **Error Handling**: Comprehensive Samsung Accessory Protocol error codes and recovery
- **Multi-Device**: Support for multiple Galaxy Watch connections per mobile app instance