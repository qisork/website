---
title: How to Use Entity Framework Core
date: 2026-01-21T16:08:26+08:00
category:
  - Programming
  - ORM
tags:
  - Notes
  - Database
  - CSharp
draft: false
description: This article introduces how to use Entity Framework Core, covering core topics such as DbContext configuration and initialization, model creation and usage, and CRUD operations, suitable for beginners to quickly grasp the basic usage of EF Core.
summary: This article introduces the basic usage methods of Entity Framework Core, including configuration and initialization of DbContext, model creation and usage, as well as practical guidelines for basic CRUD operations.
---

## Introduction

This article focuses on how to use Entity Framework Core, without delving too deeply into underlying implementation details.

Entity Framework Core (shortened as EF Core) is a lightweight, cross-platform object-relational mapping (ORM) framework created by Microsoft for the .NET platform. It enables developers to manipulate various databases by writing C# object code, eliminating the need for extensive handwritten native SQL statements.

There are two major concepts that form the core content of this article:

- DbContext configuration and initialization
- Model creation and usage

### `DbContext`

DbContext is the **core bridge class connecting C# code with the database** in EF Core (requires inheriting from EF Core's `DbContext` base class to customize). All database operations including create, read, update, and delete (CRUD) must be performed through it.

Its core functions can be summarized in 3 points:

1. Managing database connections: Automatically handles connection establishment and release to the database, eliminating the need for manual connection code;
2. Providing table operation entry: Through `DbSet<T>` properties (such as `DbSet<User>`) corresponding to a database table, serving as the sole entry point for table operations;
3. Translation and execution: Automatically converts LINQ operations on C# entity classes (such as `Add` for insertion, `Where` for querying) into SQL statements and submits them to the database for execution.

> `DbSet<T>` is a **generic collection type** provided by EF Core, where the generic parameter `T` must be a defined **entity class**.

### Model

The "model" in EF Core essentially refers to the **overall description of all entity classes in the program, relationships between entities, and mapping rules between these entities and database tables/fields**.

Its core functions can be summarized in two points:

1. Serving as EF Core's **cognitive foundation**: EF Core must first understand through the model that "the User class corresponds to the Users table, the Id property corresponds to the primary key field, and one user corresponds to multiple products" before correctly translating C# operations (such as adding a user) into corresponding SQL statements;
2. Acting as a **contract between code and database**: Whether generating the database from code (Code First) or generating code from the database (Database First), the core is based on this model to align the structure of code and database.

## DbContext Configuration and Initialization

### Configuration

The core objective of configuration is to inform EF Core about: the database address/authentication information (connection string) and database type (SQL Server/MySQL/PostgreSQL, etc.). There are two commonly used configuration methods, suitable for different scenarios:

#### `OnConfiguring` Method

This is the most basic configuration method, directly overriding the `OnConfiguring` method in the custom DbContext to manually specify the connection string and database provider.

```csharp
using Microsoft.EntityFrameworkCore;

// Custom DbContext
public class AppDbContext : DbContext
{
    // Define DbSet (corresponds to database table)
    public DbSet<User> Users { get; set; }

    // Core configuration: Override OnConfiguring to specify database connection
    protected override void OnConfiguring(DbContextOptionsBuilder optionsBuilder)
    {
        // 1. Connection string, containing database address/authentication information
        string sqlServerConn = "Server=(localdb)\\mssqllocaldb;Database=MyDatabase;Trusted_Connection=True;";
        // PostgreSQL connection string example (requires installing Npgsql.EntityFrameworkCore.PostgreSQL package)
        // string pgsqlConn = @"Host=myserver;Username=mylogin;Password=mypass;Database=mydatabase";

        // 2. Specify database provider (tell EF Core which database to use)
        optionsBuilder.UseSqlServer(sqlServerConn); // SQL Server
        // optionsBuilder.UseNpgsql(mySqlConn, ServerVersion.AutoDetect(pgsqlConn)); // PostgreSQL
    }
}

// Simple entity class
public class User
{
    public int Id { get; set; }
    public string Name { get; set; }
    public int Age { get; set; }
}
```

- `DbContextOptionsBuilder`: EF Core's configuration builder, specifically used to set database-related options;
- Database providers require installation of corresponding NuGet packages (e.g., SQL Server installs `Microsoft.EntityFrameworkCore.SqlServer`);
- Connecting to different databases requires using different `ues*` methods of `DbContextOptionsBuilder`, such as using `UseSqlite` for SQLite and `UseMySql` for MySql.

#### Dependency Injection (DI) Method

Register the DbContext to the dependency injection container, allowing the container to manage configuration and lifecycle uniformly, avoiding hardcoding and manual resource management.

1. Customize `DbContext` (no need to override `OnConfiguring`)

```csharp
using Microsoft.EntityFrameworkCore;

public class AppDbContext : DbContext
{
    // Constructor receives configuration injected by DI container (core)
    public AppDbContext(DbContextOptions<AppDbContext> options) : base(options) { }

    // Define DbSet
    public DbSet<User> Users { get; set; }
}
```

2. Register `DbContext` in `Program.cs` (project entry point)

```csharp
using Microsoft.EntityFrameworkCore;

var builder = WebApplication.CreateBuilder(args);

// 1. Set the database connection string
string connStr = "Server=(localdb)\\mssqllocaldb;Database=MyDatabase;Trusted_Connection=True;";

// 2. Register DbContext to DI container
builder.Services.AddDbContext<AppDbContext>(options =>
{
    options.UseSqlServer(connStr); // Specify database provider
});

var app = builder.Build();
app.Run();
```

- `AddDbContext`: Register `DbContext` to `DI` container, which automatically manages its creation/release;
- Constructor injects `DbContextOptions<AppDbContext>`: Receives container configuration, no need to manually write `OnConfiguring`.

### Initialization

After configuration is complete, a `DbContext` instance must be created to operate the database. Corresponding to the two configuration methods, there are two initialization approaches:

#### Direct new Instance (Corresponding to OnConfiguring Configuration)

Manually create an instance, must be wrapped with `using`, ensuring database connection is released after use (to avoid connection leaks).

```csharp
// using statement: Automatically releases DbContext resources (database connection)
using (var dbContext = new AppDbContext())
{
    // Add data
    var newUser = new User { Name = "Zhang San", Age = 25 };
    dbContext.Users.Add(newUser);
    dbContext.SaveChanges(); // Submit changes

    // Query data
    var allUsers = dbContext.Users.ToList();
    Console.WriteLine($"Found {allUsers.Count} users in total");
}
```

#### DI Injection Instance (Corresponding to DI Configuration)

Receive `DbContext` instance through constructor in controllers and service classes, automatically created and injected by the DI container, no need for manual lifecycle management.

```csharp
// ASP.NET Core controller example
[ApiController]
[Route("api/[controller]")]
public class UserController : ControllerBase
{
    // Private field stores DbContext instance
    private readonly AppDbContext _dbContext;

    // Constructor injection: DI container automatically passes in AppDbContext
    public UserController(AppDbContext dbContext)
    {
        _dbContext = dbContext;
    }

    // Using DbContext in interface
    [HttpGet]
    public IActionResult GetAllUsers()
    {
        var users = _dbContext.Users.ToList();
        return Ok(users);
    }
}
```

## Model Creation

The core of the model is **entity classes** (Entity), but not just any written class can be recognized by EF Core — EF Core follows the "convention over configuration" principle. As long as entity classes meet basic rules, `DbContext` can automatically recognize them; if custom rules are needed, they can be supplemented through annotations/Fluent API configuration.

### Basic Entity Class Definition (Zero Configuration)

Simply follow EF Core's default conventions to allow `DbContext` to automatically recognize entities and map them to tables:

- **Primary Key Convention**: Property named `Id` or "entity name + Id" (such as `UserId`), EF Core automatically recognizes it as the primary key; `int/long` type primary keys are auto-increment by default.
- **Field Mapping**: C# basic types (`string/int/decimal`, etc.) automatically map to corresponding database types.
- **Relationship Convention**: Automatically identify one-to-many/many-to-one relationships between entities through "navigation properties" (such as `List<Order>` in `User`).

```csharp
// Example 1: User entity (default mapping to Users table)
public class User
{
    // Primary key: EF Core automatically recognizes Id as primary key, int type defaults to auto-increment
    public int Id { get; set; }
    // Regular field: string→database nvarchar(MAX) (default, SQL Server type)
    public string UserName { get; set; }
    // Regular field: int→database int
    public int Age { get; set; }
    // Navigation property: one-to-many relationship, pointing to Order (does not generate actual column, only used for relationship recognition)
    public List<Order> Orders { get; set; } = new List<Order>(); // Initialize to avoid null reference
}

// Example 2: Order entity (default mapping to Orders table)
public class Order
{
    public int Id { get; set; }
    public string OrderTitle { get; set; }
    public decimal Amount { get; set; }
    // Foreign key convention: "associated entity name+Id" (UserId) automatically recognized as foreign key, linking to User's Id
    public int UserId { get; set; }
    // Reverse navigation property: confirms the user to whom the order belongs
    public User User { get; set; }
}
```

### Optional: Custom Model Configuration

If default rules do not meet requirements (such as custom table names, field lengths, non-auto-increment primary keys), configuration can be done in two ways:

- **Data Annotations (Attribute)**: Directly annotate on entities/properties, simple and intuitive;
- **Fluent API**: Configure in `OnModelCreating` of DbContext, more powerful, recommended for complex scenarios.

```csharp
// Data annotation example: Custom User entity rules
using System.ComponentModel.DataAnnotations;
using System.ComponentModel.DataAnnotations.Schema;

[Table("Sys_User")] // Custom table name (no longer default Users)
public class User
{
    [Key] // Explicitly mark primary key (optional, Id already recognized by default)
    [DatabaseGenerated(DatabaseGeneratedOption.None)] // Primary key does not auto-increment
    public int Id { get; set; }

    [Column("UserName", TypeName = "nvarchar(50)")] // Custom column name+length
    [Required] // Non-null constraint
    public string UserName { get; set; }

    [Range(0, 120)] // Numeric range constraint
    public int Age { get; set; }

    public List<Order> Orders { get; set; } = new List<Order>();
}
```

### How `DbContext` Automatically Recognizes Models

`DbContext` is the core for recognizing models. It integrates scattered entity classes into a complete "model metadata" (the basis for EF Core's understanding of database structure) through "explicit declaration + implicit discovery". The process involves 3 steps:

{{% steps %}}

#### Step 1: Explicit Declaration: Register Entities Through `DbSet<T>`

The `DbSet<T>` properties declared in the custom `DbContext` serve as the **core entry point** for `DbContext` to recognize models, essentially telling `DbContext` explicitly: "These entities need to be mapped to database tables".

```csharp
using Microsoft.EntityFrameworkCore;

public class AppDbContext : DbContext
{
    // Explicit declaration: Tell DbContext to handle User and Order entities
    public DbSet<User> Users { get; set; }
    public DbSet<Order> Orders { get; set; }

    // Configure database connection
    protected override void OnConfiguring(DbContextOptionsBuilder optionsBuilder)
    {
        string connStr = "Server=(localdb)\\mssqllocaldb;Database=MyDatabase;Trusted_Connection=True;";
        optionsBuilder.UseSqlServer(connStr);
    }

    // Step 2: Fluent API supplementary configuration (optional, override/supplement default rules)
    protected override void OnModelCreating(ModelBuilder modelBuilder)
    {
        // Custom Order's OrderTitle field: length 100+non-null
        modelBuilder.Entity<Order>()
            .Property(o => o.OrderTitle)
            .HasMaxLength(100)
            .IsRequired();

        // Explicitly configure one-to-many relationship (optional, already recognized by default)
        modelBuilder.Entity<Order>()
            .HasOne(o => o.User) // One order belongs to one user
            .WithMany(u => u.Orders) // One user has multiple orders
            .HasForeignKey(o => o.UserId); // Foreign key is UserId
    }
}
```

#### Step 2: Implicit Discovery: Recognize Associated Entities Through Navigation Properties

Even if a `DbSet<T>` for a certain entity is not declared in `DbContext`, as long as it is referenced by an already declared entity through **navigation properties**, `DbContext` will automatically discover and include it in the model.

For example: If `public DbSet<Order> Orders { get; set; }` is deleted, but `List<Order> Orders` exists in `User`, `DbContext` will still recognize the `Order` entity (because it is an associated entity of `User`).

#### Step 3: Integrate Model: `OnModelCreating` Final Confirmation

`DbContext` first collects model information based on "default conventions + data annotations", then executes the `OnModelCreating` method, using Fluent API configurations to supplement/override existing rules, ultimately forming **complete model metadata** — this serves as the core basis for subsequent conversion to database tables.

{{% /steps %}}

### Converting Models to Database Tables

After `DbContext` recognizes the complete model, it does not directly create tables. Instead, the "Migration" feature must be used to convert model metadata to database DDL (table creation SQL). The core conversion rules are as follows:

#### Core Conversion Rules (Default Conventions)

| Model Element | Database Table/Field Conversion Rules |
| :---------------------: | :----------------------------------------------------------: |
| Entity class name (e.g., User) | Table name defaults to plural form of entity name (User→Users), can be customized with `[Table]`/Fluent API |
| Entity property (e.g., UserName) | Column name defaults to match property name, type automatically maps (`string`→`nvarchar (MAX)`, `int`→`int`, `decimal`→`decimal (18,2)`) |
| Primary key (Id) | Automatically set as primary key constraint, `int/long` type defaults to auto-increment (SQL Server: `IDENTITY (1,1)`) |
| Foreign key (UserId) | Automatically creates foreign key constraint, linking to main entity primary key, cascade delete enabled by default (deletes Orders when deleting User) |
| Navigation property (Orders) | Does not generate columns, only used for EF Core to recognize relationships, driving creation of foreign key constraints |

#### Generate Actual Tables Through Migration

Execute the following commands, EF Core will generate table creation SQL based on model metadata and execute it:

```shell
# 1. Create migration file: Generate table creation SQL script (stored in project Migrations folder)
Add-Migration InitialCreate

# 2. Apply migration: Execute SQL, create tables in database
Update-Database
```

Here are the tables for `User` and `Order`:
```sql
-- 1. Create Users table (corresponding to User entity)
CREATE TABLE [Users] (
    [Id] INT NOT NULL IDENTITY(1,1), -- Corresponds to User.Id: primary key+auto-increment (EF Core int primary key defaults to IDENTITY)
    [UserName] NVARCHAR(MAX) NULL,    -- Corresponds to User.UserName: string defaults to mapping nvarchar(MAX), nullable
    [Age] INT NOT NULL,               -- Corresponds to User.Age: int defaults to mapping int, non-null (EF Core value types default to non-null)
    CONSTRAINT [PK_Users] PRIMARY KEY ([Id]) -- Primary key constraint: corresponds to User.Id as primary key
);

-- 2. Create Orders table (corresponding to Order entity)
CREATE TABLE [Orders] (
    [Id] INT NOT NULL IDENTITY(1,1),          -- Corresponds to Order.Id: primary key+auto-increment
    [OrderTitle] NVARCHAR(MAX) NULL,         -- Corresponds to Order.OrderTitle: string→nvarchar(MAX), nullable
    [Amount] DECIMAL(18, 2) NOT NULL,        -- Corresponds to Order.Amount: decimal defaults to mapping decimal(18,2), non-null
    [UserId] INT NOT NULL,                   -- Corresponds to Order.UserId: foreign key field, linking to Users.Id
    CONSTRAINT [PK_Orders] PRIMARY KEY ([Id]), -- Primary key constraint
    -- Foreign key constraint: corresponds to one-to-many relationship of navigation properties (Order.User + User.Orders)
    CONSTRAINT [FK_Orders_Users_UserId] FOREIGN KEY ([UserId])
        REFERENCES [Users] ([Id])
        ON DELETE CASCADE -- Cascade delete: deletes associated Order when deleting User (EF Core default behavior)
);
```

## CRUD Operations

- **Create**: Use `Add`/`AddRange` to mark as "new status", submit with `SaveChanges()`, supports cascade creation;
- **Read**: Core is LINQ syntax + `Include` to load associated data, pagination uses `Skip/Take`, sorting uses `OrderBy`;
- **Update**: Can "query then modify" (automatically tracks status) or "no-tracking update" (high performance), no need to manually call `Update`;
- **Delete**: Use `Remove`/`RemoveRange` to mark deletion, pay attention to cascade deletion risks, always query before deleting (to avoid accidental deletion).

### Create (Add)

Mark entities as "new status" through `Add`/`AddRange`, call `SaveChanges()` to submit to database (generates `INSERT` SQL).

- Add single entity

```csharp
using (var dbContext = new AppDbContext())
{
    // 1. Create entity object
    var newUser = new User
    {
        UserName = "Zhang San",
        Age = 28
    };

    // 2. Mark as new status (DbSet<T>.Add)
    dbContext.Users.Add(newUser);

    // 3. Submit changes (EF Core generates INSERT SQL and executes)
    int affectedRows = dbContext.SaveChanges(); // Return affected rows (here is 1)
    Console.WriteLine($"Successfully added user, user ID: {newUser.Id}"); // Auto-fill primary key Id after addition
}
```

- Add associated entities: Add user and their orders simultaneously through navigation properties (EF Core automatically handles foreign key assignment)

```csharp
using (var dbContext = new AppDbContext())
{
    var newUser = new User
    {
        UserName = "Li Si",
        Age = 30,
        // Navigation property assignment: associate orders
        Orders = new List<Order>
        {
            new Order { OrderTitle = "Buy phone", Amount = 2999.99m },
            new Order { OrderTitle = "Buy headphones", Amount = 199.99m }
        }
    };

    dbContext.Users.Add(newUser);
    dbContext.SaveChanges();
    Console.WriteLine($"Added user ID: {newUser.Id}, associated orders count: {newUser.Orders.Count}");
    // Order's UserId will be automatically filled with newUser.Id, no manual assignment needed
}
```

- Batch add

```csharp
using (var dbContext = new AppDbContext())
{
    var userList = new List<User>
    {
        new User { UserName = "Wang Wu", Age = 25 },
        new User { UserName = "Zhao Liu", Age = 35 }
    };

    // AddRange: batch mark as new
    dbContext.Users.AddRange(userList);
    dbContext.SaveChanges();
    Console.WriteLine($"Batch added {userList.Count} users completed");
}
```

### Read (Query)

Query is the most flexible operation in CRUD, with the core being the use of LINQ syntax to describe query conditions, and EF Core automatically converting to `SELECT` SQL.

- Basic query

```csharp
using (var dbContext = new AppDbContext())
{
    // 1. Query all users (generate: SELECT * FROM Users)
    var allUsers = dbContext.Users.ToList();

    // 2. Query single user (by primary key, return null if not found, recommend using FirstOrDefault)
    var targetUser = dbContext.Users.FirstOrDefault(u => u.Id == 1);
    // Note: SingleOrDefault requires result to be 0 or 1 record, more will throw error; FirstOrDefault takes first record, safer

    // 3. Conditional query (users with age>25, generate: SELECT * FROM Users WHERE Age > 25)
    var adultUsers = dbContext.Users.Where(u => u.Age > 25).ToList();

    // 4. Query only specified fields (avoid SELECT *, improve performance)
    var userNames = dbContext.Users.Select(u => new { u.Id, u.UserName }).ToList();
}
```

- Associated data query: By default, EF Core does not load navigation properties (such as `User.Orders`), need to use `Include` to load explicitly.

```csharp
using (var dbContext = new AppDbContext())
{
    // 1. Load user+associated all orders (generate JOIN SQL)
    var userWithOrders = dbContext.Users
        .Include(u => u.Orders) // Key: load navigation property Orders
        .FirstOrDefault(u => u.Id == 1);
    if (userWithOrders != null)
    {
        Console.WriteLine($"User {userWithOrders.UserName} has {userWithOrders.Orders.Count} orders");
    }

    // 2. Load order+associated user (reverse navigation)
    var orderWithUser = dbContext.Orders
        .Include(o => o.User)
        .FirstOrDefault(o => o.Id == 1);
    Console.WriteLine($"Order {orderWithUser.Id} belongs to user: {orderWithUser.User.UserName}");
}
```

- Pagination + sorting query

```csharp
using (var dbContext = new AppDbContext())
{
    int pageIndex = 1; // Page number (start from 1)
    int pageSize = 10; // Records per page

    var pagedUsers = dbContext.Users
        .OrderByDescending(u => u.Age) // Sort by age descending
        .Skip((pageIndex - 1) * pageSize) // Skip previous records
        .Take(pageSize) // Take specified number
        .ToList();

    // Total count (for calculating total pages)
    int totalCount = dbContext.Users.Count();
    Console.WriteLine($"Page {pageIndex}, total {totalCount} records");
}
```

### Update

EF Core tracks entity status, after modifying entity properties and calling `SaveChanges()`, automatically generates `UPDATE` SQL.

- Basic update (query→modify→submit)

```csharp
using (var dbContext = new AppDbContext())
{
    // 1. First query entity to update (EF Core starts tracking this entity)
    var userToUpdate = dbContext.Users.FirstOrDefault(u => u.Id == 1);
    if (userToUpdate == null) return;

    // 2. Modify properties (EF Core automatically marks as "modified status")
    userToUpdate.Age = 29;
    userToUpdate.UserName = "Zhang San_Modified";

    // 3. Submit changes (no need to manually call Update, EF Core already tracks status)
    dbContext.SaveChanges();
    Console.WriteLine("User information update completed");
}
```

- No-tracking update: If entity primary key is known, can manually mark status through `Attach` to avoid extra query operations.

```csharp
using (var dbContext = new AppDbContext())
{
    // 1. Create entity with only primary key and fields to modify
    var userToUpdate = new User { Id = 1, Age = 30 };

    // 2. Attach to DbContext and mark as "modified status"
    dbContext.Users.Attach(userToUpdate);
    dbContext.Entry(userToUpdate).Property(u => u.Age).IsModified = true; // Only mark Age as modified

    // 3. Submit changes
    dbContext.SaveChanges();
}
```

- Batch update

```csharp
using (var dbContext = new AppDbContext())
{
    // 1. Query entities to batch update
    var usersToUpdate = dbContext.Users.Where(u => u.Age < 30).ToList();

    // 2. Batch modify
    foreach (var user in usersToUpdate)
    {
        user.Age += 1; // Age+1
    }

    // 3. Submit
    dbContext.SaveChanges();
}
```

### Delete

Mark entities as "delete status" through `Remove`/`RemoveRange`, call `SaveChanges()` to generate `DELETE` SQL.

- Single entity delete

```csharp
using (var dbContext = new AppDbContext())
{
    // 1. Query entity to delete
    var userToDelete = dbContext.Users.FirstOrDefault(u => u.Id == 2);
    if (userToDelete == null) return;

    // 2. Mark as delete status
    dbContext.Users.Remove(userToDelete);

    // 3. Submit changes
    dbContext.SaveChanges();
    Console.WriteLine("User deletion completed");
}
```

- Associated entity delete

```csharp
using (var dbContext = new AppDbContext())
{
    var userToDelete = dbContext.Users.Include(u => u.Orders).FirstOrDefault(u => u.Id == 1);
    if (userToDelete == null) return;

    // Delete user → cascade delete all their orders
    dbContext.Users.Remove(userToDelete);
    dbContext.SaveChanges();
    Console.WriteLine($"Deleted user {userToDelete.Id}, simultaneously deleted {userToDelete.Orders.Count} orders");
}
```

- Batch delete

```csharp
using (var dbContext = new AppDbContext())
{
    // 1. Query entities to batch delete
    var ordersToDelete = dbContext.Orders.Where(o => o.Amount < 200).ToList();

    // 2. Batch mark delete
    dbContext.Orders.RemoveRange(ordersToDelete);

    // 3. Submit
    dbContext.SaveChanges();
    Console.WriteLine($"Batch deleted {ordersToDelete.Count} orders");
}
```

## Additional Content

### In-Memory Database

The `InMemory` database provided by EF Core is a **lightweight in-memory database simulator** that does not depend on any real database engine. Data is stored only in application memory and is completely lost after program restart.

Suitable for unit testing, quick validation of models/CRUD logic, and prototype development.

1. Install `InMemory` package before use

```shell
dotnet add package Microsoft.EntityFrameworkCore.InMemory
```

2. Configure DbContext to use in-memory database

```csharp
// OnConfiguring method
protected override void OnConfiguring(DbContextOptionsBuilder optionsBuilder)
    {
        // Key: Use in-memory database, specify database name (identify unique in-memory database instance)
        optionsBuilder.UseInMemoryDatabase("MyDatabase");
    }
```

```csharp
// Dependency injection configuration method
var services = new ServiceCollection();
services.AddDbContext<AppDbContext>(options =>
{
    // Configure in-memory database, create new instance each time (avoid test cases interfering with each other)
    options.UseInMemoryDatabase("MyDatabase");
});
```

### Asynchronous Programming

Database operations (query/add/update/delete) belong to **I/O intensive operations**. Synchronous programming makes threads wait for database responses (idle during this period); while asynchronous programming allows threads to handle other requests during waiting, greatly improving the application's concurrent processing capabilities.

EF Core provides a set of **asynchronous methods (with Async suffix)** corresponding to synchronous APIs. The syntax is almost identical to synchronous operations, requiring only the use of `async/await` keywords.

Asynchronous CRUD syntax is almost identical to synchronous, only requiring replacing synchronous methods with asynchronous versions and adding `await`.

### Simple and Complete Example

 Before starting, the following two Nuget packages need to be installed:

```shell
dotnet add package Microsoft.EntityFrameworkCore
dotnet add package Microsoft.EntityFrameworkCore.InMemory
```

- Entity class: `User.cs`

```csharp
namespace ConsoleApp1;

public class User
{
    public int Id { get; set; }
    public string? Name { get; set; }
    public int Age { get; set; }
}
```

- Database context manager: `AppDbContext.cs`

```csharp
using Microsoft.EntityFrameworkCore;

namespace ConsoleApp1;

public class AppDbContext : DbContext
{
    public DbSet<User> Users { get; set;}

    protected override void OnConfiguring(DbContextOptionsBuilder optionsBuilder)
    {
        optionsBuilder.UseInMemoryDatabase("MyDatabase");
    }
}
```

- Startup class: `Program.cs`

```csharp
using ConsoleApp1;

using (var dbContext = new AppDbContext())
{
    // Add
    var newUser = new User { Name = "Zhang San", Age = 25 };
    dbContext.Users.Add(newUser);
    dbContext.SaveChanges();
    Console.WriteLine($"Added user: {newUser.Name}, age: {newUser.Age}, ID: {newUser.Id}");

    // Query
    var users = dbContext.Users.ToList();
    Console.WriteLine("Query all users:");
    foreach (var user in users)
    {
        Console.WriteLine($"ID: {user.Id}, Name: {user.Name}, Age: {user.Age}");
    }

    // Update
    var userToUpdate = dbContext.Users.FirstOrDefault(u => u.Name == "Zhang San");
    if (userToUpdate != null)
    {
        userToUpdate.Age = 26;
        dbContext.SaveChanges();
        Console.WriteLine($"Updated user age to: {userToUpdate.Age}");
    }

    // Delete
    var userToDelete = dbContext.Users.FirstOrDefault(u => u.Name == "Zhang San");
    if (userToDelete != null)
    {
        dbContext.Users.Remove(userToDelete);
        dbContext.SaveChanges();
        Console.WriteLine($"Deleted user: {userToDelete.Name}");
    }
}
```

## Reference Links

- [Entity Framework Core Overview - EF Core | Microsoft Learn](https://learn.microsoft.com/ef/core/)
- [DbContext Lifetime, Configuration and Initialization - EF Core | Microsoft Learn](https://learn.microsoft.com/ef/core/dbcontext-configuration/)
- [Creating and Configuring Models - EF Core | Microsoft Learn](https://learn.microsoft.com/ef/core/modeling/)
- [Asynchronous Programming - EF Core | Microsoft Learn](https://learn.microsoft.com/ef/core/miscellaneous/async)
- [Database Providers - EF Core | Microsoft Learn](https://learn.microsoft.com/ef/core/providers/?tabs=dotnet-core-cli)
