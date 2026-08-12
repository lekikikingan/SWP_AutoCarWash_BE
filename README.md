# HydroLux

Backend REST API for HydroLux, a car wash service management platform. It connects customers who book and pay for wash services online, staff who operate wash stations and the wash queue, and admins who manage stations, staff, pricing, promotions, and subscriptions.

## Features

- **Authentication & Account** – register/login with JWT, change password, manage customer profile and vehicles
- **Booking** – browse stations and service packages, preview price, create/cancel bookings, view upcoming and past bookings
- **Walk-in & Check-in** – check in walk-in customers without a prior booking, scan/confirm check-in for existing bookings
- **Queue Management** – list the wash queue per station, start/complete a wash, cancel when a guest leaves, put a lane into maintenance
- **Payment** – pay for bookings and subscriptions via SePay bank-transfer QR, auto-confirm deposits through a SePay webhook
- **Refund** – request refunds, look up bank account name via VietQR, preview and confirm refunds
- **Loyalty** – track loyalty points and tier, view tier history, scheduled annual tier evaluation and reset
- **Subscription** – purchase/renew unlimited and family-shared wash subscription plans, transfer a vehicle between subscriptions
- **Family Group** – create a family group and add/search/remove members for shared subscriptions
- **Admin Management** – manage employees, stations, wash lanes, provinces/communes, service packages, add-ons, promotions/vouchers, and subscription plans
- **Dashboard** – view operational summary and revenue charts, configure system settings

## Tech Stack

- **Language:** Java 21
- **Framework:** Spring Boot 4.1.0 (Spring Web, Spring Data JPA, Spring Security, Spring WebSocket, Spring Validation)
- **Database:** MySQL (Hibernate ORM)
- **Authentication:** JWT (Nimbus JOSE JWT) + Spring Security
- **Object Mapping:** ModelMapper, MapStruct
- **Boilerplate:** Lombok
- **Build Tool:** Maven
- **Payment Integration:** SePay (bank-transfer QR + webhook), VietQR (bank account lookup)
- **Testing:** Spring Boot Starter Test (JUnit, Mockito)
- **API Tooling:** Postman

## Quick Start

### Prerequisites

- JDK 21
- Maven (or use the bundled `mvnw` / `mvnw.cmd` wrapper)
- MySQL 8.x running locally
- (Optional, for payment/refund flows) a SePay account and API key, and a VietQR client ID/API key

### Installations

1. Clone the repository
   ```bash
   git clone https://github.com/lekikikingan/SWP_AutoCarWash_BE.git
   ```
2. Install dependencies
   ```bash
   ./mvnw clean install
   ```
3. Environment setup

   This project does not use a `.env` file — configuration lives in `src/main/resources/application.properties`. Update it with your own values before running:
   - `spring.datasource.username` / `spring.datasource.password` — your local MySQL credentials
   - `app.jwt.secret` — secret key used to sign JWT tokens
   - `SEPAY_WEBHOOK_API_KEY`, `SEPAY_BANK_ACCOUNT`, `SEPAY_BANK_CODE`, `SEPAY_BANK_ACCOUNT_NAME` — set as environment variables to override the inline dev defaults
   - `VIETQR_CLIENT_ID`, `VIETQR_API_KEY` — set as environment variables (empty by default; refund bank-name lookup fails without them)

4. Development
   ```bash
   ./mvnw spring-boot:run
   ```
   The server starts on port `8080`. The `swp_auto_car_wash` database is auto-created, and tables/seed data are (re)created on every startup (`spring.jpa.hibernate.ddl-auto=create` + `data.sql`).
5. Production Build
   ```bash
   ./mvnw clean package -DskipTests
   java -jar target/AutoCarWash-0.0.1-SNAPSHOT.jar
   ```

## Repository Structure

```
src/main/java/com/swp/autocarwash/
├── auth/              # Login/register, JWT issuing & validation, roles
├── customer/          # Customer profile, vehicles, family group
├── booking/           # Booking creation, cancellation, price preview
├── staff/             # Employee management, walk-in & booking check-in
├── queue/             # Wash queue and lane operations
├── station/           # Stations, provinces, communes
├── wash/              # Wash lane entities/status
├── servicepackage/    # Service packages and add-on services
├── payment/           # Payment processing, SePay webhook
├── refund/            # Refund requests and confirmation
├── loyalty/           # Loyalty points, tiers, tier history
├── subscription/      # Unlimited & family subscription plans
├── promotion/         # Promotions and vouchers
├── system/            # Dashboard and system settings
├── common/            # Shared config, response wrappers, utilities
├── infrastructure/    # Cross-cutting infrastructure code
├── analytics/         # Scaffolded, not yet implemented
├── crm/                # Scaffolded, not yet implemented
└── feedback/          # Scaffolded, not yet implemented

src/main/resources/     # application.properties, data.sql seed data
```

### Architecture Overview

A layered Spring Boot monolith backed by a single MySQL database, organized as one package per business domain (Controller → Service → Repository via Spring Data JPA). Authentication is stateless JWT: `JwtAuthenticationFilter` validates the `Authorization: Bearer <token>` header on each request and resolves the caller's identity/role (`ROLE_ADMIN`, `ROLE_STAFF`, `ROLE_CUSTOMER`) via Spring Security. Payments for bookings and subscriptions are confirmed asynchronously through a SePay webhook endpoint, and loyalty tier evaluation/reset runs as a scheduled job.

### Example API Endpoints

| Endpoint | Method | Description |
|---|---|---|
| `/api/v1/auth/login` | POST | Authenticate a user and issue a JWT |
| `/api/v1/auth/register` | POST | Register a new customer account |
| `/api/bookings` | POST | Create a new booking |
| `/api/bookings/preview-price` | POST | Preview the total price for a booking |
| `/api/bookings/{id}/cancel` | PATCH | Cancel an existing booking |
| `/api/loyalty/profile` | GET | Get the current customer's loyalty points and tier |
| `/api/webhooks/sepay` | POST | Receive SePay bank-transfer payment confirmation |

Example Request (`POST /api/v1/auth/login`):
```bash
curl -X POST http://localhost:8080/api/v1/auth/login \
  -H "Content-Type: application/json" \
  -d '{"identity": "customer@example.com", "password": "yourpassword"}'
```

Example Response:
```bash
{
  "message": "Login successfully",
  "data": {
    "token": "<jwt-token>",
    "email": "customer@example.com",
    "name": "Nguyen Van A",
    "stationId": null
  }
}
```

### Authentication

Call `/api/v1/auth/login` to receive a token, then send it in the `Authorization` header on subsequent requests:

```bash
curl http://localhost:8080/api/loyalty/profile \
  -H "Authorization: Bearer <jwt-token>"
```

### Environment Variables

| Variable | Description |
|---|---|
| `spring.datasource.username` / `spring.datasource.password` | MySQL credentials (set directly in `application.properties`) |
| `app.jwt.secret` | Secret key used to sign JWT tokens |
| `app.jwt.expiration-ms` | JWT token expiration time, in milliseconds |
| `SEPAY_WEBHOOK_API_KEY` | API key SePay sends in the `Authorization: Apikey <key>` header when calling the webhook |
| `SEPAY_BANK_ACCOUNT` | Bank account number linked to SePay, used to build the payment QR |
| `SEPAY_BANK_CODE` | Bank code used to build the SePay QR code |
| `SEPAY_BANK_ACCOUNT_NAME` | Bank account holder name shown on the SePay QR code |
| `VIETQR_LOOKUP_URL` | VietQR API endpoint used to look up a bank account owner's name for refunds |
| `VIETQR_CLIENT_ID` | Client ID for the VietQR lookup API |
| `VIETQR_API_KEY` | API key for the VietQR lookup API |

### Testing

Tests use Spring Boot Starter Test (JUnit + Mockito). Test coverage is currently limited to the application context load test and the loyalty service.

```bash
./mvnw test
```

## License

This is an academic project built for a university course (SWP), not published under a public open-source license. It is not intended for public or commercial redistribution.

## The GitHub Repository

https://github.com/lekikikingan/SWP_AutoCarWash_BE

## Developer Checklist

- [ ] Authentication (login/register, JWT validation) works end-to-end
- [ ] Core endpoints (booking, payment, loyalty, refund, queue) return correct results — verified with Postman
- [ ] Database seeds (`data.sql`) load without errors on a clean start
- [ ] `./mvnw test` passes
- [ ] `application.properties` values are updated for the target environment before running
