# Unit Testing Reference

## Frameworks & Tooling
- **Framework**: xUnit **v3** (`xunit.v3` packages) for solutions on .NET 10; a legacy solution on xUnit v2 (2.9.x) stays on v2 — do not mix v2 and v3 in one solution, and a migration is a separate task, not a side effect of a feature.
- **Mocks**: Moq (MIT, 4.20.x) is the template standard. The 2023 SponsorLink incident is historical and the feature was removed; if procurement still requires it, NSubstitute is the approved alternative **only via an explicit ADR** — exactly one mocking library per solution.
- **Assertions**: Standard xUnit `Assert` or `FluentAssertions` (version **≤ 7.2.0** strictly enforced — 8.x is commercial, Xceed). If the frozen 7.x line becomes a support problem, the Apache-2.0 continuation fork **AwesomeAssertions** may replace it via an explicit ADR, never ad hoc.

## Project Structure (`tests/Itl.<Project>.UnitTests`)
Mirrors the hosts / stories structure:
- `WebHost/`
- `RpcHost/`
- `Stories/`

## Method Naming Convention
Pattern: `[TestedMethod]_[Condition]_[ExpectedResult]`
```csharp
[Fact]
public void GetUser_WithCorrectId_ShouldReturnInfo() { }

[Theory]
[ClassData(typeof(CreatePaymentCommandValidatorTestCases))]
public void CreatePaymentCommandValidator_InvalidRequests_ShouldThrow(CreatePaymentCommandValidatorTestCase testCase) { }
```

## Table-Driven Validator Tests via `BaseTestCasesClass`
```csharp
public class CreatePaymentCommandValidatorTestCase : BaseInvalidRequest
{
    public CreatePaymentCommand Request { get; set; } = null!;   // always set in the test case initializer
    public override Type ExceptionType { get; set; } = typeof(FluentValidation.ValidationException);
}

public class CreatePaymentCommandValidatorTestCases : BaseTestCasesClass
{
    protected override IEnumerable<object[]> Data =>
    [
        [
            new CreatePaymentCommandValidatorTestCase
            {
                Because = "StoreId is null",
                Request = new CreatePaymentCommand { StoreId = null, Data = ... },
                ExpectedExceptionMessage = "'Store Id' must not be empty."
            }
        ]
    ];
}
```

## Rules
- Unit tests must be pure: zero network, disk, or real database calls.
- Pure logic, custom mappers, and ALL FluentValidation rules must be tested.
