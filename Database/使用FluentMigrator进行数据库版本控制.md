# FluentMigrator：优雅的数据库迁移工具

## 什么是FluentMigrator？

FluentMigrator是一个开源的.NET数据库迁移框架，它允许开发者使用代码（C#或VB.NET）来管理数据库架构的变更。与传统的SQL脚本迁移方式不同，FluentMigrator提供了一种更结构化、更易维护的方式来处理数据库演进。

## 为什么选择FluentMigrator？

1. **代码优先**：迁移定义在代码中，便于版本控制
2. **跨数据库支持**：支持SQL Server、MySQL、PostgreSQL、SQLite等多种数据库
3. **事务支持**：迁移可以在事务中执行
4. **可测试性**：迁移代码可以像普通代码一样被测试
5. **依赖管理**：可以处理迁移之间的依赖关系

## FluentMigrator如何工作？

FluentMigrator通过维护一个特殊的版本表（默认名为`VersionInfo`）来跟踪哪些迁移已经应用。每次运行迁移时，它会检查哪些迁移尚未执行，并按顺序执行它们。

基本工作流程：
1. 创建迁移类（继承自`Migration`）
2. 定义`Up()`方法（向前迁移）和`Down()`方法（回滚）
3. 运行迁移工具应用变更

## 安装FluentMigrator

可以通过NuGet安装FluentMigrator：

```bash
dotnet add package FluentMigrator
dotnet add package FluentMigrator.Runner
dotnet add package FluentMigrator.Runner.SqlServer  # 如果使用SQL Server
```


## 基本使用示例

### 1. 创建第一个迁移

```csharp
[Migration(202401010001)]
public class CreateUserTable : Migration
{
    public override void Up()
    {
        Create.Table("Users")
            .WithColumn("Id").AsInt32().PrimaryKey().Identity()
            .WithColumn("Username").AsString(50).NotNullable()
            .WithColumn("Email").AsString(100).NotNullable()
            .WithColumn("CreatedAt").AsDateTime().NotNullable().WithDefaultValue(SystemMethods.CurrentDateTime);
    }

    public override void Down()
    {
        Delete.Table("Users");
    }
}
```


### 2. 添加索引和外键

```csharp
[Migration(202401010002)]
public class AddUserRelationships : Migration
{
    public override void Up()
    {
        Create.Index("IX_Users_Email")
            .OnTable("Users")
            .OnColumn("Email").Ascending()
            .WithOptions().Unique();
            
        Create.Table("Posts")
            .WithColumn("Id").AsInt32().PrimaryKey().Identity()
            .WithColumn("Title").AsString(100).NotNullable()
            .WithColumn("Content").AsString(int.MaxValue).NotNullable()
            .WithColumn("UserId").AsInt32().NotNullable()
            .WithColumn("CreatedAt").AsDateTime().NotNullable().WithDefaultValue(SystemMethods.CurrentDateTime);
            
        Create.ForeignKey("FK_Posts_Users")
            .FromTable("Posts").ForeignColumn("UserId")
            .ToTable("Users").PrimaryColumn("Id");
    }

    public override void Down()
    {
        Delete.ForeignKey("FK_Posts_Users").OnTable("Posts");
        Delete.Table("Posts");
        Delete.Index("IX_Users_Email").OnTable("Users");
    }
}
```

### 3. 数据迁移

```csharp
[Migration(202401010003)]
public class SeedInitialData : Migration
{
    public override void Up()
    {
        Insert.IntoTable("Users")
            .Row(new { Username = "admin", Email = "admin@example.com" })
            .Row(new { Username = "user1", Email = "user1@example.com" });
    }

    public override void Down()
    {
        Delete.FromTable("Users")
            .Row(new { Username = "admin" })
            .Row(new { Username = "user1" });
    }
}
```

## 运行迁移

### 命令行工具

安装命令行工具：

```bash
dotnet tool install -g FluentMigrator.DotNet.Cli
```

运行迁移：

```bash
dotnet fm migrate -p sqlserver -c "Server=.;Database=MyDb;Trusted_Connection=True;" -a "path/to/your/assembly.dll"
```

### 在代码中运行

```csharp
var serviceProvider = new ServiceCollection()
    .AddFluentMigratorCore()
    .ConfigureRunner(rb => rb
        .AddSqlServer()
        .WithGlobalConnectionString("Server=.;Database=MyDb;Trusted_Connection=True;")
        .ScanIn(typeof(CreateUserTable).Assembly).For.Migrations())
    .AddLogging(lb => lb.AddFluentMigratorConsole())
    .BuildServiceProvider(false);

using (var scope = serviceProvider.CreateScope())
{
    var runner = scope.ServiceProvider.GetRequiredService<IMigrationRunner>();
    runner.MigrateUp();
}
```

## 高级特性

### 1. 条件迁移

```csharp
IfDatabase("sqlserver").Create.Table("SqlServerSpecificTable")...
IfDatabase("postgres").Create.Table("PostgresSpecificTable")...
```

### 2. 自定义表达式

```csharp
public static class CustomMigrationExtensions
{
    public static ICreateTableColumnOptionOrWithColumnSyntax WithAuditColumns(
        this ICreateTableWithColumnSyntax table)
    {
        return table
            .WithColumn("CreatedBy").AsString().NotNullable()
            .WithColumn("CreatedAt").AsDateTime().NotNullable()
            .WithColumn("ModifiedBy").AsString().Nullable()
            .WithColumn("ModifiedAt").AsDateTime().Nullable();
    }
}

// 使用
Create.Table("Products").WithAuditColumns()...
```

### 3. 事务控制

```csharp
[Migration(202401010004, TransactionBehavior.None)]
public class NonTransactionalMigration : Migration
{
    public override void Up()
    {
        // 这个迁移不会在事务中执行
    }
    // ...
}
```

## 最佳实践

1. **有意义的版本号**：使用如`yyyyMMddHHmm`格式的版本号，便于排序和理解
2. **小而专注的迁移**：每个迁移应该只做一件事
3. **总是实现Down方法**：即使只是抛出`NotImplementedException`也比没有好
4. **测试迁移**：编写测试验证迁移的正确性
5. **环境感知**：使用不同的配置适应开发、测试和生产环境

## 与其他工具比较

| 特性            | FluentMigrator | Entity Framework Migrations | DbUp |
|----------------|---------------|----------------------------|------|
| 代码优先         | 是             | 是                          | 否   |
| SQL脚本支持      | 是             | 有限                        | 是   |
| 跨数据库支持      | 是             | 是                          | 是   |
| 回滚支持         | 是             | 是                          | 否   |
| 依赖注入支持      | 是             | 是                          | 有限 |

## 结论

FluentMigrator为.NET开发者提供了一个强大而灵活的方式来管理数据库迁移。它的流畅API使得定义迁移变得直观，而代码优先的方法则提供了更好的可维护性和可测试性。无论你是从零开始一个新项目，还是需要为现有项目引入迁移策略，FluentMigrator都是一个值得考虑的选择。

对于需要更多控制权、跨数据库支持或更复杂迁移场景的项目，FluentMigrator尤其适合。它的学习曲线相对平缓，但提供了足够的深度来处理大多数数据库演进需求。 