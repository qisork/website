---
title: 如何使用 Entity Framework Core
date: 2026-01-21T16:08:18+08:00
category: 
  - 编程
  - ORM
tags:
  - 笔记
  - 数据库
  - csharp
draft: false
description: 本文介绍如何使用 Entity Framework Core，涵盖 DbContext 配置和初始化、模型创建与使用、CRUD 操作等核心知识点，适合初学者快速掌握 EF Core 基础用法。
summary: 本文介绍了 Entity Framework Core 的基本使用方法，包括 DbContext 的配置与初始化、模型的创建与使用，以及基本的 CRUD 操作实践指南。
---

## 前言

在本篇文章中，着重介绍如何使用 Entity Framework Core，不过多涉及底层细节。

Entity Framework Core（简称 EF Core）是微软为.NET 平台打造的轻量级、跨平台对象关系映射（ORM）框架，以编写 C# 对象代码的方式操作各类数据库，无需大量手写原生 SQL 语句。

其中有两大概念，也是本文要讲述的内容：

- DbContext 配置和初始化
- 模型的创建与使用

### `DbContext`

`DbContext` 是 EF Core 中**连接 C# 代码与数据库的核心桥梁类**（需要继承 EF Core 的 `DbContext` 基类自定义），所有对数据库的增删改查操作，都必须通过它来完成。

它的核心作用可以概括为 3 点：

1. 管理数据库连接：自动处理和数据库的连接建立、释放，无需你手动写连接代码；
2. 提供表操作入口：通过 `DbSet<T>` 属性（比如 `DbSet<User>`）对应数据库的一张表，是操作表的唯一入口；
3. 翻译与执行：把对 C# 实体类的 LINQ 操作（比如 `Add` 新增、`Where` 查询）自动转换成 SQL 语句，提交到数据库执行。

> `DbSet<T>` 是 EF Core 提供的**泛型集合类型**，泛型参数 `T` 必须是定义的**实体类**。

### 模型

EF Core 里的 “模型”，本质是**程序中所有实体类、实体间关系，以及这些实体与数据库表 / 字段映射规则的整体描述**。

它的核心作用可以概括为两点：

1. 是 EF Core 的**认知基础**：EF Core 只有先通过模型知道 “User 类对应 Users 表、Id 属性对应主键字段、一个用户对应多个商品”，才能正确把 C# 操作（比如新增用户）翻译成对应的 SQL 语句；
2. 是**代码与数据库的契约**：不管是从代码生成数据库（Code First），还是从数据库生成代码（Database First），核心都是基于这个模型来对齐代码和数据库的结构。

## DbContext 配置和初始化

### 配置

配置的核心目标是让 EF Core 知道：要连接的数据库地址 / 认证信息（连接字符串）、数据库类型（SQL Server/MySQL/PostgreSQL 等）。有两种常用配置方式，适配不同场景：

#### `OnConfiguring` 方法

这是最基础的配置方式，直接在自定义 DbContext 中重写 `OnConfiguring` 方法，手动指定连接字符串和数据库提供器。

```csharp
using Microsoft.EntityFrameworkCore;

// 自定义 DbContext
public class AppDbContext : DbContext
{
    // 定义 DbSet（对应数据库表）
    public DbSet<User> Users { get; set; }

    // 核心配置：重写 OnConfiguring 指定数据库连接
    protected override void OnConfiguring(DbContextOptionsBuilder optionsBuilder)
    {
        // 1. 连接字符串，包含数据库地址/认证信息
        string sqlServerConn = "Server=(localdb)\\mssqllocaldb;Database=MyDatabase;Trusted_Connection=True;";
        // PostgreSQL 连接字符串示例（需安装 Npgsql.EntityFrameworkCore.PostgreSQL 包）
        // string pgsqlConn = @"Host=myserver;Username=mylogin;Password=mypass;Database=mydatabase";

        // 2. 指定数据库提供器（告诉 EF Core 用哪种数据库）
        optionsBuilder.UseSqlServer(sqlServerConn); // SQL Server
        // optionsBuilder.UseNpgsql(mySqlConn, ServerVersion.AutoDetect(pgsqlConn)); // PostgreSQL
    }
}

// 简单实体类
public class User
{
    public int Id { get; set; }
    public string Name { get; set; }
    public int Age { get; set; }
}
```

- `DbContextOptionsBuilder`：EF Core 的配置构建器，专门用来设置数据库相关选项；
- 数据库提供器需安装对应 NuGet 包（如 SQL Server 安装 `Microsoft.EntityFrameworkCore.SqlServer`）；
- 链接不同的数据库，需要使用 `DbContextOptionsBuilder` 不同的 `ues*` 方法，比如使用 SQLite 使用 `UseSqlite`、MySql 使用 `UseMySql`。

#### 依赖注入（DI）方法

将 DbContext 注册到依赖注入容器，由容器统一管理配置和生命周期，避免硬编码和手动管理资源。

1. 自定义 `DbContext`（无需重写 `OnConfiguring`）

```csharp
using Microsoft.EntityFrameworkCore;

public class AppDbContext : DbContext
{
    // 构造函数接收 DI 容器传入的配置（核心）
    public AppDbContext(DbContextOptions<AppDbContext> options) : base(options) { }

    // 定义DbSet
    public DbSet<User> Users { get; set; }
}
```

2. 在 `Program.cs` 中注册 `DbContext`（项目入口）

```csharp
using Microsoft.EntityFrameworkCore;

var builder = WebApplication.CreateBuilder(args);

// 1. 设置数据库连接字符串
string connStr = "Server=(localdb)\\mssqllocaldb;Database=MyDatabase;Trusted_Connection=True;";

// 2. 注册 DbContext 到 DI 容器
builder.Services.AddDbContext<AppDbContext>(options =>
{
    options.UseSqlServer(connStr); // 指定数据库提供器
});

var app = builder.Build();
app.Run();
```

- `AddDbContext`：将 `DbContext` 注册到 `DI` 容器，容器自动管理其创建 / 释放；
- 构造函数注入 `DbContextOptions<AppDbContext>`：接收容器的配置，无需手动写 `OnConfiguring`。

### 初始化

配置完成后，需创建 `DbContext` 实例才能操作数据库，对应两种配置方式有两种初始化方式：

#### 直接 new 实例（对应 OnConfiguring 配置）

手动创建实例，必须用 `using` 包裹，确保使用后释放数据库连接（避免连接泄露）。

```csharp
// using语句：自动释放 DbContext 资源（数据库连接）
using (var dbContext = new AppDbContext())
{
    // 新增数据
    var newUser = new User { Name = "张三", Age = 25 };
    dbContext.Users.Add(newUser);
    dbContext.SaveChanges(); // 提交更改

    // 查询数据
    var allUsers = dbContext.Users.ToList();
    Console.WriteLine($"共查询到 {allUsers.Count} 个用户");
}
```

#### DI 注入实例（对应 DI 配置）

在控制器、服务类中通过构造函数接收 `DbContext` 实例，由 DI 容器自动创建和注入，无需手动管理生命周期。

```csharp
// ASP.NET Core 控制器示例
[ApiController]
[Route("api/[controller]")]
public class UserController : ControllerBase
{
    // 私有字段存储 DbContext 实例
    private readonly AppDbContext _dbContext;

    // 构造函数注入：DI 容器自动传入 AppDbContext
    public UserController(AppDbContext dbContext)
    {
        _dbContext = dbContext;
    }

    // 接口中使用 DbContext
    [HttpGet]
    public IActionResult GetAllUsers()
    {
        var users = _dbContext.Users.ToList();
        return Ok(users);
    }
}
```

## 模型创建

模型的核心是**实体类**（Entity），但不是随便写的类都能被 EF Core 识别 ——EF Core 遵循 “约定优于配置” 原则，只要实体类满足基础规则，`DbContext` 就能自动识别；若需自定义规则，也可通过注解 / Fluent API 补充配置。

### 基础实体类定义（零配置）

只需遵循 EF Core 的默认约定，就能让 `DbContext` 自动识别实体并映射为表：

- **主键约定**：属性名为 `Id` 或 “实体名 + Id”（如 `UserId`），EF Core 自动识别为主键；`int/long` 类型主键默认自增。
- **字段映射**：C# 基础类型（`string/int/decimal` 等）自动映射为数据库对应类型。
- **关系约定**：通过 “导航属性”（如 `User` 中的 `List<Order>`）自动识别实体间的一对多 / 多对一关系。

```csharp
// 示例1：用户实体（默认映射到 Users 表）
public class User
{
    // 主键：EF Core 自动识别 Id 为主键，int 类型默认自增
    public int Id { get; set; }
    // 普通字段：string → 数据库 nvarchar(MAX)（默认, SQL Server 的类型）
    public string UserName { get; set; }
    // 普通字段：int→数据库int
    public int Age { get; set; }
    // 导航属性：一对多关系，指向 Order（不生成实际列，仅用于识别关系）
    public List<Order> Orders { get; set; } = new List<Order>(); // 初始化避免空引用
}

// 示例2：订单实体（默认映射到Orders表）
public class Order
{
    public int Id { get; set; }
    public string OrderTitle { get; set; }
    public decimal Amount { get; set; }
    // 外键约定：“关联实体名+Id”（UserId）自动识别为外键，关联 User 的 Id
    public int UserId { get; set; }
    // 反向导航属性：确认订单归属的用户
    public User User { get; set; }
}
```

### 可选：自定义模型配置

若默认规则不满足需求（如自定义表名、字段长度、主键不自增），可通过两种方式配置：

- **数据注解（Attribute）**：直接标注在实体 / 属性上，简单直观；
- **Fluent API**：在 DbContext 的 `OnModelCreating` 中配置，功能更强大，推荐复杂场景。

```csharp
// 数据注解示例：自定义User实体规则
using System.ComponentModel.DataAnnotations;
using System.ComponentModel.DataAnnotations.Schema;

[Table("Sys_User")] // 自定义表名（不再默认是Users）
public class User
{
    [Key] // 显式标记主键（可选，默认Id已识别）
    [DatabaseGenerated(DatabaseGeneratedOption.None)] // 主键不自增
    public int Id { get; set; }

    [Column("UserName", TypeName = "nvarchar(50)")] // 自定义列名+长度
    [Required] // 非空约束
    public string UserName { get; set; }

    [Range(0, 120)] // 数值范围约束
    public int Age { get; set; }

    public List<Order> Orders { get; set; } = new List<Order>();
}
```

### `DbContext` 如何自动识别模型

`DbContext` 是识别模型的核心，它通过 “显性声明 + 隐性发现” 的方式，把分散的实体类整合为完整的 “模型元数据”（EF Core 认知数据库结构的依据），过程分 3 步：

{{% steps %}}

#### 第一步：显性声明：通过 `DbSet<T>` “注册” 实体

在自定义 `DbContext` 中声明的 `DbSet<T>` 属性，是 `DbContext` 识别模型的**核心入口**，相当于明确告诉 `DbContext`：“这些实体需要映射到数据库表”。

```csharp
using Microsoft.EntityFrameworkCore;

public class AppDbContext : DbContext
{
    // 显性声明：告诉 DbContext 要处理 User 和 Order 实体
    public DbSet<User> Users { get; set; }
    public DbSet<Order> Orders { get; set; }

    // 配置数据库连接
    protected override void OnConfiguring(DbContextOptionsBuilder optionsBuilder)
    {
        string connStr = "Server=(localdb)\\mssqllocaldb;Database=MyDatabase;Trusted_Connection=True;";
        optionsBuilder.UseSqlServer(connStr);
    }

    // 第二步：Fluent API 补充配置（可选，覆盖/补充默认规则）
    protected override void OnModelCreating(ModelBuilder modelBuilder)
    {
        // 自定义Order的OrderTitle字段：长度100+非空
        modelBuilder.Entity<Order>()
            .Property(o => o.OrderTitle)
            .HasMaxLength(100)
            .IsRequired();

        // 显式配置一对多关系（可选，默认已识别）
        modelBuilder.Entity<Order>()
            .HasOne(o => o.User) // 一个订单属于一个用户
            .WithMany(u => u.Orders) // 一个用户有多个订单
            .HasForeignKey(o => o.UserId); // 外键是UserId
    }
}
```

#### 第二步：隐性发现：通过导航属性识别关联实体

即使没在 `DbContext` 中声明某个实体的 `DbSet<T>`，只要它被已声明的实体通过**导航属性**引用，`DbContext` 也会自动发现并纳入模型。

比如：若删除 `public DbSet<Order> Orders { get; set; }`，但 `User` 中有 `List<Order> Orders`，`DbContext` 依然会识别 `Order` 实体（因为它是 `User` 的关联实体）。

#### 第三步：整合模型：`OnModelCreating` 最终确认

`DbContext` 会先基于 “默认约定 + 数据注解” 收集模型信息，再执行 `OnModelCreating` 方法，用 Fluent API 的配置补充 / 覆盖原有规则，最终形成**完整的模型元数据**—— 这是后续转换为数据库表的核心依据。

{{% /steps %}}

### 将模型转换为数据库表

`DbContext` 识别出完整模型后，不会直接创建表，需通过 “迁移（Migration）” 功能将模型元数据转换为数据库的 DDL（建表 SQL），核心转换规则如下：

#### 核心转换规则（默认约定）

|       模型元素       |                                    数据库表 / 字段转换规则                                    |
| :--------------: | :---------------------------------------------------------------------------------: |
|   实体类名（如 User）   |               表名默认是实体名的复数形式（User→Users），可通过 `[Table]`/Fluent API 自定义                |
| 实体属性（如 UserName） | 列名默认和属性名一致，类型自动映射（`string`→`nvarchar (MAX)`、`int`→`int`、`decimal`→`decimal (18,2)`） |
|      主键（Id）      |               自动设为主键约束，`int/long` 类型默认自增（SQL Server：`IDENTITY (1,1)`）               |
|    外键（UserId）    |                  自动创建外键约束，关联主实体主键，默认开启级联删除（删除 User 时自动删除其 Orders）                   |
|   导航属性（Orders）   |                           不生成列，仅用于 EF Core 识别关系，驱动外键约束的创建                           |

#### 通过迁移生成实际表

执行以下命令，EF Core 会根据模型元数据生成建表 SQL 并执行：

```shell
# 1. 创建迁移文件：生成建表 SQL 脚本（存于项目 Migrations 文件夹）
Add-Migration InitialCreate

# 2. 应用迁移：执行 SQL，在数据库中创建表
Update-Database
```

以下是 `User` 和 `Order` 的表：
```sql
-- 1. 创建Users表（对应User实体）
CREATE TABLE [Users] (
    [Id] INT NOT NULL IDENTITY(1,1), -- 对应User.Id：主键+自增（EF Core int主键默认IDENTITY）
    [UserName] NVARCHAR(MAX) NULL,    -- 对应User.UserName：string默认映射nvarchar(MAX)，可空
    [Age] INT NOT NULL,               -- 对应User.Age：int默认映射int，非空（EF Core值类型默认非空）
    CONSTRAINT [PK_Users] PRIMARY KEY ([Id]) -- 主键约束：对应User.Id为主键
);

-- 2. 创建Orders表（对应Order实体）
CREATE TABLE [Orders] (
    [Id] INT NOT NULL IDENTITY(1,1),          -- 对应Order.Id：主键+自增
    [OrderTitle] NVARCHAR(MAX) NULL,         -- 对应Order.OrderTitle：string→nvarchar(MAX)，可空
    [Amount] DECIMAL(18, 2) NOT NULL,        -- 对应Order.Amount：decimal默认映射decimal(18,2)，非空
    [UserId] INT NOT NULL,                   -- 对应Order.UserId：外键字段，关联Users.Id
    CONSTRAINT [PK_Orders] PRIMARY KEY ([Id]), -- 主键约束
    -- 外键约束：对应导航属性的一对多关系（Order.User + User.Orders）
    CONSTRAINT [FK_Orders_Users_UserId] FOREIGN KEY ([UserId]) 
        REFERENCES [Users] ([Id]) 
        ON DELETE CASCADE -- 级联删除：删除User时自动删除其关联的Order（EF Core默认行为）
);
```

## CRUD 操作

- **新增**：用 `Add`/`AddRange` 标记新增，`SaveChanges()` 提交，支持级联新增；
- **查询**：核心是 LINQ 语法 +`Include` 加载关联数据，分页用 `Skip/Take`，排序用 `OrderBy`；
- **更新**：可 “查询后修改”（自动跟踪状态）或 “无跟踪更新”（高性能），无需手动调用 `Update`；
- **删除**：用 `Remove`/`RemoveRange` 标记删除，注意级联删除的风险，务必先查询再删除（避免误删）。

### Create（新增）

通过 `Add`/`AddRange` 标记实体为 “新增状态”，调用 `SaveChanges()` 提交到数据库（生成 `INSERT` SQL）。

- 新增单个实体

```csharp
using (var dbContext = new AppDbContext())
{
    // 1. 创建实体对象
    var newUser = new User
    {
        UserName = "张三",
        Age = 28
    };

    // 2. 标记为新增状态（DbSet<T>.Add）
    dbContext.Users.Add(newUser);

    // 3. 提交更改（EF Core 生成 INSERT SQL 并执行）
    int affectedRows = dbContext.SaveChanges(); // 返回受影响的行数（此处为 1）
    Console.WriteLine($"新增用户成功，用户ID：{newUser.Id}"); // 新增后自动填充主键 Id
}
```

- 新增关联实体：通过导航属性同时新增用户和其订单（EF Core 自动处理外键赋值）

```csharp
using (var dbContext = new AppDbContext())
{
    var newUser = new User
    {
        UserName = "李四",
        Age = 30,
        // 导航属性赋值：关联订单
        Orders = new List<Order>
        {
            new Order { OrderTitle = "购买手机", Amount = 2999.99m },
            new Order { OrderTitle = "购买耳机", Amount = 199.99m }
        }
    };

    dbContext.Users.Add(newUser);
    dbContext.SaveChanges();
    Console.WriteLine($"新增用户ID：{newUser.Id}，关联订单数：{newUser.Orders.Count}");
    // 订单的UserId会自动填充为 newUser.Id，无需手动赋值
}
```

- 批量增加

```csharp
using (var dbContext = new AppDbContext())
{
    var userList = new List<User>
    {
        new User { UserName = "王五", Age = 25 },
        new User { UserName = "赵六", Age = 35 }
    };

    // AddRange：批量标记新增
    dbContext.Users.AddRange(userList);
    dbContext.SaveChanges();
    Console.WriteLine($"批量新增 {userList.Count} 个用户完成");
}
```

### Read（查询）

查询是 CRUD 中最灵活的操作，核心是通过 LINQ 语法描述查询条件，EF Core 自动转换为 `SELECT` SQL。

- 基础查询

```csharp
using (var dbContext = new AppDbContext())
{
    // 1. 查询所有用户（生成：SELECT * FROM Users）
    var allUsers = dbContext.Users.ToList();

    // 2. 查询单个用户（按主键，找不到返回 null，推荐用 FirstOrDefault）
    var targetUser = dbContext.Users.FirstOrDefault(u => u.Id == 1);
    // 注意：SingleOrDefault要求结果只能是0或1条，多了会报错；FirstOrDefault取第一条，更安全

    // 3. 条件查询（年龄>25的用户，生成：SELECT * FROM Users WHERE Age > 25）
    var adultUsers = dbContext.Users.Where(u => u.Age > 25).ToList();

    // 4. 只查询指定字段（避免SELECT *，提升性能）
    var userNames = dbContext.Users.Select(u => new { u.Id, u.UserName }).ToList();
}
```

- 关联数据查询：默认情况下 EF Core 不会加载导航属性（如 `User.Orders`），需用 `Include` 显式加载。

```csharp
using (var dbContext = new AppDbContext())
{
    // 1. 加载用户+关联的所有订单（生成 JOIN SQL）
    var userWithOrders = dbContext.Users
        .Include(u => u.Orders) // 关键：加载导航属性 Orders
        .FirstOrDefault(u => u.Id == 1);
    if (userWithOrders != null)
    {
        Console.WriteLine($"用户{userWithOrders.UserName}有{userWithOrders.Orders.Count}个订单");
    }

    // 2. 加载订单+关联的用户（反向导航）
    var orderWithUser = dbContext.Orders
        .Include(o => o.User)
        .FirstOrDefault(o => o.Id == 1);
    Console.WriteLine($"订单{orderWithUser.Id}所属用户：{orderWithUser.User.UserName}");
}
```

- 分页 + 排序查询

```csharp
using (var dbContext = new AppDbContext())
{
    int pageIndex = 1; // 页码（从 1 开始）
    int pageSize = 10; // 每页条数

    var pagedUsers = dbContext.Users
        .OrderByDescending(u => u.Age) // 按年龄降序排序
        .Skip((pageIndex - 1) * pageSize) // 跳过前面的记录
        .Take(pageSize) // 取指定条数
        .ToList();

    // 总条数（用于计算总页数）
    int totalCount = dbContext.Users.Count();
    Console.WriteLine($"第{pageIndex}页，共{totalCount}条数据");
}
```

### Update（更新）

EF Core 跟踪实体状态，修改实体属性后调用 `SaveChanges()`，自动生成 `UPDATE` SQL。

- 基础更新（查询→修改→提交）

```csharp
using (var dbContext = new AppDbContext())
{
    // 1. 先查询要更新的实体（EF Core 开始跟踪该实体）
    var userToUpdate = dbContext.Users.FirstOrDefault(u => u.Id == 1);
    if (userToUpdate == null) return;

    // 2. 修改属性（EF Core自动标记为“修改状态”）
    userToUpdate.Age = 29;
    userToUpdate.UserName = "张三_修改后";

    // 3. 提交更改（无需手动调用 Update，EF Core 已跟踪状态）
    dbContext.SaveChanges();
    Console.WriteLine("用户信息更新完成");
}
```

- 无跟踪更新：如果已知实体主键，可通过 `Attach` 手动标记状态，避免额外的查询操作。

```csharp
using (var dbContext = new AppDbContext())
{
    // 1. 创建仅含主键和要修改字段的实体
    var userToUpdate = new User { Id = 1, Age = 30 };

    // 2. 附加到 DbContext 并标记为“修改状态”
    dbContext.Users.Attach(userToUpdate);
    dbContext.Entry(userToUpdate).Property(u => u.Age).IsModified = true; // 仅标记 Age 为修改

    // 3. 提交更改
    dbContext.SaveChanges();
}
```

- 批量更新

```csharp
using (var dbContext = new AppDbContext())
{
    // 1. 查询要批量更新的实体
    var usersToUpdate = dbContext.Users.Where(u => u.Age < 30).ToList();

    // 2. 批量修改
    foreach (var user in usersToUpdate)
    {
        user.Age += 1; // 年龄+1
    }

    // 3. 提交
    dbContext.SaveChanges();
}
```

### Delete（删除）

通过 `Remove`/`RemoveRange` 标记实体为 “删除状态”，调用 `SaveChanges()` 生成 `DELETE` SQL。

- 单个实体删除

```csharp
using (var dbContext = new AppDbContext())
{
    // 1. 查询要删除的实体
    var userToDelete = dbContext.Users.FirstOrDefault(u => u.Id == 2);
    if (userToDelete == null) return;

    // 2. 标记为删除状态
    dbContext.Users.Remove(userToDelete);

    // 3. 提交更改
    dbContext.SaveChanges();
    Console.WriteLine("用户删除完成");
}
```

- 关联实体删除

```csharp
using (var dbContext = new AppDbContext())
{
    var userToDelete = dbContext.Users.Include(u => u.Orders).FirstOrDefault(u => u.Id == 1);
    if (userToDelete == null) return;

    // 删除用户 → 级联删除其所有订单
    dbContext.Users.Remove(userToDelete);
    dbContext.SaveChanges();
    Console.WriteLine($"删除用户{userToDelete.Id}，同时删除{userToDelete.Orders.Count}个订单");
}
```

- 批量删除

```csharp
using (var dbContext = new AppDbContext())
{
    // 1. 查询要批量删除的实体
    var ordersToDelete = dbContext.Orders.Where(o => o.Amount < 200).ToList();

    // 2. 批量标记删除
    dbContext.Orders.RemoveRange(ordersToDelete);

    // 3. 提交
    dbContext.SaveChanges();
    Console.WriteLine($"批量删除{ordersToDelete.Count}个订单");
}
```

## 额外内容

### 在内存中的数据库

EF Core 提供的 `InMemory` 数据库是一个**轻量级的内存数据库模拟器**，它不依赖任何真实数据库引擎，数据仅存储在应用内存中，程序重启后数据全部丢失。

适合于单元测试、快速验证模型 / CRUD 逻辑、原型开发。

1. 使用前需先安装 `InMemory` 包

```shell
dotnet add package Microsoft.EntityFrameworkCore.InMemory
```

2. 配置 DbContext 使用内存数据库

```csharp
// OnConfiguring 方法
protected override void OnConfiguring(DbContextOptionsBuilder optionsBuilder)
    {
        // 关键：使用内存数据库，指定数据库名称（标识唯一的内存数据库实例）
        optionsBuilder.UseInMemoryDatabase("MyDatabase");
    }
```

```csharp
// 依赖注入配置方法
var services = new ServiceCollection();
services.AddDbContext<AppDbContext>(options =>
{
    // 配置内存数据库，每次创建新实例（避免测试用例互相干扰）
    options.UseInMemoryDatabase("MyDatabase");
});
```

### 异步编程

数据库操作（查询 / 新增 / 更新 / 删除）属于 **IO 密集型操作**，同步编程会让线程等待数据库响应（期间线程闲置）；而异步编程能让线程在等待时去处理其他请求，大幅提升应用的并发处理能力。

EF Core 提供了一套和同步 API 对应的**异步方法（后缀为 Async）**，语法和同步操作几乎一致，只需配合 `async/await` 关键字使用。

异步 CRUD 语法和同步几乎一致，仅需将同步方法替换为异步版本并添加 `await`。

### 简单且完整的案例

 在开始之前，需要安装如下两个 Nuget 包：

```shell
dotnet add package Microsoft.EntityFrameworkCore
dotnet add package Microsoft.EntityFrameworkCore.InMemory
```

- 实体类：`User.cs`

```csharp
namespace ConsoleApp1;

public class User
{
    public int Id { get; set; }
    public string? Name { get; set; }
    public int Age { get; set; }
}
```

- 数据库上下文管理器：`AppDbContext.cs`

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

- 启动类：`Program.cs`

```csharp
using ConsoleApp1;

using (var dbContext = new AppDbContext())
{
    // 新增
    var newUser = new User { Name = "张三", Age = 25 };
    dbContext.Users.Add(newUser);
    dbContext.SaveChanges();
    Console.WriteLine($"新增用户: {newUser.Name}, 年龄: {newUser.Age}, ID: {newUser.Id}");

    // 查询
    var users = dbContext.Users.ToList();
    Console.WriteLine("查询所有用户:");
    foreach (var user in users)
    {
        Console.WriteLine($"ID: {user.Id}, 姓名: {user.Name}, 年龄: {user.Age}");
    }

    // 更新
    var userToUpdate = dbContext.Users.FirstOrDefault(u => u.Name == "张三");
    if (userToUpdate != null)
    {
        userToUpdate.Age = 26;
        dbContext.SaveChanges();
        Console.WriteLine($"更新用户年龄为: {userToUpdate.Age}");
    }

    // 删除
    var userToDelete = dbContext.Users.FirstOrDefault(u => u.Name == "张三");
    if (userToDelete != null)
    {
        dbContext.Users.Remove(userToDelete);
        dbContext.SaveChanges();
        Console.WriteLine($"删除用户: {userToDelete.Name}");
    }
}
```

## 参考链接

- [Entity Framework Core 概述 - EF Core | Microsoft Learn](https://learn.microsoft.com/ef/core/)
- [DbContext 生存期、配置和初始化 - EF Core | Microsoft Learn](https://learn.microsoft.com/ef/core/dbcontext-configuration/)
- [创建和配置模型 - EF Core | Microsoft Learn](https://learn.microsoft.com/ef/core/modeling/)
- [异步编程 - EF Core | Microsoft Learn](https://learn.microsoft.com/ef/core/miscellaneous/async)
- [数据库提供程序 - EF Core | Microsoft Learn](https://learn.microsoft.com/ef/core/providers/?tabs=dotnet-core-cli)