# C# Naming Conventions Reference

## Constants
Upper case with underscores:
```csharp
const double PI_VALUE = 3.14;
const string DEFAULT_TOPIC = "orders-stream";
```

## Classes, Records, Structs, Enums, Interfaces
PascalCase:
```csharp
public class OrderProcessor;
public record UserInfoDto;
public struct GeoPoint;
public enum OrderStatus;
public interface IOrderDomainService; // Starts with 'I'
```

## Methods and Properties
PascalCase at every accessibility, for instance and static properties:
```csharp
public string FirstName { get; set; } = string.Empty;
public async Task<Guid> CreateOrderAsync(...) { }
```

## Private and Protected Fields
Non-constant `private`, `protected`, `protected internal` and `private protected` fields use `_camelCase`. Constants follow the rule above:
```csharp
private readonly IMediator _mediator;
protected int _retryCount;
```

## Test Methods
Three-part pattern: `[TestedMethod]_[Condition]_[ExpectedResult]`
```csharp
[Fact]
public void GetUser_WithCorrectId_ShouldReturnInfo() { }

[Theory]
[ClassData(typeof(CreatePaymentCommandValidatorTestCases))]
public void CreatePaymentCommandValidator_InvalidRequests_ShouldThrow(...) { }
```

## Database & External Protocol Naming
- Database tables and columns: `snake_case` (e.g. `store_orders`, `customer_id`, `created_date`).
- REST URL paths: `kebab-case` (e.g. `api/payment-receipts`).
- Kafka topics / RabbitMQ queues: `kebab-case` (e.g. `order-created-events`).
