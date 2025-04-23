
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
- Follow Microsoft C# conventions.
- Prefer explicit typing over `var` unless it improves readability.
- Use expression-bodied members for short methods or properties.
- Use nullable reference types (`<Nullable>enable</Nullable>` in .csproj).
- Consistent naming (PascalCase for classes, camelCase for variables, _camelCase for private fields).

### 3.2. Method Design Standards
```csharp
public async Task<ResponseDto> DoSomethingAsync(InputDto input)
{
    // 1. Validate Input
    // 2. Transform
    // 3. Business Logic
    // 4. Data Access
    // 5. Return
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
// ❌ Bad
int a = 5; // assigning 5

// ✅ Good
const int MaxRetryCount = 5;
```

### 3.5. Null Safety
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

### 4.4. Use Method Syntax (Standard)
```csharp
// ✅ Method Syntax
var users = await _context.Users
    .Where(u => u.IsActive)
    .OrderByDescending(u => u.CreatedAt)
    .ToListAsync();

// ❌ Query Syntax
var users = (from u in _context.Users
             where u.IsActive
             select u).ToList();
```

---

## 5. 🔐 Security & Validation

- Use FluentValidation for API layer input validation.
- Never expose domain entities in API responses; use DTOs.
- Use ASP.NET Core’s built-in policy-based authorization.

---

## 6. 🧪 Unit Testing Guidelines

| Layer          | Test Type     |
|----------------|---------------|
| Domain         | Pure unit tests |
| Application    | Service tests with mocks |
| API            | Integration tests |
| Infrastructure | In-memory integration tests |

---

## 7. ❌ Error Handling

Use global exception handling via middleware in the API layer.

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

    private static Task HandleExceptionAsync(HttpContext context, Exception ex)
    {
        var statusCode = ex switch
        {
            ArgumentException => 400,
            UnauthorizedAccessException => 401,
            _ => 500
        };

        var error = new
        {
            title = "An error occurred.",
            detail = ex.Message,
            status = statusCode,
            traceId = context.TraceIdentifier
        };

        context.Response.ContentType = "application/json";
        context.Response.StatusCode = statusCode;

        return context.Response.WriteAsJsonAsync(error);
    }
}
```

---

## 8. 📜 Logging Standards (Serilog)

### 8.1. Setup
```csharp
Log.Logger = new LoggerConfiguration()
    .Enrich.FromLogContext()
    .WriteTo.Console()
    .WriteTo.File("Logs/log-.txt", rollingInterval: RollingInterval.Day)
    .CreateLogger();

builder.Host.UseSerilog();
```

### 8.2. Usage
```csharp
_logger.LogInformation("Creating order for user {UserId}", userId);
_logger.LogError(ex, "Error processing order {OrderId}", orderId);
```

### 8.3. Best Practices
- Use structured logging (JSON).
- Enrich logs with trace ID and user context.
- Avoid logging sensitive data.

