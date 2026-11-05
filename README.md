# Crave — Food Delivery Platform

Two production-shaped microservices that together implement a food-delivery platform:
customers order from restaurants, pay online or cash-on-delivery, and delivery agents
pick up and deliver the food — with live updates pushed to every client app.

This repo is the **system overview**. The code lives in two sibling repositories:

| Service | Repo | Role |
| --- | --- | --- |
| `core-service` | [marwanoosama/crave-core-service](https://github.com/marwanoosama/crave-core-service) | Identity, restaurants, branches, catalog, RBAC — the **read-mostly** side. |
| `order-service` | [marwanoosama/crave-order-service](https://github.com/marwanoosama/crave-order-service) | Orders, payments, deliveries, agent assignment, restaurant finance, WebSocket — the **write/money** side. |
