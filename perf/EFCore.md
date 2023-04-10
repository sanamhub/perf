# EF Core Performance

## TOC

### Use projections

Instead of loading entire entities with many properties that you don't need, use projections to select only the properties that you need. For example, instead of doing this:

```cs
var orders = context.Orders.ToList();
foreach(var order in orders)
{
    Console.WriteLine(order.Id + " " + order.CustomerName);
}
```

Do this:

```cs
var orders = context.Orders.Select(o => new { o.Id, o.CustomerName }).ToList();
foreach(var order in orders)
{
    Console.WriteLine(order.Id + " " + order.CustomerName);
}
```

### Use Async Methods

Use asynchronous methods for database operations to free up the calling thread and improve scalability. For example, instead of doing this:

```cs
var orders = context.Orders.ToList();
```

You can do this:

```cs
var orders = await context.Orders.ToListAsync();
```

### Use compiled queries

Use compiled queries to reduce the overhead of query compilation. For example, instead of doing this:

```cs
var orders = context.Orders.Where(o => o.CustomerName == "John").ToList();
```

You can do this:

```cs
static readonly Func<MyDbContext, string, List<Order>> getOrdersByCustomerName =
    EF.CompileQuery((MyDbContext context, string customerName) =>
        context.Orders.Where(o => o.CustomerName == customerName).ToList());

var orders = getOrdersByCustomerName(context, "John");
```

### Avoid non-cancellable queries

Make sure that your queries can be cancelled, especially for long-running queries. For example, instead of doing this:

```cs
var orders = context.Orders.ToList();
```

You can do this:

```cs
var cancellationTokenSource = new CancellationTokenSource();
cancellationTokenSource.CancelAfter(TimeSpan.FromSeconds(30));

try
{
    var orders = await context.Orders.ToListAsync(cancellationTokenSource.Token);
}
catch (OperationCanceledException ex)
{
    // Handle cancellation
}
```

### Include only necessary tables

When loading related entities, use Include to avoid generating multiple queries, but include only the necessary tables. For example, instead of doing this:

```cs
var orders = context.Orders.Include(o => o.Customer).ToList();
```

You can do this:

```cs
var orders = context.Orders.Include(o => o.Customer.Address).ToList();
```

### Select desired rows

Use Select to select only the desired rows, especially for queries that return a lot of data. For example, instead of doing this:

```cs
var orders = context.Orders.ToList();
```

You can do this:

```cs
var orderIds = context.Orders.Select(o => o.Id).ToList();
```

### For bulk processing use chunks

When processing large amounts of data, use chunks to avoid loading all the data at once. For example, instead of doing this:

```cs
var orders = context.Orders.ToList();
foreach(var order in orders)
{
    // Process order
}
```

You can do this:

```cs
const int ChunkSize = 1000;

for (int i = 0; ; i += ChunkSize)
{
    var orders = context.Orders.Skip(i).Take(ChunkSize).ToList();
    if (orders.Count == 0)
    {
        break;
    }

    foreach(var order in orders)
    {
        // Process order
    }
}
```

### Use Any method of LINQ for filters

Use the Any method instead of Count to check if any records match a condition. For example, instead of doing this:

```cs
var count = context.Orders.Count(o => o.CustomerName == "John");
if (count > 0)
{
    // ...
}
```

You can do this:

```cs
if (context.Orders.Any(o => o.CustomerName == "John"))
{
    // ...
}
```

### Use DbContextPool with db contexts

Use DbContextPool to reuse DbContext instances and reduce the overhead of creating and disposing them. For example, instead of doing this:

```cs
using (var context = new MyDbContext())
{
    var orders = context.Orders.ToList();
    // ...
}
```

You can do this:

```cs
private readonly IDbContextPool<MyDbContext> _contextPool;

public MyService(IDbContextPool<MyDbContext> contextPool)
{
    _contextPool = contextPool;
}

public void DoSomething()
{
    using (var context = _contextPool.Rent())
    {
        var orders = context.Orders.ToList();
        // ...
    }
}
```

### For multiple filters use IQueryable

Use IQueryable instead of IEnumerable for queries with multiple filters to allow the query to be composed and optimized by the database. For example, instead of doing this:

```cs
var orders = context.Orders.ToList().Where(o => o.CustomerName == "John").Where(o => o.TotalAmount > 100);
```

You can do this:

```cs
var orders = context.Orders.Where(o => o.CustomerName == "John").Where(o => o.TotalAmount > 100).ToList();
```

### Don't use raw SQL queries everywhere

Avoid using raw SQL queries everywhere and use them only when necessary. For example, instead of doing this:

```cs
var orders = context.Orders.FromSqlRaw("SELECT * FROM Orders WHERE CustomerName = 'John'").ToList();
```

You can do this:

```cs
var orders = context.Orders.Where(o => o.CustomerName == "John").ToList();
```

### Use AsNoTracking for readonly operations

Use AsNoTracking for read-only operations to avoid the overhead of tracking changes. For example, instead of doing this:

```cs
var orders = context.Orders.ToList();
```

You can do this:

```cs
var orders = context.Orders.AsNoTracking().ToList();
```

### Use Pagination, don't bring all data once

Use pagination to limit the amount of data that is returned and improve performance. For example, instead of doing this:

```cs
var orders = context.Orders.ToList();
```

You can do this:

```cs
var pageSize = 100;
var pageIndex = 0;

var orders = context.Orders.Skip(pageIndex * pageSize).Take(pageSize).ToList();
```

### Use TryGetNonEnuneratedCount for Count

Use TryGetNonEnumeratedCount to get the count of a query without loading all the data. For example, instead of doing this:

```cs
var count = context.Orders.Count();
```

You can do this:

```cs
var count = context.Orders.TryGetNonEnumeratedCount();
```

### Avoid using SQL Queries in for each loop

Avoid executing SQL queries in a loop and try to execute them in batches to reduce the number of database round trips. For example, instead of doing this:

```cs
foreach(var orderId in orderIds)
{
    var order = context.Orders.Find(orderId);
    // ...
}
```

You can do this:

```cs
var orders = context.Orders.Where(o => orderIds.Contains(o.Id)).ToList();
foreach(var order in orders)
{
    // ...
}
```

### Reduce round trips to database by calling SaveChangesAsync one time for bulk operations

Use SaveChangesAsync only once for bulk operations to reduce the number of round trips to the database. For example, instead of doing this:

```cs
foreach(var order in orders)
{
    context.Orders.Update(order);
    await context.SaveChangesAsync();
}
```

You can do this:

```cs
foreach(var order in orders)
{
    context.Orders.Update(order);
}

await context.SaveChangesAsync();
```
