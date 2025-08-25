# EntityCoreFramework_dotNET
A comprehensive guide to learning entityCore framework with dotNet
# EntityCore Framework for .NET

A modern, lightweight entity management framework for .NET applications that provides intuitive data modeling, relationship management, and query capabilities with full async/await support.

## Table of Contents

- [Installation](#installation)
- [Quick Start](#quick-start)
- [Core Concepts](#core-concepts)
- [Entity Definition](#entity-definition)
- [Relationships](#relationships)
- [Query Builder](#query-builder)
- [Validation](#validation)
- [Middleware & Interceptors](#middleware--interceptors)
- [Configuration](#configuration)
- [Advanced Features](#advanced-features)
- [API Reference](#api-reference)
- [Examples](#examples)
- [ASP.NET Core Integration](#aspnet-core-integration)
- [Contributing](#contributing)
- [License](#license)

## Installation

### Package Manager Console
```powershell
Install-Package EntityCore
```

### .NET CLI
```bash
dotnet add package EntityCore
```

### PackageReference
```xml
<PackageReference Include="EntityCore" Version="1.0.0" />
```

## Quick Start

```csharp
using EntityCore;
using EntityCore.Attributes;
using System.ComponentModel.DataAnnotations;

// Configure EntityCore
var services = new ServiceCollection();
services.AddEntityCore(options =>
{
    options.UseSqlServer("Server=.;Database=MyApp;Trusted_Connection=true;");
    options.EnableSensitiveDataLogging();
});

// Define an entity
[Entity("Users")]
public class User : EntityBase<Guid>
{
    [Required]
    [MaxLength(100)]
    public string Name { get; set; } = string.Empty;

    [Required]
    [EmailAddress]
    [Unique]
    public string Email { get; set; } = string.Empty;

    [DefaultValue(true)]
    public bool IsActive { get; set; } = true;

    public DateTime CreatedAt { get; set; } = DateTime.UtcNow;

    // Navigation properties
    public virtual ICollection<Order> Orders { get; set; } = new List<Order>();
}

// Use the entity
public class UserService
{
    private readonly IEntityRepository<User> _userRepository;

    public UserService(IEntityRepository<User> userRepository)
    {
        _userRepository = userRepository;
    }

    public async Task<User> CreateUserAsync(string name, string email)
    {
        var user = new User
        {
            Id = Guid.NewGuid(),
            Name = name,
            Email = email
        };

        await _userRepository.AddAsync(user);
        await _userRepository.SaveChangesAsync();
        
        return user;
    }
}
```

## Core Concepts

### Entities
Entities represent data models in your application. They inherit from `EntityBase<TKey>` and define the structure, validation rules, and relationships of your data.

### Repositories
EntityCore uses the repository pattern with `IEntityRepository<T>` to provide a consistent interface for data operations across different storage backends.

### Query Builder
A fluent, strongly-typed API for building complex queries with full IntelliSense support and compile-time safety.

### Unit of Work
Automatic transaction management and change tracking through the Unit of Work pattern.

## Entity Definition

### Basic Entity

```csharp
[Entity("Products")]
public class Product : EntityBase<int>
{
    [Required]
    [MaxLength(100)]
    public string Name { get; set; } = string.Empty;

    [MaxLength(1000)]
    public string? Description { get; set; }

    [Column(TypeName = "decimal(18,2)")]
    [Range(0.01, double.MaxValue)]
    public decimal Price { get; set; }

    [Range(0, int.MaxValue)]
    [DefaultValue(0)]
    public int Stock { get; set; }

    [DefaultValue(true)]
    public bool IsActive { get; set; } = true;

    [Column(TypeName = "json")]
    public Dictionary<string, object>? Metadata { get; set; }

    // Computed property
    [NotMapped]
    public bool IsInStock => Stock > 0;

    // Audit fields (automatically managed)
    public DateTime CreatedAt { get; set; }
    public DateTime? UpdatedAt { get; set; }
    public string? CreatedBy { get; set; }
    public string? UpdatedBy { get; set; }
}
```

### Entity Base Classes

```csharp
// For entities with GUID primary keys
public abstract class EntityBase<TKey> : IEntity<TKey>
{
    [Key]
    public TKey Id { get; set; } = default!;
    
    public DateTime CreatedAt { get; set; } = DateTime.UtcNow;
    public DateTime? UpdatedAt { get; set; }
    public string? CreatedBy { get; set; }
    public string? UpdatedBy { get; set; }
    
    [Timestamp]
    public byte[] RowVersion { get; set; } = Array.Empty<byte>();
}

// For auditable entities
public abstract class AuditableEntity<TKey> : EntityBase<TKey>, IAuditable
{
    public bool IsDeleted { get; set; }
    public DateTime? DeletedAt { get; set; }
    public string? DeletedBy { get; set; }
}
```

### Field Attributes

```csharp
public class User : EntityBase<Guid>
{
    [Required(ErrorMessage = "Name is required")]
    [StringLength(50, MinimumLength = 2)]
    public string Name { get; set; } = string.Empty;

    [EmailAddress]
    [Unique(ErrorMessage = "Email must be unique")]
    public string Email { get; set; } = string.Empty;

    [Phone]
    public string? PhoneNumber { get; set; }

    [Range(18, 120)]
    public int Age { get; set; }

    [RegularExpression(@"^[A-Z][a-z]+$")]
    public string? LastName { get; set; }

    [JsonProperty]
    public UserPreferences? Preferences { get; set; }

    [Encrypted]
    public string? SocialSecurityNumber { get; set; }

    [Index]
    public string? Department { get; set; }

    [ConcurrencyCheck]
    public int Version { get; set; }
}
```

## Relationships

### One-to-Many

```csharp
[Entity("Authors")]
public class Author : EntityBase<Guid>
{
    [Required]
    public string Name { get; set; } = string.Empty;
    
    public string? Bio { get; set; }

    // Navigation property
    public virtual ICollection<Book> Books { get; set; } = new List<Book>();
}

[Entity("Books")]
public class Book : EntityBase<Guid>
{
    [Required]
    public string Title { get; set; } = string.Empty;
    
    public string? Description { get; set; }
    
    [Column(TypeName = "decimal(18,2)")]
    public decimal Price { get; set; }

    // Foreign key
    public Guid AuthorId { get; set; }

    // Navigation property
    [ForeignKey(nameof(AuthorId))]
    public virtual Author Author { get; set; } = null!;
}
```

### Many-to-Many

```csharp
[Entity("Users")]
public class User : EntityBase<Guid>
{
    [Required]
    public string Username { get; set; } = string.Empty;

    // Many-to-many navigation
    public virtual ICollection<Role> Roles { get; set; } = new List<Role>();
}

[Entity("Roles")]
public class Role : EntityBase<Guid>
{
    [Required]
    public string Name { get; set; } = string.Empty;
    
    public string? Description { get; set; }

    // Many-to-many navigation
    public virtual ICollection<User> Users { get; set; } = new List<User>();
}

// Junction entity (optional for explicit control)
[Entity("UserRoles")]
public class UserRole
{
    public Guid UserId { get; set; }
    public Guid RoleId { get; set; }
    public DateTime AssignedAt { get; set; } = DateTime.UtcNow;
    public string? AssignedBy { get; set; }

    [ForeignKey(nameof(UserId))]
    public virtual User User { get; set; } = null!;

    [ForeignKey(nameof(RoleId))]
    public virtual Role Role { get; set; } = null!;
}
```

### One-to-One

```csharp
[Entity("Users")]
public class User : EntityBase<Guid>
{
    [Required]
    [EmailAddress]
    public string Email { get; set; } = string.Empty;

    // One-to-one navigation
    public virtual UserProfile? Profile { get; set; }
}

[Entity("UserProfiles")]
public class UserProfile : EntityBase<Guid>
{
    public string? FirstName { get; set; }
    public string? LastName { get; set; }
    public DateTime? DateOfBirth { get; set; }
    public string? Bio { get; set; }

    // Foreign key (same as primary key for 1:1)
    public Guid UserId { get; set; }

    // Navigation property
    [ForeignKey(nameof(UserId))]
    public virtual User User { get; set; } = null!;
}
```

## Query Builder

### Basic Queries

```csharp
public class UserService
{
    private readonly IEntityRepository<User> _repository;

    public UserService(IEntityRepository<User> repository)
    {
        _repository = repository;
    }

    // Find all
    public async Task<List<User>> GetAllUsersAsync()
    {
        return await _repository.GetAllAsync();
    }

    // Find by ID
    public async Task<User?> GetUserByIdAsync(Guid id)
    {
        return await _repository.GetByIdAsync(id);
    }

    // Find with conditions
    public async Task<User?> GetUserByEmailAsync(string email)
    {
        return await _repository.FirstOrDefaultAsync(u => u.Email == email);
    }

    // Find multiple with conditions
    public async Task<List<User>> GetActiveUsersAsync()
    {
        return await _repository.WhereAsync(u => u.IsActive);
    }
}
```

### Advanced Queries

```csharp
public class ProductService
{
    private readonly IEntityRepository<Product> _repository;

    public ProductService(IEntityRepository<Product> repository)
    {
        _repository = repository;
    }

    public async Task<PagedResult<Product>> SearchProductsAsync(
        string? searchTerm = null,
        decimal? minPrice = null,
        decimal? maxPrice = null,
        string? category = null,
        int page = 1,
        int pageSize = 10)
    {
        var query = _repository.Query();

        // Dynamic filtering
        if (!string.IsNullOrEmpty(searchTerm))
        {
            query = query.Where(p => p.Name.Contains(searchTerm) || 
                                   p.Description!.Contains(searchTerm));
        }

        if (minPrice.HasValue)
        {
            query = query.Where(p => p.Price >= minPrice.Value);
        }

        if (maxPrice.HasValue)
        {
            query = query.Where(p => p.Price <= maxPrice.Value);
        }

        if (!string.IsNullOrEmpty(category))
        {
            query = query.Where(p => p.Category!.Name == category);
        }

        // Include related data
        query = query.Include(p => p.Category)
                     .Include(p => p.Reviews);

        // Sorting
        query = query.OrderByDescending(p => p.CreatedAt);

        // Pagination
        return await query.ToPagedResultAsync(page, pageSize);
    }

    public async Task<ProductStats> GetProductStatsAsync()
    {
        var stats = await _repository.Query()
            .GroupBy(p => p.Category!.Name)
            .Select(g => new ProductCategoryStats
            {
                CategoryName = g.Key,
                ProductCount = g.Count(),
                AveragePrice = g.Average(p => p.Price),
                TotalValue = g.Sum(p => p.Price * p.Stock)
            })
            .ToListAsync();

        return new ProductStats { CategoryStats = stats };
    }
}
```

### Query Extensions

```csharp
public static class QueryExtensions
{
    public static IQueryable<T> WhereActive<T>(this IQueryable<T> query) 
        where T : class, IActivatable
    {
        return query.Where(x => x.IsActive);
    }

    public static IQueryable<T> WhereNotDeleted<T>(this IQueryable<T> query) 
        where T : class, IAuditable
    {
        return query.Where(x => !x.IsDeleted);
    }

    public static IQueryable<T> OrderByCreated<T>(this IQueryable<T> query) 
        where T : class, IEntity
    {
        return query.OrderByDescending(x => x.CreatedAt);
    }

    public static async Task<PagedResult<T>> ToPagedResultAsync<T>(
        this IQueryable<T> query, 
        int page, 
        int pageSize)
    {
        var totalCount = await query.CountAsync();
        var items = await query
            .Skip((page - 1) * pageSize)
            .Take(pageSize)
            .ToListAsync();

        return new PagedResult<T>
        {
            Items = items,
            TotalCount = totalCount,
            Page = page,
            PageSize = pageSize,
            TotalPages = (int)Math.Ceiling((double)totalCount / pageSize)
        };
    }
}
```

## Validation

### Data Annotations

```csharp
public class User : EntityBase<Guid>
{
    [Required(ErrorMessage = "Name is required")]
    [StringLength(100, MinimumLength = 2, ErrorMessage = "Name must be between 2 and 100 characters")]
    public string Name { get; set; } = string.Empty;

    [Required]
    [EmailAddress(ErrorMessage = "Invalid email format")]
    public string Email { get; set; } = string.Empty;

    [Range(18, 120, ErrorMessage = "Age must be between 18 and 120")]
    public int Age { get; set; }

    [RegularExpression(@"^\+?[1-9]\d{1,14}$", ErrorMessage = "Invalid phone number")]
    public string? PhoneNumber { get; set; }

    [Url(ErrorMessage = "Invalid URL format")]
    public string? Website { get; set; }
}
```

### Custom Validation Attributes

```csharp
public class StrongPasswordAttribute : ValidationAttribute
{
    public override bool IsValid(object? value)
    {
        if (value is not string password)
            return false;

        return password.Length >= 8 &&
               password.Any(char.IsUpper) &&
               password.Any(char.IsLower) &&
               password.Any(char.IsDigit) &&
               password.Any(ch => "!@#$%^&*()".Contains(ch));
    }

    public override string FormatErrorMessage(string name)
    {
        return $"{name} must be at least 8 characters long and contain uppercase, lowercase, digit, and special character.";
    }
}

public class User : EntityBase<Guid>
{
    [StrongPassword]
    public string Password { get; set; } = string.Empty;
}
```

### Fluent Validation Integration

```csharp
public class UserValidator : AbstractValidator<User>
{
    public UserValidator()
    {
        RuleFor(x => x.Name)
            .NotEmpty().WithMessage("Name is required")
            .Length(2, 100).WithMessage("Name must be between 2 and 100 characters");

        RuleFor(x => x.Email)
            .NotEmpty().WithMessage("Email is required")
            .EmailAddress().WithMessage("Invalid email format")
            .MustAsync(BeUniqueEmail).WithMessage("Email must be unique");

        RuleFor(x => x.Age)
            .GreaterThanOrEqualTo(18).WithMessage("Must be at least 18 years old")
            .LessThanOrEqualTo(120).WithMessage("Must be less than 120 years old");
    }

    private async Task<bool> BeUniqueEmail(User user, string email, CancellationToken cancellationToken)
    {
        // Check if email is unique
        // Implementation depends on your repository
        return true; // Placeholder
    }
}
```

### Entity-Level Validation

```csharp
public class Order : EntityBase<Guid>, IValidatableObject
{
    public decimal Subtotal { get; set; }
    public decimal Tax { get; set; }
    public decimal Total { get; set; }
    public DateTime OrderDate { get; set; }
    public DateTime? ShippingDate { get; set; }

    public IEnumerable<ValidationResult> Validate(ValidationContext validationContext)
    {
        var results = new List<ValidationResult>();

        // Business rule validation
        if (Total != Subtotal + Tax)
        {
            results.Add(new ValidationResult(
                "Total must equal subtotal plus tax",
                new[] { nameof(Total) }));
        }

        if (ShippingDate.HasValue && ShippingDate < OrderDate)
        {
            results.Add(new ValidationResult(
                "Shipping date cannot be before order date",
                new[] { nameof(ShippingDate) }));
        }

        return results;
    }
}
```

## Middleware & Interceptors

### Entity Interceptors

```csharp
public class AuditInterceptor : IEntityInterceptor
{
    private readonly ICurrentUserService _currentUserService;

    public AuditInterceptor(ICurrentUserService currentUserService)
    {
        _currentUserService = currentUserService;
    }

    public async Task BeforeInsertAsync<T>(T entity, CancellationToken cancellationToken = default) 
        where T : class
    {
        if (entity is EntityBase<object> auditableEntity)
        {
            auditableEntity.CreatedAt = DateTime.UtcNow;
            auditableEntity.CreatedBy = _currentUserService.UserId;
        }
    }

    public async Task BeforeUpdateAsync<T>(T entity, CancellationToken cancellationToken = default) 
        where T : class
    {
        if (entity is EntityBase<object> auditableEntity)
        {
            auditableEntity.UpdatedAt = DateTime.UtcNow;
            auditableEntity.UpdatedBy = _currentUserService.UserId;
        }
    }

    public async Task BeforeDeleteAsync<T>(T entity, CancellationToken cancellationToken = default) 
        where T : class
    {
        if (entity is AuditableEntity<object> softDeleteEntity)
        {
            softDeleteEntity.IsDeleted = true;
            softDeleteEntity.DeletedAt = DateTime.UtcNow;
            softDeleteEntity.DeletedBy = _currentUserService.UserId;
        }
    }
}
```

### Domain Events

```csharp
public class UserCreatedEvent : IDomainEvent
{
    public Guid UserId { get; }
    public string Email { get; }
    public DateTime OccurredOn { get; }

    public UserCreatedEvent(Guid userId, string email)
    {
        UserId = userId;
        Email = email;
        OccurredOn = DateTime.UtcNow;
    }
}

public class User : EntityBase<Guid>
{
    // ... properties ...

    public static User Create(string name, string email)
    {
        var user = new User
        {
            Id = Guid.NewGuid(),
            Name = name,
            Email = email
        };

        // Raise domain event
        user.RaiseDomainEvent(new UserCreatedEvent(user.Id, user.Email));

        return user;
    }
}

public class UserCreatedEventHandler : INotificationHandler<UserCreatedEvent>
{
    private readonly IEmailService _emailService;

    public UserCreatedEventHandler(IEmailService emailService)
    {
        _emailService = emailService;
    }

    public async Task Handle(UserCreatedEvent notification, CancellationToken cancellationToken)
    {
        await _emailService.SendWelcomeEmailAsync(notification.Email);
    }
}
```

## Configuration

### Startup Configuration

```csharp
public class Startup
{
    public void ConfigureServices(IServiceCollection services)
    {
        // Add EntityCore
        services.AddEntityCore(options =>
        {
            // Database provider
            options.UseSqlServer(connectionString, sqlOptions =>
            {
                sqlOptions.EnableRetryOnFailure();
                sqlOptions.CommandTimeout(30);
            });

            // Development settings
            if (Environment.IsDevelopment())
            {
                options.EnableSensitiveDataLogging();
                options.EnableDetailedErrors();
            }

            // Configure conventions
            options.UseNamingConvention(NamingConvention.SnakeCase);
            options.UseSoftDelete();
            options.UseAuditFields();

            // Configure interceptors
            options.AddInterceptor<AuditInterceptor>();
            options.AddInterceptor<ValidationInterceptor>();
        });

        // Add repositories
        services.AddScoped(typeof(IEntityRepository<>), typeof(EntityRepository<>));

        // Add domain event handling
        services.AddMediatR(typeof(UserCreatedEventHandler));

        // Add validation
        services.AddValidatorsFromAssembly(Assembly.GetExecutingAssembly());
    }

    public void Configure(IApplicationBuilder app, IWebHostEnvironment env)
    {
        // Initialize EntityCore
        app.UseEntityCore();
        
        // Apply migrations
        if (env.IsDevelopment())
        {
            app.UseEntityCoreMigrations();
        }
    }
}
```

### Configuration Options

```csharp
public class EntityCoreOptions
{
    public string ConnectionString { get; set; } = string.Empty;
    public DatabaseProvider Provider { get; set; } = DatabaseProvider.SqlServer;
    public bool EnableSensitiveDataLogging { get; set; }
    public bool EnableDetailedErrors { get; set; }
    public int CommandTimeout { get; set; } = 30;
    public bool EnableRetryOnFailure { get; set; } = true;
    public NamingConvention NamingConvention { get; set; } = NamingConvention.Default;
    public bool UseSoftDelete { get; set; }
    public bool UseAuditFields { get; set; }
    public CachingOptions Caching { get; set; } = new();
    public List<Type> Interceptors { get; set; } = new();
}

public class CachingOptions
{
    public bool Enabled { get; set; }
    public TimeSpan DefaultExpiration { get; set; } = TimeSpan.FromMinutes(30);
    public int MaxSize { get; set; } = 1000;
    public CacheProvider Provider { get; set; } = CacheProvider.Memory;
}
```

### appsettings.json

```json
{
  "EntityCore": {
    "ConnectionString": "Server=.;Database=MyApp;Trusted_Connection=true;",
    "Provider": "SqlServer",
    "EnableSensitiveDataLogging": false,
    "EnableDetailedErrors": false,
    "CommandTimeout": 30,
    "EnableRetryOnFailure": true,
    "NamingConvention": "SnakeCase",
    "UseSoftDelete": true,
    "UseAuditFields": true,
    "Caching": {
      "Enabled": true,
      "DefaultExpiration": "00:30:00",
      "MaxSize": 1000,
      "Provider": "Redis"
    }
  }
}
```

## Advanced Features

### Transactions

```csharp
public class OrderService
{
    private readonly IUnitOfWork _unitOfWork;
    private readonly IEntityRepository<Order> _orderRepository;
    private readonly IEntityRepository<OrderItem> _orderItemRepository;

    public OrderService(IUnitOfWork unitOfWork, 
                       IEntityRepository<Order> orderRepository,
                       IEntityRepository<OrderItem> orderItemRepository)
    {
        _unitOfWork = unitOfWork;
        _orderRepository = orderRepository;
        _orderItemRepository = orderItemRepository;
    }

    public async Task<Order> CreateOrderAsync(CreateOrderRequest request)
    {
        using var transaction = await _unitOfWork.BeginTransactionAsync();
        
        try
        {
            var order = new Order
            {
                Id = Guid.NewGuid(),
                CustomerId = request.CustomerId,
                OrderDate = DateTime.UtcNow
            };

            await _orderRepository.AddAsync(order);

            foreach (var item in request.Items)
            {
                var orderItem = new OrderItem
                {
                    OrderId = order.Id,
                    ProductId = item.ProductId,
                    Quantity = item.Quantity,
                    Price = item.Price
                };

                await _orderItemRepository.AddAsync(orderItem);
            }

            await _unitOfWork.SaveChangesAsync();
            await transaction.CommitAsync();

            return order;
        }
        catch
        {
            await transaction.RollbackAsync();
            throw;
        }
    }
}
```

### Migrations

```csharp
[Migration(20240101000001)]
public class CreateUsersTable : Migration
{
    public override void Up(MigrationBuilder migrationBuilder)
    {
        migrationBuilder.CreateTable(
            name: "Users",
            columns: table => new
            {
                Id = table.Column<Guid>(nullable: false),
                Name = table.Column<string>(maxLength: 100, nullable: false),
                Email = table.Column<string>(maxLength: 255, nullable: false),
                IsActive = table.Column<bool>(nullable: false, defaultValue: true),
                CreatedAt = table.Column<DateTime>(nullable: false, defaultValueSql: "GETUTCDATE()"),
                UpdatedAt = table.Column<DateTime>(nullable: true),
                RowVersion = table.Column<byte[]>(rowVersion: true, nullable: false)
            },
            constraints: table =>
            {
                table.PrimaryKey("PK_Users", x => x.Id);
            });

        migrationBuilder.CreateIndex(
            name: "IX_Users_Email",
            table: "Users",
            column: "Email",
            unique: true);
    }

    public override void Down(MigrationBuilder migrationBuilder)
    {
        migrationBuilder.DropTable("Users");
    }
}
```

### Caching

```csharp
public class CachedUserService : IUserService
{
    private readonly IEntityRepository<User> _repository;
    private readonly IMemoryCache _cache;
    private readonly ILogger<CachedUserService> _logger;

    public CachedUserService(IEntityRepository<User> repository, 
                           IMemoryCache cache,
                           ILogger<CachedUserService> logger)
    {
        _repository = repository;
        _cache = cache;
        _logger = logger;
    }

    public async Task<User?> GetUserByIdAsync(Guid id)
    {
        string cacheKey = $"user_{id}";

        if (_cache.TryGetValue(cacheKey, out User? cachedUser))
        {
            _logger.LogDebug("User {UserId} found in cache", id);
            return cachedUser;
        }

        var user = await _repository.GetByIdAsync(id);
        
        if (user != null)
        {
            _cache.Set(cacheKey, user, TimeSpan.FromMinutes(30));
            _logger.LogDebug("User {UserId} cached", id);
        }

        return user;
    }

    [CacheEvict("user_{0}")]
    public async Task<User> UpdateUserAsync(Guid id, UpdateUserRequest request)
    {
        var user = await _repository.GetByIdAsync(id);
        if (user == null)
            throw new EntityNotFoundException($"User with ID {id} not found");

        user.Name = request.Name;
        user.Email = request.Email;

        await _repository.UpdateAsync(user);
        await _repository.SaveChangesAsync();

        return user;
    }
}
```

## API Reference

### IEntityRepository<T>

#### Query Methods
```csharp
Task<T?> GetByIdAsync<TKey>(TKey id, CancellationToken cancellationToken = default);
Task<List<T>> GetAllAsync(CancellationToken cancellationToken = default);
Task<T?> FirstOrDefaultAsync(Expression<Func<T, bool>> predicate, CancellationToken cancellationToken = default);
Task<List<T>> WhereAsync(Expression<Func<T, bool>> predicate, CancellationToken cancellationToken = default);
Task<bool> AnyAsync(Expression<Func<T, bool>> predicate, CancellationToken cancellationToken = default);
Task<int> CountAsync(Expression<Func<T, bool>>? predicate = null, CancellationToken cancellationToken = default);
IQueryable<T> Query();
```

#### Modification Methods
```csharp
Task<T> AddAsync(T entity, CancellationToken cancellationToken = default);
Task AddRangeAsync(IEnumerable<T> entities, CancellationToken cancellationToken = default);
Task<T> UpdateAsync(T entity, CancellationToken cancellationToken = default);
Task UpdateRangeAsync(IEnumerable<T> entities, CancellationToken cancellationToken = default);
Task DeleteAsync(T entity, CancellationToken cancellationToken = default);
Task DeleteRangeAsync(IEnumerable<T> entities, CancellationToken cancellationToken = default);
Task<int> SaveChangesAsync(CancellationToken cancellationToken = default);
```

### IUnitOfWork

```csharp
Task<IDbContextTransaction> BeginTransactionAsync(CancellationToken cancellationToken = default);
Task<int> SaveChangesAsync(CancellationToken cancellationToken = default);
IEntityRepository<T> Repository<T>() where T : class;
void Dispose();
```

## Examples

### E-commerce Application

```csharp
[Entity("Categories")]
public class Category : EntityBase<int>
{
    [Required]
    [MaxLength(100)]
    public string Name { get; set; } = string.Empty;

    [MaxLength(255)]
    public string? Description { get; set; }

    public virtual ICollection<Product> Products { get; set; } = new List<Product>();
}

[Entity("Products")]
public class Product : EntityBase<Guid>
{
    [Required]
    [MaxLength(200)]
    public string Name { get; set; } = string.Empty;

    [MaxLength(1000)]
    public string? Description { get; set; }

    [Column(TypeName = "decimal(18,2)")]
    [Range(0.01, double.MaxValue)]
    public decimal Price { get; set; }

    [Range(0, int.MaxValue)]
    public int Stock { get; set; }

    public int CategoryId { get; set; }

    [ForeignKey(nameof(CategoryId))]
    public virtual Category Category { get; set; } = null!;

    public virtual ICollection<OrderItem> OrderItems { get; set; } = new List<OrderItem>();
    public virtual ICollection<ProductTag> ProductTags { get; set; } = new List<ProductTag>();
}

[Entity("Tags")]
public class Tag : EntityBase<int>
{
    [Required]
    [MaxLength(50)]
    public string Name { get; set; } = string.Empty;

    public virtual ICollection<ProductTag> ProductTags { get; set; } = new List<ProductTag>();
}

[Entity("ProductTags")]
public class ProductTag
{
    public Guid ProductId { get; set; }
    public int TagId { get; set; }

    [ForeignKey(nameof(ProductId))]
    public virtual Product Product { get; set; } = null!;

    [ForeignKey(nameof(TagId))]
    public virtual Tag Tag { get; set; } = null!;
}

// Service Implementation
public class ProductService : IProductService
{
    private readonly IEntityRepository<Product> _productRepository;
    private readonly IEntityRepository<Category> _categoryRepository;
    private readonly ILogger<ProductService> _logger;

    public ProductService(
        IEntityRepository<Product> productRepository,
        IEntityRepository<Category> categoryRepository,
        ILogger<ProductService> logger)
    {
        _productRepository = productRepository;
        _categoryRepository = categoryRepository;
        _logger = logger;
    }

    public async Task<PagedResult<ProductDto>> SearchProductsAsync(ProductSearchRequest request)
    {
        var query = _productRepository.Query()
            .Include(p => p.Category)
            .Include(p => p.ProductTags)
            .ThenInclude(pt => pt.Tag)
            .Where(p => !p.IsDeleted);

        // Apply filters
        if (!string.IsNullOrEmpty(request.SearchTerm))
        {
            query = query.Where(p => p.Name.Contains(request.SearchTerm) ||
                                   p.Description!.Contains(request.SearchTerm));
        }

        if (request.CategoryId.HasValue)
        {
            query = query.Where(p => p.CategoryId == request.CategoryId);
        }

        if (request.MinPrice.HasValue)
        {
            query = query.Where(p => p.Price >= request.MinPrice);
        }

        if (request.MaxPrice.HasValue)
        {
            query = query.Where(p => p.Price <= request.MaxPrice);
        }

        if (request.InStock)
        {
            query = query.Where(p => p.Stock > 0);
        }

        // Apply sorting
        query = request.SortBy?.ToLower() switch
        {
            "name" => request.SortDescending 
                ? query.OrderByDescending(p => p.Name)
                : query.OrderBy(p => p.Name),
            "price" => request.SortDescending
                ? query.OrderByDescending(p => p.Price)
                : query.OrderBy(p => p.Price),
            "created" => request.SortDescending
                ? query.OrderByDescending(p => p.CreatedAt)
                : query.OrderBy(p => p.CreatedAt),
            _ => query.OrderBy(p => p.Name)
        };

        return await query
            .Select(p => new ProductDto
            {
                Id = p.Id,
                Name = p.Name,
                Description = p.Description,
                Price = p.Price,
                Stock = p.Stock,
                CategoryName = p.Category.Name,
                Tags = p.ProductTags.Select(pt => pt.Tag.Name).ToList()
            })
            .ToPagedResultAsync(request.Page, request.PageSize);
    }
}
```

### Blog System

```csharp
[Entity("Posts")]
public class Post : AuditableEntity<Guid>
{
    [Required]
    [MaxLength(200)]
    public string Title { get; set; } = string.Empty;

    [Required]
    [MaxLength(250)]
    [Unique]
    public string Slug { get; set; } = string.Empty;

    [Required]
    public string Content { get; set; } = string.Empty;

    [MaxLength(500)]
    public string? Excerpt { get; set; }

    public PostStatus Status { get; set; } = PostStatus.Draft;

    public DateTime? PublishedAt { get; set; }

    public Guid AuthorId { get; set; }

    [ForeignKey(nameof(AuthorId))]
    public virtual User Author { get; set; } = null!;

    public virtual ICollection<Comment> Comments { get; set; } = new List<Comment>();
    public virtual ICollection<PostTag> PostTags { get; set; } = new List<PostTag>();

    [NotMapped]
    public bool IsPublished => Status == PostStatus.Published && PublishedAt.HasValue;

    public void Publish()
    {
        if (Status == PostStatus.Draft)
        {
            Status = PostStatus.Published;
            PublishedAt = DateTime.UtcNow;
            RaiseDomainEvent(new PostPublishedEvent(Id, Title, AuthorId));
        }
    }

    public static Post Create(string title, string content, Guid authorId)
    {
        var post = new Post
        {
            Id = Guid.NewGuid(),
            Title = title,
            Slug = GenerateSlug(title),
            Content = content,
            AuthorId = authorId
        };

        post.RaiseDomainEvent(new PostCreatedEvent(post.Id, post.Title, authorId));
        return post;
    }

    private static string GenerateSlug(string title)
    {
        return title.ToLowerInvariant()
            .Replace(" ", "-")
            .Replace("'", "")
            .Replace("\"", "")
            .Replace(".", "")
            .Replace(",", "");
    }
}

[Entity("Comments")]
public class Comment : AuditableEntity<Guid>
{
    [Required]
    [MaxLength(1000)]
    public string Content { get; set; } = string.Empty;

    public bool IsApproved { get; set; } = false;

    public Guid PostId { get; set; }
    public Guid AuthorId { get; set; }
    public Guid? ParentCommentId { get; set; }

    [ForeignKey(nameof(PostId))]
    public virtual Post Post { get; set; } = null!;

    [ForeignKey(nameof(AuthorId))]
    public virtual User Author { get; set; } = null!;

    [ForeignKey(nameof(ParentCommentId))]
    public virtual Comment? ParentComment { get; set; }

    public virtual ICollection<Comment> Replies { get; set; } = new List<Comment>();
}

public enum PostStatus
{
    Draft = 0,
    Published = 1,
    Archived = 2
}

// Blog Service
public class BlogService : IBlogService
{
    private readonly IEntityRepository<Post> _postRepository;
    private readonly IEntityRepository<Comment> _commentRepository;
    private readonly ICurrentUserService _currentUserService;

    public BlogService(
        IEntityRepository<Post> postRepository,
        IEntityRepository<Comment> commentRepository,
        ICurrentUserService currentUserService)
    {
        _postRepository = postRepository;
        _commentRepository = commentRepository;
        _currentUserService = currentUserService;
    }

    public async Task<PagedResult<PostSummaryDto>> GetPublishedPostsAsync(
        int page = 1, 
        int pageSize = 10,
        string? tag = null)
    {
        var query = _postRepository.Query()
            .Include(p => p.Author)
            .Include(p => p.PostTags)
            .ThenInclude(pt => pt.Tag)
            .Where(p => p.Status == PostStatus.Published && !p.IsDeleted);

        if (!string.IsNullOrEmpty(tag))
        {
            query = query.Where(p => p.PostTags.Any(pt => pt.Tag.Name == tag));
        }

        return await query
            .OrderByDescending(p => p.PublishedAt)
            .Select(p => new PostSummaryDto
            {
                Id = p.Id,
                Title = p.Title,
                Slug = p.Slug,
                Excerpt = p.Excerpt,
                PublishedAt = p.PublishedAt!.Value,
                AuthorName = p.Author.Name,
                CommentCount = p.Comments.Count(c => c.IsApproved),
                Tags = p.PostTags.Select(pt => pt.Tag.Name).ToList()
            })
            .ToPagedResultAsync(page, pageSize);
    }

    public async Task<PostDetailDto?> GetPostBySlugAsync(string slug)
    {
        var post = await _postRepository.Query()
            .Include(p => p.Author)
            .Include(p => p.Comments.Where(c => c.IsApproved && c.ParentCommentId == null))
            .ThenInclude(c => c.Author)
            .Include(p => p.Comments)
            .ThenInclude(c => c.Replies.Where(r => r.IsApproved))
            .ThenInclude(r => r.Author)
            .Include(p => p.PostTags)
            .ThenInclude(pt => pt.Tag)
            .FirstOrDefaultAsync(p => p.Slug == slug && 
                               p.Status == PostStatus.Published && 
                               !p.IsDeleted);

        if (post == null)
            return null;

        return new PostDetailDto
        {
            Id = post.Id,
            Title = post.Title,
            Content = post.Content,
            PublishedAt = post.PublishedAt!.Value,
            AuthorName = post.Author.Name,
            Tags = post.PostTags.Select(pt => pt.Tag.Name).ToList(),
            Comments = post.Comments
                .Where(c => c.ParentCommentId == null)
                .Select(c => new CommentDto
                {
                    Id = c.Id,
                    Content = c.Content,
                    AuthorName = c.Author.Name,
                    CreatedAt = c.CreatedAt,
                    Replies = c.Replies.Select(r => new CommentDto
                    {
                        Id = r.Id,
                        Content = r.Content,
                        AuthorName = r.Author.Name,
                        CreatedAt = r.CreatedAt
                    }).ToList()
                }).ToList()
        };
    }
}
```

## ASP.NET Core Integration

### Controller Implementation

```csharp
[ApiController]
[Route("api/[controller]")]
public class ProductsController : ControllerBase
{
    private readonly IProductService _productService;
    private readonly ILogger<ProductsController> _logger;

    public ProductsController(IProductService productService, ILogger<ProductsController> logger)
    {
        _productService = productService;
        _logger = logger;
    }

    [HttpGet]
    public async Task<ActionResult<PagedResult<ProductDto>>> GetProducts(
        [FromQuery] ProductSearchRequest request)
    {
        try
        {
            var result = await _productService.SearchProductsAsync(request);
            return Ok(result);
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Error searching products");
            return StatusCode(500, "An error occurred while searching products");
        }
    }

    [HttpGet("{id:guid}")]
    public async Task<ActionResult<ProductDto>> GetProduct(Guid id)
    {
        var product = await _productService.GetProductByIdAsync(id);
        
        if (product == null)
            return NotFound();

        return Ok(product);
    }

    [HttpPost]
    [Authorize(Roles = "Admin")]
    public async Task<ActionResult<ProductDto>> CreateProduct([FromBody] CreateProductRequest request)
    {
        if (!ModelState.IsValid)
            return BadRequest(ModelState);

        try
        {
            var product = await _productService.CreateProductAsync(request);
            return CreatedAtAction(nameof(GetProduct), new { id = product.Id }, product);
        }
        catch (ValidationException ex)
        {
            return BadRequest(ex.Message);
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Error creating product");
            return StatusCode(500, "An error occurred while creating the product");
        }
    }

    [HttpPut("{id:guid}")]
    [Authorize(Roles = "Admin")]
    public async Task<ActionResult<ProductDto>> UpdateProduct(Guid id, [FromBody] UpdateProductRequest request)
    {
        if (!ModelState.IsValid)
            return BadRequest(ModelState);

        try
        {
            var product = await _productService.UpdateProductAsync(id, request);
            return Ok(product);
        }
        catch (EntityNotFoundException)
        {
            return NotFound();
        }
        catch (ValidationException ex)
        {
            return BadRequest(ex.Message);
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Error updating product {ProductId}", id);
            return StatusCode(500, "An error occurred while updating the product");
        }
    }

    [HttpDelete("{id:guid}")]
    [Authorize(Roles = "Admin")]
    public async Task<IActionResult> DeleteProduct(Guid id)
    {
        try
        {
            await _productService.DeleteProductAsync(id);
            return NoContent();
        }
        catch (EntityNotFoundException)
        {
            return NotFound();
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Error deleting product {ProductId}", id);
            return StatusCode(500, "An error occurred while deleting the product");
        }
    }
}
```

### Dependency Injection Setup

```csharp
public static class ServiceCollectionExtensions
{
    public static IServiceCollection AddEntityCoreServices(
        this IServiceCollection services, 
        IConfiguration configuration)
    {
        // Add EntityCore
        services.AddEntityCore(options =>
        {
            var connectionString = configuration.GetConnectionString("DefaultConnection");
            options.UseSqlServer(connectionString);
            
            options.UseAuditFields();
            options.UseSoftDelete();
            
            if (Environment.GetEnvironmentVariable("ASPNETCORE_ENVIRONMENT") == "Development")
            {
                options.EnableSensitiveDataLogging();
                options.EnableDetailedErrors();
            }
        });

        // Add repositories
        services.AddScoped(typeof(IEntityRepository<>), typeof(EntityRepository<>));
        services.AddScoped<IUnitOfWork, UnitOfWork>();

        // Add business services
        services.AddScoped<IProductService, ProductService>();
        services.AddScoped<IBlogService, BlogService>();
        services.AddScoped<IUserService, UserService>();

        // Add interceptors
        services.AddScoped<IEntityInterceptor, AuditInterceptor>();
        services.AddScoped<IEntityInterceptor, ValidationInterceptor>();

        // Add domain event handlers
        services.AddMediatR(cfg => cfg.RegisterServicesFromAssembly(typeof(UserCreatedEventHandler).Assembly));

        // Add validation
        services.AddValidatorsFromAssembly(Assembly.GetExecutingAssembly());

        // Add caching
        services.AddMemoryCache();
        services.AddStackExchangeRedisCache(options =>
        {
            options.Configuration = configuration.GetConnectionString("Redis");
        });

        return services;
    }
}
```

### Health Checks

```csharp
public class EntityCoreHealthCheck : IHealthCheck
{
    private readonly IEntityRepository<User> _repository;

    public EntityCoreHealthCheck(IEntityRepository<User> repository)
    {
        _repository = repository;
    }

    public async Task<HealthCheckResult> CheckHealthAsync(
        HealthCheckContext context, 
        CancellationToken cancellationToken = default)
    {
        try
        {
            await _repository.CountAsync(cancellationToken: cancellationToken);
            return HealthCheckResult.Healthy("EntityCore database connection is healthy");
        }
        catch (Exception ex)
        {
            return HealthCheckResult.Unhealthy(
                "EntityCore database connection failed", 
                ex);
        }
    }
}

// In Startup.cs
services.AddHealthChecks()
    .AddCheck<EntityCoreHealthCheck>("entitycore");
```

## Contributing

We welcome contributions to EntityCore! Please follow these guidelines:

### Development Setup

```bash
git clone https://github.com/your-org/entitycore-dotnet.git
cd entitycore-dotnet
dotnet restore
dotnet build
```

### Running Tests

```bash
# Unit tests
dotnet test EntityCore.Tests

# Integration tests
dotnet test EntityCore.IntegrationTests

# All tests with coverage
dotnet test --collect:"XPlat Code Coverage"
```

### Coding Standards

- Follow Microsoft's C# coding conventions
- Use nullable reference types
- Write comprehensive unit tests
- Document public APIs with XML comments
- Use async/await for all I/O operations

### Pull Request Process

1. Fork the repository
2. Create a feature branch (`git checkout -b feature/amazing-feature`)
3. Commit your changes (`git commit -m 'Add amazing feature'`)
4. Push to the branch (`git push origin feature/amazing-feature`)
5. Open a Pull Request

## License

This project is licensed under the MIT License - see the [LICENSE](LICENSE) file for details.

---

**EntityCore for .NET** - Making enterprise data management simple, powerful, and type-safe.
