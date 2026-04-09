#  Automated Greenhouse Management System (AGMS)

AGMS is a cloud-native, microservices-based platform designed for modern precision agriculture. The system automates climate control by fetching real-time telemetry from external IoT sensors and applying custom business rules to maintain ideal growing conditions for crops.

---

## 🏗️ System Architecture

The system is built on the **Spring Cloud** ecosystem and follows a fully distributed microservices architecture with centralized configuration, service discovery, and a unified API gateway entry point.

```
                         ┌─────────────────────┐
                         │     API Gateway      │  :8080
                         │   (Spring Cloud GW)  │
                         └────────┬────────────┘
                                  │ routes all requests
          ┌───────────────────────┼──────────────────────────┐
          │                       │                          │
   ┌──────▼──────┐       ┌───────▼──────┐        ┌─────────▼──────┐
   │ Auth Service│       │ Zone Service │        │ Sensor Service │
   │   :8085     │       │   :8081      │        │    :8082       │
   └─────────────┘       └─────────────┘        └────────────────┘
          │                       │                          │
   ┌──────▼──────┐       ┌───────▼──────┐
   │ Crop Service│       │  Automation  │
   │   :8084     │       │  Service :8083│
   └─────────────┘       └─────────────┘
          │
   ┌──────▼──────────────────────┐
   │  Service Registry (Eureka)  │  :8761
   │  Config Server              │  :8888
   └─────────────────────────────┘
```

### Infrastructure Services

| Service | Port | Description |
|---|---|---|
| Service Registry | `8761` | Eureka Server — service discovery for all microservices |
| Config Server | `8888` | Centralized Git-backed configuration server |
| API Gateway | `8080` | Single entry point; handles routing and JWT auth |

### Domain Microservices

| Service | Port | Description |
|---|---|---|
| Auth Service | `8085` | User registration, login, and JWT token generation |
| Zone Management Service | `8081` | Greenhouse zone and IoT device registration |
| Sensor Telemetry Service | `8082` | Fetches real-time telemetry from external IoT API |
| Automation Service | `8083` | Rule engine — processes sensor data and triggers actions |
| Crop Inventory Service | `8084` | Manages crop batches and growth lifecycle stages |

---

## 🛠️ Tech Stack

| Layer | Technology |
|---|---|
| Language | Java 21 |
| Framework | Spring Boot 3.5.x |
| Cloud | Spring Cloud 2025.0.x (Eureka, Gateway, Config, OpenFeign) |
| Security | JWT (jjwt 0.11.5) |
| Database | MySQL |
| Build Tool | Maven |
| Utilities | Lombok, WebFlux (Reactive HTTP client) |

---

## ✅ Prerequisites

Before running the system, ensure you have the following installed and configured:

- **Java 21** — [Download here](https://adoptium.net/)
- **Maven** — [Download here](https://maven.apache.org/download.cgi)
- **MySQL** — databases must be created as per each service's configuration
- **Git** — required by the Config Server to pull configurations

> **Config Repository:** The Config Server pulls all service configurations from a centralized Git repository.
> Clone or fork the config repo and update the URI in `configserver/src/main/resources/application.properties`:
> ```properties


---

## 🏁 Startup Instructions

> ⚠️ **Order matters.** Services **must** be started in the sequence below. Starting domain services before infrastructure causes startup failures.

### Step 1 — Infrastructure Services (Start First)

**1. Service Registry (Eureka Server)**
```bash
cd serviceregistry
./mvnw spring-boot:run
```
- Port: `8761`
- Wait until the Eureka dashboard is accessible at `http://localhost:8761`

**2. Config Server**
```bash
cd configserver
./mvnw spring-boot:run
```
- Port: `8888`
- Fetches all service configs from the Git repository
- Must be fully UP before any domain service is started

**3. API Gateway**
```bash
cd api-gateway
./mvnw spring-boot:run
```
- Port: `8080`
- All client requests route through this single entry point

---

### Step 2 — Domain Microservices (Start Second)

Once all infrastructure services are healthy, start the domain services (order within this group does not matter):

```bash
# Auth Service
cd auth-service && ./mvnw spring-boot:run

# Zone Management Service
cd zone-service && ./mvnw spring-boot:run

# Sensor Telemetry Service
cd sensor-service && ./mvnw spring-boot:run

# Automation Service
cd automation-service && ./mvnw spring-boot:run

# Crop Inventory Service
cd crop-service && ./mvnw spring-boot:run
```

---

## 🔒 Security

### Internal Security
- All internal API routes are protected via **JWT (JSON Web Token)** authorization enforced at the **API Gateway** level.
- Clients must include a valid `Bearer` token in the `Authorization` header for every protected request.

### External Security
- The **Sensor Telemetry Service** authenticates with the external IoT provider using its own credentials to securely obtain telemetry data — these credentials are never exposed to the client.

### Token Flow
```
Client → POST /auth/register   → receives no token (registration only)
Client → POST /auth/login      → receives { accessToken, refreshToken }
Client → Any protected route   → Authorization: Bearer <accessToken>
Client → POST /auth/refresh    → receives new accessToken using refreshToken
```

---

## 🧪 Testing with Postman

A complete Postman Collection is included in the project root covering all API endpoints across every service.

**File:** `AGMS_Postman_Collection.json`

### Setup Steps

1. Open Postman → **Import** → select `AGMS_Postman_Collection.json`
2. Call **Auth Service → Register** to create a new user account
3. Call **Auth Service → Login** to retrieve your **Bearer Token**
4. For all subsequent requests, the token must be set in the `Authorization` header:
   ```
   Authorization: Bearer <your_token_here>
   ```

### Available API Groups

| Group | Base URL |
|---|---|
| Auth Service | `http://localhost:8080/auth/` |
| Zone Management | `http://localhost:8080/api/zones/` |
| Sensor Telemetry | `http://localhost:8080/api/sensors/` |
| Automation & Control | `http://localhost:8080/api/automation/` |
| Crop Inventory | `http://localhost:8080/api/crops/` |

---

## 📡 Key API Endpoints

### Auth Service
| Method | Endpoint | Description | Auth Required |
|---|---|---|---|
| POST | `/auth/register` | Register a new user | ❌ |
| POST | `/auth/login` | Login and get tokens | ❌ |
| POST | `/auth/refresh` | Refresh access token | ❌ |

### Zone Management
| Method | Endpoint | Description | Auth Required |
|---|---|---|---|
| GET | `/api/zones/{id}` | Get a specific zone | ✅ |
| POST | `/api/zones` | Create a new zone | ✅ |
| PUT | `/api/zones/{id}` | Update zone details | ✅ |
| DELETE | `/api/zones/{id}` | Delete a zone | ✅ |

### Sensor Telemetry
| Method | Endpoint | Description | Auth Required |
|---|---|---|---|
| GET | `/api/sensors/latest` | Get latest sensor readings | ✅ |

### Automation & Control
| Method | Endpoint | Description | Auth Required |
|---|---|---|---|
| POST | `/api/automation/process` | Process sensor data for a zone | ✅ |
| GET | `/api/automation/logs` | Retrieve automation action logs | ✅ |

### Crop Inventory
| Method | Endpoint | Description | Auth Required |
|---|---|---|---|
| GET | `/api/crops` | List all crops for a user | ✅ |
| POST | `/api/crops` | Register a new crop batch | ✅ |
| PUT | `/api/crops/{id}/status` | Update crop lifecycle status | ✅ |

---

## 📊 Monitoring

All registered microservices can be monitored via the **Eureka Dashboard**:

- **URL:** [http://localhost:8761](http://localhost:8761)

A healthy system will show all services (auth-service, zone-service, sensor-service, automation-service, crop-service, api-gateway) registered and `UP`.

See `docs/eureka-dashboard.png` for a visual reference of a healthy system state.

---

## 📁 Project Structure

```
automated-greenhouse-management-system/
├── api-gateway/            # Spring Cloud Gateway — routing & JWT filter
├── auth-service/           # JWT-based user authentication & registration
├── automation-service/     # Rule engine & action logging
├── configserver/           # Centralized Git-backed config server
├── crop-service/           # Crop batch & lifecycle management
├── sensor-service/         # External IoT telemetry bridge (WebFlux)
├── serviceregistry/        # Eureka service discovery server
├── zone-service/           # Greenhouse zone & device management
├── docs/
│   └── eureka-dashboard.png
├── AGMS_Postman_Collection.json
└── README.md
```

---

## 🚀 Quick Start Checklist

- [ ] Java 21 installed
- [ ] MySQL running with required databases created
- [ ] Config repo forked/created and URI updated in `configserver/application.properties`
- [ ] Start **serviceregistry** → wait for `http://localhost:8761`
- [ ] Start **configserver** → wait for `http://localhost:8888`
- [ ] Start **api-gateway**
- [ ] Start all domain microservices
- [ ] Import `AGMS_Postman_Collection.json` into Postman
- [ ] Register a user and login to obtain Bearer Token
- [ ] Test endpoints using the Bearer Token

---

## 📝 License

This project is intended for educational purposes.
