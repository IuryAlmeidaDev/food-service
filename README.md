# Food Service

Microsserviço de planejamento de refeições e lista de compras da plataforma **LifeOS**.

## Stack

- Java 21
- Quarkus 3.37.3
- PostgreSQL (`food_db`)
- Apache Kafka (SmallRye Reactive Messaging)
- Flyway (migrations)

## Identidade

| Campo | Valor |
|-------|-------|
| groupId | `dev.iury.lifeos` |
| artifactId | `food-service` |
| Pacote raiz | `dev.iury.lifeos.food` |
| Porta HTTP | `8083` |

## Documentação

- [Design Spec](docs/superpowers/specs/2026-09-08-food-service-design.md)
