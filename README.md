
# 🔧 Backend Standards Document

## 🧰 Tech Stack
- **Framework:** .NET 8 (ASP.NET Core MVC)
- **Architecture:** Layered (API, Domain, Business Logic, Data Access)
- **ORM:** Entity Framework Core
- **Database:** SQL Server

---

## 1. 🏗️ Project Structure & Layering

### 📁 Layered Architecture Overview
```
/src
  ├── API                // Exposes endpoints, maps DTOs, handles requests/responses
  ├── Application        // Business logic, use cases, services
  ├── Domain             // Entities, value objects, enums, interfaces
  ├── Infrastructure     // Data access, external service integrations
  └── Shared             // Common utilities, exceptions, logging, constants
```

### 🧱 Responsibilities by Layer

| Layer           | Responsibility                                                        |
|-----------------|------------------------------------------------------------------------|
| **API**         | HTTP endpoints, request/response mapping, validation, error handling  |
| **Application** | Coordinates use cases, services, workflows, and DTO mapping           |
| **Domain**      | Core business rules, domain models, invariants                        |
| **Infrastructure** | Implements repositories, EF Core DbContext, external service calls  |
| **Shared**      | Reusable components, constants, utilities, common exceptions          |

---

## 2. 🧾 Naming Conventions

### 📦 Class Naming

| Type          | Convention             | Example                    |
|---------------|------------------------|----------------------------|
| Controller    | `{Entity}Controller`   | `UserController`           |
| Service       | `{Entity}Service`      | `OrderService`             |
| Interface     | `I{Entity}Service`     | `IOrderService`            |
| Repository    | `{Entity}Repository`   | `ProductRepository`        |
| DbContext     | `{Project}DbContext`   | `AppDbContext`             |

### 🔤 Method Naming

- Use **PascalCase** for all method names.
- Prefix with **verbs**: `GetById`, `CreateAsync`, `UpdateStatusAsync`, `DeleteById`
- Suffix async methods with `Async`.

### 🗂️ File & Folder Naming

- Use **PascalCase**: `UserService.cs`, `OrderController.cs`
- Group by feature:
  ```
  /Services/Orders/OrderService.cs
  /Controllers/Orders/OrderController.cs
  /Repositories/Orders/OrderRepository.cs
  ```

---

## 3. 💡 Coding Standards

### 3.1. 🔤 C# Language Conventions

- Follow [Microsoft's C# coding conventions](https://learn.microsoft.com/en-us/dotnet/csharp/fundamentals/coding-style/coding-conventions).
- Prefer **explicit typing** (`int`, `string`) over `var`, unless the type is obvious and improves readability.
- Use **expression-bodied members** for one-liners:
  ```csharp
  public bool IsActive => Status == Status.Active;
  ```
- Use `async/await` for all I/O operations.
- Enable nullable reference types:
  ```xml
  <Nullable>enable</Nullable>
  ```

#### Naming Guidelines
| Element           | Convention         | Example                   |
|-------------------|--------------------|---------------------------|
| Classes           | PascalCase         | `UserService`             |
| Interfaces        | Prefix with `I`    | `IUserService`            |
| Methods           | PascalCase + Verb  | `GetUserByIdAsync()`      |
| Private fields    | `_camelCase`       | `_userRepository`         |
| Local variables   | camelCase          | `userId`, `amount`        |
| Constants         | PascalCase         | `MaxRetryCount`           |

---

### 3.2. ⚙️ Method Design Standards

Each method should follow a consistent pattern:
```csharp
public async Task<ResponseDto> ProcessOrderAsync(OrderDto input)
{
    // 1. Validate Input
    // 2. Transform DTO → Domain
    // 3. Business Logic
    // 4. Persist via Repository
    // 5. Return Response
}
```

- Keep methods short and single-responsibility.
- Break long methods into private helpers.

---

### 3.3. 🧱 Class Design Guidelines

- Use **constructor injection** for all dependencies.
- No `static` classes unless for pure utilities.
- Use **SRP**: services should only orchestrate, not perform low-level operations.

```csharp
public class OrderService : IOrderService
{
    private readonly IOrderRepository _orderRepository;
    private readonly INotificationService _notificationService;

    public OrderService(IOrderRepository orderRepository, INotificationService notificationService)
    {
        _orderRepository = orderRepository;
        _notificationService = notificationService;
    }

    public async Task<OrderDto> CreateOrderAsync(OrderDto dto)
    {
        var order = new Order(dto.CustomerId, dto.Amount);
        await _orderRepository.AddAsync(order);
        await _notificationService.NotifyOrderCreated(order.Id);
        return new OrderDto(order);
    }
}
```

---

### 3.4. 🧹 Code Hygiene

- Avoid unnecessary comments; use clear naming instead.
- Prefer early returns over nested `if`.
- Group logic consistently within methods:
  1. Validation
  2. Transformation
  3. Persistence
  4. Response

```csharp
if (amount <= 0)
    throw new ArgumentOutOfRangeException(nameof(amount), "Amount must be positive");
```

---

### 3.5. 🧼 Null Safety & Defensive Programming

- Use null guards for all public-facing APIs.
- Use non-nullable collections by default:
  ```csharp
  var items = new List<Item>(); // avoid List<Item>?
  ```
- Avoid null in DTOs; use default values if applicable.

---

### 3.6. ✅ Async Guidelines

- Always suffix async methods with `Async`
- Never use `async void` unless it's an event handler
- Chain async calls instead of blocking:
  ```csharp
  var result = await _service.CalculateAsync();
  ```

---

### 3.7. 🧪 Logging and Error Handling in Methods

- Use structured logging:
  ```csharp
  _logger.LogInformation("Processing order {OrderId}", order.Id);
  ```

- Do not swallow exceptions:
  ```csharp
  try
  {
      await _repository.SaveAsync();
  }
  catch (DbUpdateException ex)
  {
      _logger.LogError(ex, "Database error");
      throw new ApplicationException("Could not save order");
  }
  ```


## 4. 🗄️ Entity Framework Core Standards

### 4.1. Design Philosophy

- EF Core lives only in the Infrastructure layer.
- Use Fluent API only; no annotations in domain models.

### 4.2. DbContext Guidelines
```csharp
public class AppDbContext : DbContext
{
    public AppDbContext(DbContextOptions<AppDbContext> options) : base(options) { }

    protected override void OnModelCreating(ModelBuilder modelBuilder)
    {
        modelBuilder.ApplyConfigurationsFromAssembly(typeof(AppDbContext).Assembly);
    }
}
```

---

### 4.3. Entity Configuration (Fluent API)

```csharp
public class OrderConfiguration : IEntityTypeConfiguration<Order>
{
    public void Configure(EntityTypeBuilder<Order> builder)
    {
        builder.ToTable("Orders");
        builder.HasKey(o => o.Id);
        builder.Property(o => o.TotalAmount).HasPrecision(18, 2);
    }
}
```

---

### 4.4. Query Guidelines

- Use `.AsNoTracking()` for read-only queries.
- Use `.Select()` for projection.

---

### 4.5. Repository Usage

- Use feature-specific repositories (`IOrderRepository`).
- Do not expose `DbSet<T>` or query logic from services.

---

### 4.6. Migrations

```bash
dotnet ef migrations add AddOrderTable
```

Use `Database.Migrate()` only in development.

---

### 4.7. Performance Best Practices

- Use projection (`.Select()`).
- Index frequently queried fields.
- Paginate large results.

---

### 4.8. EF Core Testing

- Use InMemory or SQLite for integration tests.
- Use `HasData()` for seed data.

---

### 4.9. Pitfalls to Avoid

| ❌ Pitfall        | ✅ Better Approach         |
|------------------|----------------------------|
| Annotations      | Fluent API                 |
| Early `.ToList()`| Use deferred execution     |

---

### 4.10. LINQ Style

```csharp
// ✅ Method syntax only
var users = _context.Users
    .Where(u => u.IsActive)
    .ToList();
```

---

## 5. 🔐 Security & Validation

- Validate all inputs using FluentValidation.
- Use DTOs, never expose domain entities.
- Use policy-based authorization in ASP.NET Core.

---

## 6. 🧪 Unit Testing Guidelines

| Layer           | Type              | Notes                        |
|-----------------|-------------------|------------------------------|
| Domain          | Unit tests        | No mocks                     |
| Application     | Unit tests        | Mock dependencies            |
| API             | Integration tests | Test HTTP layer              |
| Infrastructure  | EF tests          | Use InMemory or SQLite       |

---

## 7. ❌ Error Handling

- Use global exception middleware.
- Return structured responses (with status, detail, traceId).

---

## 8. 📜 Logging Standards (Serilog)

- Use structured logs via Serilog.
- Log request context with `TraceId`, user info.
- Avoid logging sensitive data.
- Use log levels consistently (`Information`, `Error`, `Debug`, etc.)

---

