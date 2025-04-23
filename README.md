
# 🔧 Backend Standards Document

## 🧰 Tech Stack
- **Framework:** .NET 8 (ASP.NET Core MVC)
- **Architecture:** Layered (API, Domain, Business Logic, Data Access)
- **ORM:** Entity Framework Core
- **Database:** SQL Server

---

## 1. 🏗️ Project Structure & Layering

### Layered Architecture Overview
```
/src
  ├── API                // Exposes endpoints, maps DTOs, handles requests/responses
  ├── Application        // Business logic, use cases, services
  ├── Domain             // Entities, value objects, enums, interfaces
  ├── Infrastructure     // Data access, external service integrations
  └── Shared             // Common utilities, exceptions, logging, constants
```

### Responsibilities by Layer

| Layer         | Responsibility |
|---------------|----------------|
| **API**        | Controllers, DTO mapping, input validation |
| **Application**| Orchestrates use cases, services, workflows |
| **Domain**     | Core business models and rules |
| **Infrastructure** | Implements repositories, EF DbContext, 3rd party integrations |

---

## 2. 🧾 Naming Conventions

### Classes
- **Controllers:** `{Entity}Controller`
- **Services:** `{Entity}Service`
- **Interfaces:** `I{Entity}Service`
- **Repositories:** `{Entity}Repository`
- **DbContext:** `{ProjectName}DbContext`

### Methods
- Verb-based naming: `GetById`, `CreateAsync`, `UpdateStatus`
- Async methods: suffix with `Async`

### Files & Folders
- Use PascalCase
- Group by feature inside each layer

---

## 3. 💡 Coding Standards

### 3.1. C# Language Conventions
Use explicit typing, expression-bodied members for simple methods, and consistent naming conventions across the codebase.

### 3.2. Method Design Standards
```csharp
public async Task<ResponseDto> DoSomethingAsync(InputDto input)
{
    // 1. Validate Input
    // 2. Perform Transformations (if needed)
    // 3. Execute Business Logic
    // 4. Access Data via Repository
    // 5. Return Response
}
```

### 3.3. Class Design Guidelines
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

### 3.4. Code Hygiene
```csharp
// Bad
int a = 5; // assigning 5

// Good
const int MaxRetryCount = 5;
```

### 3.5. Null Safety
```xml
<Nullable>enable</Nullable>
```

```csharp
if (string.IsNullOrWhiteSpace(name))
    throw new ArgumentException("Name is required", nameof(name));
```

---

## 4. 🗄️ Entity Framework Core Standards

### 4.1. DbContext Design
```csharp
public class AppDbContext : DbContext
{
    public AppDbContext(DbContextOptions<AppDbContext> options)
        : base(options) { }

    public DbSet<Order> Orders => Set<Order>();

    protected override void OnModelCreating(ModelBuilder modelBuilder)
    {
        modelBuilder.ApplyConfigurationsFromAssembly(typeof(AppDbContext).Assembly);
    }
}
```

### 4.2. Fluent API Configuration
```csharp
public class OrderConfiguration : IEntityTypeConfiguration<Order>
{
    public void Configure(EntityTypeBuilder<Order> builder)
    {
        builder.ToTable("Orders");
        builder.HasKey(o => o.Id);
        builder.Property(o => o.TotalAmount).HasPrecision(18, 2);
        builder.HasOne(o => o.Customer)
               .WithMany(c => c.Orders)
               .HasForeignKey(o => o.CustomerId);
    }
}
```

### 4.3. LINQ Query Example
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

### 4.4. Use Method Syntax
```csharp
// ✅ Method Syntax
var users = await _context.Users
    .Where(u => u.IsActive)
    .OrderByDescending(u => u.CreatedAt)
    .ToListAsync();
```

---

## 5. 🔐 Security & Validation

- Validate input in the API layer using libraries like **FluentValidation**.
- Never expose domain entities directly in API responses. Use **DTOs**.
- Implement **policy-based authorization** for protected endpoints.

---

## 6. 🧪 Unit Testing Guidelines

| Layer          | Test Type     |
|----------------|---------------|
| Domain         | Pure unit tests |
| Application    | Service tests with mocks |
| API            | Integration tests (controller + middleware) |
| Infrastructure | Integration tests with in-memory database |

---

## 7. ❌ Error Handling

### Global Exception Middleware
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

## 8. 📜 Logging with Serilog

### Setup in Program.cs
```csharp
Log.Logger = new LoggerConfiguration()
    .Enrich.FromLogContext()
    .WriteTo.Console()
    .WriteTo.File("Logs/log-.txt", rollingInterval: RollingInterval.Day)
    .CreateLogger();

builder.Host.UseSerilog();
```

### Usage Examples
```csharp
_logger.LogInformation("Creating order for user {UserId} with amount {Amount}", userId, amount);
_logger.LogError(ex, "Error occurred while processing order {OrderId}", orderId);
```
