
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
| **CrossCutting** | Logging, caching, validation, and other shared concerns              |

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
## 4. 🗄️ Entity Framework Core Standards

### 4.1. 🧱 Design Philosophy

- Treat EF Core as an **infrastructure concern**. It should not bleed into domain or application layers.
- Use the **Code First** approach with **Fluent API** for all entity configuration.
- Domain entities must not contain any EF-specific annotations or logic.
- Organize persistence logic in a **Data Access Layer** (Infrastructure project).

---

### 4.2. 📁 DbContext Guidelines

#### ✅ Structure
```csharp
public class AppDbContext : DbContext
{
    public AppDbContext(DbContextOptions<AppDbContext> options)
        : base(options) { }

    public DbSet<Order> Orders => Set<Order>();
    public DbSet<Customer> Customers => Set<Customer>();

    protected override void OnModelCreating(ModelBuilder modelBuilder)
    {
        modelBuilder.ApplyConfigurationsFromAssembly(typeof(AppDbContext).Assembly);
    }
}
```

#### 📌 Guidelines
- Use `ApplyConfigurationsFromAssembly` to load all entity configurations.
- Separate entity mappings into individual configuration files (one per entity).
- Use `DbSet<T>` only for root aggregates that require querying.

---

### 4.3. 🔧 Entity Configuration (Fluent API)

```csharp
public class OrderConfiguration : IEntityTypeConfiguration<Order>
{
    public void Configure(EntityTypeBuilder<Order> builder)
    {
        builder.ToTable("Orders");

        builder.HasKey(o => o.Id);

        builder.Property(o => o.TotalAmount)
               .HasPrecision(18, 2)
               .IsRequired();

        builder.HasOne(o => o.Customer)
               .WithMany(c => c.Orders)
               .HasForeignKey(o => o.CustomerId);
    }
}
```

- Avoid `[Required]`, `[Key]`, etc. in domain models.
- Use `IsRequired`, `HasMaxLength`, etc. only in configuration classes.

---

### 4.4. 🔍 Query Guidelines

| Practice               | Recommendation                                 |
|------------------------|------------------------------------------------|
| Read-only queries      | Use `.AsNoTracking()`                          |
| Related data           | Prefer `.Include()` or projection via `.Select()` |
| Avoid over-fetching    | Use projection to DTOs in query itself         |
| Paginate results       | Always use `.Skip()` and `.Take()`             |

#### ✅ Example Query with Projection
```csharp
var orders = await _context.Orders
    .AsNoTracking()
    .Where(o => o.CustomerId == customerId)
    .Select(o => new OrderDto
    {
        Id = o.Id,
        Total = o.TotalAmount,
        CreatedAt = o.CreatedDate
    })
    .ToListAsync();
```

---

### 4.5. 📚 Repository Usage

- Use **feature-specific repositories** like `IOrderRepository`, not generic ones.
- Repository methods should:
  - Accept/filter by primitive parameters (not full DTOs).
  - Return domain entities or aggregates.
- Keep logic out of `DbContext`. Place it in services or repositories.

---

### 4.6. 🔄 Migrations

#### ⚙️ Best Practices

- Always use meaningful names:
  ```
  dotnet ef migrations add AddOrderTable
  dotnet ef migrations add UpdateCustomerEmailIndex
  ```

- Organize migrations in a dedicated `/Migrations` folder.
- Use automatic `Database.Migrate()` **only in development**:
  ```csharp
  if (app.Environment.IsDevelopment())
  {
      using var scope = app.Services.CreateScope();
      var db = scope.ServiceProvider.GetRequiredService<AppDbContext>();
      db.Database.Migrate();
  }
  ```

---

### 4.7. 🚀 Performance Best Practices

| Tip                          | Why                                  |
|-----------------------------|---------------------------------------|
| Use `.Select()` projection  | Avoid loading unused properties       |
| Use `.AsNoTracking()`       | Improves read performance             |
| Index critical columns      | Improves query speed                  |
| Paginate large collections  | Prevents memory overload              |
| Avoid client-side evaluation| Prevents runtime crashes              |

---

### 4.8. ✅ Testing with EF Core

| Scenario       | Approach                        |
|----------------|----------------------------------|
| Unit Testing   | Mock repository or use in-memory DbContext |
| Integration    | Use SQLite or EF InMemory provider          |
| Seed Data      | Use `HasData()` in Fluent API                |

---

### 4.9. ⚠️ Common Pitfalls to Avoid

| ❌ Pitfall                          | ✅ Better Approach                                 |
|------------------------------------|----------------------------------------------------|
| EF annotations in domain entities  | Use Fluent API in the Infrastructure layer         |
| Returning DbSet<T> directly        | Return DTOs or projected types                     |
| Applying `.ToList()` too early     | Use deferred execution until final transformation  |
| Complex joins in LINQ              | Refactor into simpler projections or raw SQL       |

---

### 4.10. 🔤 LINQ Style Standard

Use **method syntax** only for all LINQ queries:
```csharp
// ✅ Good
var users = _context.Users
    .Where(u => u.IsActive)
    .OrderByDescending(u => u.CreatedAt)
    .ToList();

// ❌ Avoid
var users = (from u in _context.Users
             where u.IsActive
             select u).ToList();
```

---

## 5. 🔐 Security & Validation

### 5.1. ✅ Input Validation

- Perform **all input validation at the API layer**.
- Use [**FluentValidation**](https://docs.fluentvalidation.net/en/latest/) for clean and testable validation rules.
- Never trust client-side validation alone.

#### 🔧 Example: Using FluentValidation
```csharp
public class CreateUserDtoValidator : AbstractValidator<CreateUserDto>
{
    public CreateUserDtoValidator()
    {
        RuleFor(x => x.Email).NotEmpty().EmailAddress();
        RuleFor(x => x.Password).NotEmpty().MinimumLength(8);
    }
}
```

- Register validators in `Program.cs`:
```csharp
builder.Services.AddValidatorsFromAssemblyContaining<CreateUserDtoValidator>();
```

---

### 5.2. 🚫 Never Expose Domain Entities

- Do **not expose domain models** directly in API responses or requests.
- Always use **DTOs** to isolate API contracts from core business logic.

```csharp
// ❌ Avoid
public User GetUser(int id) { ... }

// ✅ Recommended
public UserDto GetUser(int id) { ... }
```

---

### 5.3. 🔐 Authorization

- Use **ASP.NET Core’s built-in policy-based authorization**.
- Group sensitive operations under policies, not just roles.

```csharp
[Authorize(Policy = "RequireAdminRole")]
public IActionResult DeleteUser(int id) { ... }
```

- Use claims-based or scope-based checks for flexibility.

---

### 5.4. 🧱 Layered Security Responsibility

| Layer         | Responsibility                          |
|---------------|------------------------------------------|
| **API**        | Input validation, authentication, and request shaping |
| **Application**| Rule enforcement, access checks (e.g., ownership)     |
| **Domain**     | Invariant enforcement and guard clauses                |

---

### 5.5. ⚠️ Other Security Best Practices

- Do **not log** sensitive values (e.g., passwords, tokens).
- Use HTTPS across all environments.
- Enable CSRF protection (for cookie-based apps).
- Return **problem details** in a structured format without exposing internals.
- Sanitize input where raw SQL is used (if applicable).

---

## 6. 🧪 Unit Testing Guidelines

### 6.1. 🎯 Testing Principles

- Focus on **testing behavior**, not implementation.
- Keep tests **independent**, **repeatable**, and **fast**.
- Use the **Arrange–Act–Assert** pattern in all tests.

---

### 6.2. 🏗️ Testing Strategy by Layer

| Layer           | Type of Test             | Approach                         |
|-----------------|--------------------------|----------------------------------|
| **Domain**      | Unit tests               | Test business logic in isolation |
| **Application** | Service tests            | Mock repositories and services   |
| **API**         | Integration tests        | Test controller + middleware     |
| **Infrastructure** | Integration with EF Core | Use in-memory DB or SQLite       |

---

### 6.3. 🧪 Domain Layer Testing

- No mocks needed.
- Test logic, rules, and value objects.

```csharp
[Fact]
public void TotalAmount_ShouldBeCalculatedCorrectly()
{
    var order = new Order(10, 5); // qty * price
    Assert.Equal(50, order.TotalAmount);
}
```

---

### 6.4. 🧪 Application Layer Testing

- Use mocks (e.g., Moq) for dependencies like repositories.
- Focus on service behavior and orchestration.

```csharp
[Fact]
public async Task CreateOrder_ShouldCallRepository()
{
    var mockRepo = new Mock<IOrderRepository>();
    var service = new OrderService(mockRepo.Object, ...);

    await service.CreateOrderAsync(new OrderDto(...));

    mockRepo.Verify(r => r.AddAsync(It.IsAny<Order>()), Times.Once);
}
```

---

### 6.5. 🧪 API Layer Testing

- Use `WebApplicationFactory<T>` or `TestServer` to test the full HTTP pipeline.
- Validate routing, middleware, and controller behavior.

```csharp
[Fact]
public async Task GetOrders_ShouldReturn200()
{
    var client = _factory.CreateClient();
    var response = await client.GetAsync("/api/orders");
    response.EnsureSuccessStatusCode();
}
```

---

### 6.6. 🧪 Infrastructure Testing

- Use **EF Core InMemory** or **SQLite in-memory** provider.
- Validate real persistence logic with real DbContext.

```csharp
var options = new DbContextOptionsBuilder<AppDbContext>()
    .UseInMemoryDatabase("TestDb").Options;

using var context = new AppDbContext(options);
```

---

### 6.7. 🔁 General Best Practices

- Name test methods clearly: `MethodName_State_ExpectedResult`.
- Avoid shared state between tests.
- Use `[Theory]` with `[InlineData]` for parameterized testing.

---

## 7. ❌ Error Handling

### 7.1. 🎯 Principle

- Handle **all exceptions globally** using middleware at the API layer.
- Avoid `try-catch` blocks in controllers and services unless transforming known exceptions.
- Return **structured and consistent error responses** (e.g., Problem Details format).

---

### 7.2. ⚙️ Global Exception Middleware (Example)

```csharp
public class GlobalExceptionMiddleware
{
    private readonly RequestDelegate _next;
    private readonly ILogger<GlobalExceptionMiddleware> _logger;

    public GlobalExceptionMiddleware(RequestDelegate next, ILogger<GlobalExceptionMiddleware> logger)
    {
        _next = next;
        _logger = logger;
    }

    public async Task InvokeAsync(HttpContext context)
    {
        try
        {
            await _next(context);
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Unhandled exception occurred.");
            await HandleExceptionAsync(context, ex);
        }
    }

    private static Task HandleExceptionAsync(HttpContext context, Exception exception)
    {
        var statusCode = exception switch
        {
            ArgumentException => StatusCodes.Status400BadRequest,
            UnauthorizedAccessException => StatusCodes.Status401Unauthorized,
            KeyNotFoundException => StatusCodes.Status404NotFound,
            _ => StatusCodes.Status500InternalServerError
        };

        var errorResponse = new
        {
            title = "An error occurred while processing your request.",
            detail = exception.Message,
            status = statusCode,
            traceId = context.TraceIdentifier
        };

        context.Response.ContentType = "application/json";
        context.Response.StatusCode = statusCode;

        return context.Response.WriteAsJsonAsync(errorResponse);
    }
}
```

---

### 7.3. 🛠️ Register Middleware in `Program.cs`

```csharp
app.UseMiddleware<GlobalExceptionMiddleware>();
```

---

### 7.4. 🔁 Custom Exceptions

- Define custom exceptions for application-specific rules:
```csharp
public class BusinessRuleViolationException : Exception
{
    public BusinessRuleViolationException(string message) : base(message) {}
}
```

- Handle custom exceptions in the middleware:
```csharp
case BusinessRuleViolationException:
    statusCode = StatusCodes.Status422UnprocessableEntity;
    break;
```

---

### 7.5. 📦 Structured Error Response Format

```json
{
  "title": "An error occurred while processing your request.",
  "detail": "Customer ID cannot be null.",
  "status": 400,
  "traceId": "00-abcd1234efgh5678"
}
```

---

### 7.6. 🛡️ Best Practices

| ✅ Do                          | ❌ Avoid                        |
|-------------------------------|--------------------------------|
| Log exceptions with context   | Logging stack traces in responses |
| Include trace ID in response  | Sending raw exception messages |
| Transform known exceptions    | Catching general `Exception` everywhere |
| Use `ProblemDetails` spec     | Hard-coded error strings       |


---

## 8. 📜 Logging Standards (Using Serilog)

### 8.1. 🎯 Logging Philosophy

- Logging must be:
  - **Structured** – JSON format for easy parsing and searchability.
  - **Centralized** – Use Serilog sinks for console, file, and optional external tools like Seq or Elasticsearch.
  - **Contextual** – Enrich logs with useful metadata like `TraceId`, `UserId`, etc.

---

### 8.2. 🔧 Serilog Setup

#### Install NuGet Packages
```bash
dotnet add package Serilog.AspNetCore
dotnet add package Serilog.Sinks.Console
dotnet add package Serilog.Sinks.File
dotnet add package Serilog.Sinks.Seq
```

#### Configure in `Program.cs`
```csharp
Log.Logger = new LoggerConfiguration()
    .Enrich.FromLogContext()
    .Enrich.WithProperty("Application", "MyApp")
    .WriteTo.Console()
    .WriteTo.File("Logs/log-.txt", rollingInterval: RollingInterval.Day)
    .WriteTo.Seq("http://localhost:5341") // Optional
    .CreateLogger();

builder.Host.UseSerilog();
```

---

### 8.3. ✅ Logging Best Practices

| Where            | What to Log                                | Level        |
|------------------|---------------------------------------------|--------------|
| **API Layer**     | Incoming request, status, traceId          | `Information`|
| **Exception Handler** | Errors with context, stack trace      | `Error`      |
| **Service Layer** | Business events, rules passed or failed    | `Debug` / `Info` |
| **External APIs** | URL, status, latency                       | `Information`/`Warning`|

---

### 8.4. 🧱 Usage Examples

#### Informational Log
```csharp
_logger.LogInformation("User {UserId} placed order {OrderId}", userId, orderId);
```

#### Error Log with Exception
```csharp
_logger.LogError(ex, "Error while processing order {OrderId}", order.Id);
```

---

### 8.5. 🔍 Enriching Logs with Trace ID

To automatically enrich every log with `TraceId`:
```csharp
app.Use(async (context, next) =>
{
    LogContext.PushProperty("TraceId", context.TraceIdentifier);
    await next();
});
```

---

### 8.6. ⚠️ Logging Do’s and Don’ts

| ✅ Do                             | ❌ Don’t                            |
|----------------------------------|------------------------------------|
| Log correlation/trace IDs        | Log passwords or secrets           |
| Use structured messages          | Use string concatenation in logs   |
| Mask PII before logging          | Dump entire objects or responses   |
| Log intent and result            | Log only raw exceptions            |


---

