# Protobuf Structure & Naming Conventions

## Project Structure (`Itl.<Project>.Protos`)
```
Protos/
├── Common/                # Shared messages (models.proto, pagination.proto)
├── Parcial/               # C# partial classes for generated models (UUIDModel, etc.)
└── Services/              # One proto file per service (<name>_service.proto)
```

## Naming Standards
- Service name: `PascalCase`, ending with `Service` (e.g. `OrderManagementService`).
- Methods: `PascalCase` (e.g. `CreateOrder`, `GetOrderStatus`).
- Request / Response models: Named after the method: `<MethodName>Request` and `<MethodName>Response` (e.g. `CreateOrderRequest`, `CreateOrderResponse`).
- Fields: `snake_case` in proto (e.g. `customer_email`, `product_id`).
- File names: `snake_case` (e.g. `order_management_service.proto`).
- Enums: PascalCase enum name, values prefixed with UPPER_SNAKE_CASE of the enum name.

## Header Template
```protobuf
syntax = "proto3";

import "Common/models.proto";

package itl.<project>.v1;

option csharp_namespace = "Itl.<Project>.Protos.Services.Orders";
```

## Compatibility Rules (wire format)
Clients and servers are deployed independently, so `.proto` changes must keep the wire format compatible:
- **Never change or reuse a field number.** When a field is removed, reserve both its number and name:
  ```protobuf
  message CreateOrderRequest {
    reserved 4;
    reserved "legacy_discount";
    string order_id = 1;
  }
  ```
- Never change a field's type or switch between `optional`/`repeated`; add a new field instead.
- Adding a field, a message, an RPC method or an enum value is compatible. Enums start with `<ENUM>_UNSPECIFIED = 0`, and receivers handle unknown values.
- Renaming a field is wire-compatible but breaks generated code and JSON transcoding — treat it as breaking.
- Breaking changes go to a new package version (`itl.<project>.v2`) served side by side with v1 until all clients migrate.
- Deadlines: clients always set a deadline (per call or a default in the client factory); servers pass `context.CancellationToken` everywhere.
- Retries only for idempotent methods and only on `Unavailable` / `DeadlineExceeded`.
