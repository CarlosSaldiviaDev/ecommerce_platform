# Repository Analysis — E‑commerce Tire Platform

High-level analysis of the **EcommerceTire** repository: inferred architecture, main dependencies, and system purpose. Based on code and configuration exploration, without exposing sensitive data.

---

## 1. System Purpose

The system is an **e‑commerce platform focused on tire sales**. It provides:

- **Storefront:** Product catalog by category, product detail, cart, checkout, and order management for end customers.
- **Administration dashboard:** Management of products, categories, users, shipping/payment types, user types, exchange rates, profit percentages, and orders (by status, date, and user email).
- **Identity and users:** Registration, login (web and external provider, e.g. Google), password recovery, profile and user data (including addresses).

The business domain is reflected in entities such as product, category, order, order detail, user, address, payment type, shipping type, tax, unit, user type, order status, exchange rate, and profit percentage. There is support for monthly cash flows and products sold, consistent with a tire retail business and internal operations.

---

## 2. Inferred Architecture

### 2.1 Overview

- **Client:** A single-page web application (SPA) that consumes REST APIs.
- **Server:** Multiple microservices sharing a relational database and, in deployment, a cache layer (Redis). Frontend–backend communication is HTTP/HTTPS with JSON; some microservices communicate over HTTP (e.g. product → user).

### 2.2 Frontend (`ecommerce_tire_front/react`)

- **Stack:** React 19, TypeScript, Vite 7, React Router 7, Redux Toolkit, Axios, Tailwind CSS, Heroicons. Integration with Firebase and `@react-oauth/google` for authentication (e.g. Google Sign-In).
- **Folder structure (Clean Architecture style):**
  - **`src/app`:** Entry point (Redux store, router).
  - **`src/core`:** Configuration (axios, service hosts, Firebase), constants (routes, strings, error codes), repository interfaces, utilities, storage (localStorage), global types (ApiResponse), error handling.
  - **`src/domain`:** Models (order, user, product, category, login, cart, payment, tax, etc.) and use cases (login, user, product, order).
  - **`src/data`:** Repository implementations that call microservices via Axios (Login, User, Product, Order).
  - **`src/presentation`:** Pages, components, layouts, state (slices: user, ui, product, cart) and providers (e.g. global alerts).
- **Backend communication:** The frontend knows three “services” by name (login, user, product). Base URLs are injected via environment variables (`VITE_API_URL_*`). A single Axios client is configured per service; authenticated requests carry the JWT token in the Authorization header (Bearer). Response interceptor handles 401 (token expiration) with session cleanup and alert.
- **Routes:** Home, login, register, categories, product detail, cart, orders (list, detail, by status/date, by email), dashboard (products, categories, users, exchange rate), my account, update data, password reset.

### 2.3 Backend (`ecommerce_tire_backend`)

- **Stack:** Java 17, Spring Boot 3.4, Gradle. Each microservice is a separate Gradle project producing an executable JAR.
- **Microservices:**
  - **login-app:** Authentication (login by email/password, JWT generation, health). Uses commons and login repository to fetch user (and user with address).
  - **user-app:** User management (CRUD, token validation, health, optional photo upload and email sending, e.g. password reset). Uses commons.
  - **product-app:** Catalog (products, categories) and orders (create, list, filter by status/date, by user, detail, update). Uses commons and, via WebClient (WebFlux), calls user-app to validate user. Mail service and static resource routes (product/category images).
  - **manage-app:** System administration (health, create/drop tables, load initial data, backups with CSV/Excel, etc.). Uses commons and file libraries (Apache POI, Tika, Commons CSV, etc.).
- **commons-app:** Shared library (dependency `com.ecommerce.tire.commons:commons-app`). Includes: security (JWT, authentication filter, security config), API response model (`ApiResponse`), shared domain models (user, category, options, units, user types, taxes, status, shipping type, payment, exchange rate, profit percentage, app versions), options controller (commons), options service, mail service (SendGrid), utilities (e.g. map conversion, MD5 hash, login validation), global exception handling and logging. No direct Redis usage detected in commons code (Redis is present in Docker as a service linked to containers).
- **Per-microservice pattern:** Typical layers — **presentation** (REST controllers), **application** (services), **domain** (models, repository interfaces), **infrastructure** (JPA implementations, HTTP clients such as `UserClient` in product-app).
- **Database:** All services that persist data use the same MySQL database (shared schema). Tables inferred from entity names and `db` data: user, address, product, category, order, order_details, payment, tax, unit, type_user, status, shipping_type, exchange_rate, profit_percentage, versions_apps, cash_monthly, products_sold, order_completed.
- **Inter-service communication:** product-app acts as consumer of user-app (WebClient) to validate user when required. Login and user do not call other microservices in the reviewed code.
- **Deployment:** Docker Compose defines MySQL, Redis, and the four microservices; each with environment variables for DB URL, Redis, and static paths; manage, user and login have SendGrid/email configuration. A `docker-compose-raspberry.yml` variant exists for ARM.

### 2.4 Flow Summary

1. The user uses the SPA in the browser; the SPA routes and maintains state (Redux, localStorage for token and user).
2. For operations that need the backend, the SPA sends HTTP requests (Axios) to the appropriate microservice (login, user, or product); the JWT token is attached when applicable.
3. Login-app and user-app validate JWT and perform authentication and user logic; product-app validates JWT and, when needed, calls user-app to validate user, and handles products and orders.
4. Each microservice that persists data uses JPA against MySQL; in deployment they share Redis (Compose configuration).
5. Manage-app is used for administrative tasks (schema, initial data, backups) and is not part of the normal end-user flow from the documented frontend.

---

## 3. Main Dependencies

### 3.1 Frontend (package.json)

| Dependency | Purpose |
|------------|---------|
| react, react-dom | UI and rendering |
| react-router-dom | SPA routing |
| @reduxjs/toolkit, react-redux | Global state (user, cart, products, UI) |
| axios | HTTP client to microservices |
| firebase, @react-oauth/google | Authentication (e.g. Google) and backend integration |
| @heroicons/react | Icons |
| vite | Build and dev server |
| typescript | Static typing |
| tailwindcss, @tailwindcss/postcss, autoprefixer, postcss | Styling |
| eslint, typescript-eslint | Linting |

### 3.2 Backend (Gradle)

**Common to all microservices (and commons):**

- Spring Boot 3.4.3 (spring-boot-gradle-plugin, dependency-management).
- spring-boot-starter-web, spring-boot-starter-data-jpa, spring-boot-starter-security.
- mysql:mysql-connector-java (runtime).
- jjwt (api + impl + jackson) for JWT.
- Lombok (compileOnly + annotationProcessor).
- Jackson (com.fasterxml.jackson.core:jackson-databind).
- H2 (runtime, tests).
- spring-boot-starter-test, JUnit Platform.

**commons-app only:**

- SendGrid (com.sendgrid:sendgrid-java) for email.

**product-app only:**

- spring-boot-starter-webflux (WebClient to call user-app).
- com.googlecode.json-simple.
- com.ecommerce.tire.commons:commons-app.
- hibernate-types (Vlad Mihalcea) for entity types.

**manage-app only:**

- com.ecommerce.tire.commons:commons-app.
- Apache Commons Lang3, Commons CSV, Apache POI (poi-ooxml), Apache Tika (tika-core), spring-boot-starter-jdbc.

**login-app and user-app:**

- Base dependencies plus commons-app only (no WebFlux or file libraries).

### 3.3 Infrastructure (Docker)

- MySQL 5.7 image for the database.
- Redis 6 Alpine image for cache.
- Each microservice is built from its own Dockerfile (context in each `*-app`) and may mount commons-app code as a volume. Services expose different ports (manage, user, login, product in the 9020–9050 range in internal documentation).

---

## 4. Conclusions

- **Purpose:** E‑commerce platform for tires with storefront, administration, and user and order management.
- **Architecture:** SPA (React) + microservices (Spring Boot) with shared DB and cache (Redis in deployment); REST/JSON and JWT; inter-service communication (product → user) over HTTP.
- **Key dependencies:** Frontend — React, Vite, Redux, Axios, Firebase/Google OAuth, Tailwind; backend — Spring Boot (web, JPA, security), MySQL, JWT, Commons (JWT, mail, models, security), WebFlux only in product-app, and file/backup libraries in manage-app.

This analysis reflects the repository structure and technology at the architecture and dependency level, without including secrets, internal URLs, or sensitive implementation details.
