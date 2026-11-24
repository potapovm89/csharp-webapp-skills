# Engineering Principles Reference

## SOLID
- **Single Responsibility (SRP)**: Each class has one reason to change. Controllers only handle HTTP orchestration; commands encapsulate one business use-case; domain services manage entity persistence.
- **Open/Closed (OCP)**: Extend behavior via interfaces, middleware, and domain service overrides without modifying existing tested core classes.
- **Liskov Substitution (LSP)**: Derived domain services (`BaseLDomainService<T>`) and entities must honor base class contracts without unexpected side-effects.
- **Interface Segregation (ISP)**: Clients depend only on interfaces they use (e.g. `IKnowCreator`, `ICanBeSoftDeleted`, `ISynchronizable`).
- **Dependency Inversion (DIP)**: Depend on abstractions (`IOrderDomainService`, `IMediator`, `ITransactionScope`), inject via constructor DI.

## KISS (Keep It Simple, Stupid)
- Favor readable, explicit C# code over clever metaprogramming or deep inheritance hierarchies.
- Choose straightforward LINQ queries; avoid overcomplicated single-line expressions.

## DRY (Don't Repeat Yourself)
- Extract common infrastructure (middleware, filters, exceptions, option binders) into `Itl.<Project>.Common`.
- Share base testing classes (`BaseTestCasesClass`) in `Itl.<Project>.Tests`.
- Do not duplicate DTO definitions across projects; use `Itl.<Project>.Contracts`.

## YAGNI (You Aren't Gonna Need It)
- Do not build speculative abstractions or unused endpoints ahead of requirements.
- Implement only what is required by the current task.
