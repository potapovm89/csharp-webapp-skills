# DTO Conventions Reference

## Contracts vs Domain DTOs
- **Contracts (`Itl.<Project>.Contracts`)**: Public DTOs for external consumers, REST API, gRPC, and message brokers.
  - Suffix `Dto` is mandatory (e.g. `OrderDto`, `CreateCustomerRequestDto`).
  - Common DTOs in root (`HttpErrorDto`, `IdDto`).
  - Feature-specific DTOs grouped by domain folders (`Account/`, `Orders/`).
  - No internal project dependencies.
- **Domain DTOs (`Itl.<Project>.Domain/Dtos`)**: Internal projections and data transfer objects used purely within the solution, not exposed as public inter-service contracts.

## Rules for DTOs
- Decorated with `[DataContract]` and `[DataMember]`.
- All properties have XML comments ending with a period.
- Required properties marked with C# `required` keyword.
- Separate Request and Response DTOs: never reuse domain entities in public contracts.
