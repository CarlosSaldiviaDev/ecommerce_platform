# E‑commerce Tire Platform

## Overview

E‑commerce Tire Platform is a full‑stack e‑commerce system for tire products. It provides a web storefront for browsing and purchasing, plus an administrative dashboard for managing catalog, orders, users, and configuration. The system is built as a distributed application with a single-page frontend and a microservices backend.

## Architecture Summary

The platform follows a **client–server** and **microservices** architecture:

- **Client:** A React single-page application (SPA) that handles authentication, product catalog, cart, orders, and admin workflows. It communicates with backend services over HTTP.
- **Backend:** Multiple Spring Boot microservices, each responsible for a bounded domain (authentication, users, products and orders, and management/administration). Services share a common library and a relational database, with a cache layer for performance and resilience.
- **Data:** A relational database stores persistent data; a cache is used for session and frequently accessed data.

This separation allows independent scaling and deployment of frontend and backend services while keeping the system understandable and maintainable.

## Main Technologies

| Layer | Technologies |
|-------|--------------|
| **Frontend** | React, TypeScript, Vite, Redux Toolkit, React Router, Tailwind CSS, Axios |
| **Auth / Identity** | JWT, Firebase (e.g. Google Sign-In), backend auth service |
| **Backend** | Java, Spring Boot, Gradle |
| **Data** | Relational database (SQL), in-memory cache |
| **DevOps / Run** | Docker, Docker Compose |

## High-Level System Flow

1. **User access:** The user opens the web application in a browser. The SPA loads and routes to the appropriate view (e.g. home, catalog, cart, login, or dashboard).
2. **Authentication:** For protected actions, the client obtains a token (e.g. via login or third-party provider). The token is sent with subsequent API requests; backend services validate it and enforce authorization.
3. **Catalog and cart:** The user browses categories and products and adds items to the cart. Product and category data are requested from the backend via REST; cart state can be kept in the client (e.g. Redux/local storage) until checkout.
4. **Checkout and orders:** Checkout creates an order by sending cart and user data to the order API. The order service persists the order and may trigger notifications. Users and admins can list and view order details via dedicated endpoints.
5. **Administration:** Authenticated admin users use dashboard routes to manage products, categories, users, and configuration (e.g. exchange rates). These actions call the corresponding management, product, and user backend services.
6. **Backend processing:** Each microservice handles its domain (login, users, products/orders, management), validates requests, uses the shared database and cache as needed, and returns structured JSON responses.

End-to-end, the flow is: **Browser → SPA (React) → HTTP/REST → Microservices → Database / Cache → Response back to SPA.**

## Main Capabilities

- **Customer-facing:** User registration and login (including social login), product catalog with categories and search, product detail views, shopping cart, order placement, order history and order detail views, user profile and account management, password reset.
- **Administrative:** Dashboard for managing products (create, edit), categories (create, edit), users (edit), exchange-rate and other configuration; order management with filtering by status and date; user lookup by email.
- **Cross-cutting:** Centralized API error handling and global alerts in the UI; JWT-based authentication and authorization across services; optional email notifications (e.g. for orders or password reset); health checks for backend services; containerized deployment via Docker Compose.

For a more detailed view of components, data flow, and deployment, see [system-overview.md](./system-overview.md).
