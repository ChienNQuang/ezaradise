---
title: "Can we extend EF Core with Dapper?"
date: 2023-09-10T13:10:08+07:00
draft: false
author: "Chien Nguyen Quang"
language: "English"
list: false
---

EF Core is, undeniably, very convenient when it comes to its flexibility as an ORM. It comes with features that make our lives a lot easier: Eager/lazy loading, changes tracking, migrating tool out of the box. However, it has to trade a huge performance in order to achieve such convenience.

Dapper, on the other hand, being a micro-ORM, by using raw sql and does not have an additional layer in between your application and database, Dapper outperforms EF Core by a long mile.

According to [Nick Chapsas's Benchmark](https://youtu.be/Q4LtKa_HTHU?si=wwl-E3H2HAtTlC0H), Dapper's read operations can be 3x faster and can consume a whole lot less memory than EF's. Update operations in both ORMs are very close to each other in terms of speed, but EF eats a ton more memory than Dapper. With [EF's Compiled Queries](https://youtu.be/OxqAUIYemMs?si=O6BPmMczJb50f2fV), EF can get very close to Dapper, although it introduces additional memory allocations for keeping compiled queries in memory.

So to keep the best of both worlds, in this article we will try to invent a way to flexibly mix and match between using EF Core's DbSet with Linq and Dapper's methods. [Mukesh's article](https://codewithmukesh.com/blog/using-entity-framework-core-and-dapper/) already proposed a way to do it. It is a good implementation that covers most cases. But I see there are some points that can still be improved:

- In his demo project, he used 3 distinct abstractions for just data access: `IApplicationDbContext`, `IApplicationWriteDbConnection`,  `IApplicationReadDbConnection`. This although satisfies CQS pattern, but this much abstraction to just do one thing is often not desirable in a typical 3-layer architecture solution/project. One more problem with it is when you want to have a transaction going on, every query must be called in the `Write` connection interface, not its `Read` counterpart
- The developers have to manually take care of the transaction
- What if I want an update operation in Dapper, and another one using EF Core? This can be achieved with manual transaction management, but in many cases you would want all operations until you call `SaveChanges()` to be inside a single transaction. Thus transaction management could be repetitive

So after what we learned from Mukesh's implementation. We can lay out some requirement to our implementation:

- Transaction should be assumed by all update operations until we call `SaveChanges` or a variation of it
- Since we use Dapper we have to put our queries somewhere, preferrably a repository of some kind. This repository should be easy to implement
- We can use both Dapper's queries and EF's commands in the same method without breaking consistency in reads

So with these requirements in mind, how can we implement it?

Dapper provides us with methods that operate on `IDbConnection` and they can take in an `IDbTransaction`. But the problem with these 2 interfaces is that they are usually not used in EF Core. However a `DbContext` has this:

``` C#
public virtual DatabaseFacade Database { get; }
```

And `DatabaseFacade` conherently has this:

``` C#
public static IDbContextTransaction BeginTransaction(this DatabaseFacade databaseFacade, IsolationLevel isolationLevel);
```

Oh my! What in the world is an `IDbContextTransaction`!? It turns out that EF Core does not use `IDbTransaction` directly and just use their own thing. These 2 interfaces don't share anything and we cannot substitute one for another.

Good for us, EF team added this extension method:

``` C#
public static DbTransaction GetDbTransaction(this IDbContextTransaction dbContextTransaction);
```

So it finally hits, with a bit of support from DI, we can implement this very elegantly. I'll show that later, now some time on the demo project

Inspired from [Tech School's project idea](https://www.youtube.com/watch?v=rx6CPDK_5mU&list=PLy_6D98if3ULEtXtNSY_2qN21VCKgoQAE) SimpleBank, we will implement a simple version of it with these 3 main entities:

``` C#
public class Account
{
    public Guid Id { get; set; }
    public string Username { get; set; } = null!;
    public int Balance { get; set; }
}

public class Transfer
{
    public int Id { get; set; }
    public Account FromAccount { get; set; } = null!;
    public Account ToAccount { get; set; } = null!;
    public int Amount { get; set; }
    public DateTime CreatedAt { get; set; }
}

public class Entry
{
    public int Id { get; set; }
    public Account Account { get; set; } = null!;
    public int Amount { get; set; }
    public DateTime CreatedAt { get; set; }
}
```

Create a simple DbContext, instead of calling it `ApplicationDbContext`, I will use `EfContext` to clearly show its intention:

``` C#
public class EfContext : DbContext
{
    public EfContext(DbContextOptions<EfContext> options) : base(options)
    {
    }
    public DbSet<Account> Accounts => Set<Account>();
    public DbSet<Transfer> Transfers => Set<Transfer>();
    public DbSet<Entry> Entries => Set<Entry>();
}
```

Pretty simple, this `DbContext` can now be used directly in code and to do migration.

We can configure the database like this:

``` C#
var connectionString = configuration.GetConnectionString("Default");
services.AddDbContext<EfContext>(opt =>
{
    opt.UseNpgsql(connectionString)
        .UseCamelCaseNamingConvention();
});
```

I used the `UseCamelCaseNamingConvention()` method that comes from `EF.NamingConventions` to configure EF Core to create migrations with camel cased table names.

Now here comes the service registration part that'll take the most of our brains:

``` C#
services.AddScoped<DbContext>(s => s.GetRequiredService<EfContext>());

services.AddScoped<IDbConnection>(s =>
{
    var dbContext = s.GetRequiredService<DbContext>();
    var connection = dbContext.Database.GetDbConnection();
    return connection;
});

services.AddScoped<IDbTransaction>(s =>
{
    var dbContext = s.GetRequiredService<DbContext>();
    var dbContextTransaction = dbContext.Database.BeginTransaction();
    return dbContextTransaction.GetDbTransaction();
});
```

We register a `DbContext` that will return the `EfContext` that we configured above. Then we register an `IDbConnection` that get back the `DbContext` from the same service scope, and an `IDbTransaction` that get back that same `DbContext` and call `BeginTransaction` on it.

This at a whole, means that whenever we are injecting that `IDbTransaction`, then the transaction will be automatically created, and because this connection and transaction are both from the `EfContext`, all EF Core's and Dapper's operations will be assumed with a transaction because they basically are inside one single `IDbTransaction`.

However, this has a problem.

How do we use these interfaces? One requirement that we have to implement is to have some types of repositories to keep sql operations, so we can try something like this with `Account`:

``` C#
public class AccountRepository
{
    private readonly IDbConnection _connection;
    private readonly IDbTransaction _transaction;


    public AccountRepository(
        IDbConnection connection,
        IDbTransaction transaction)
    {
        _connection = connection;
        _transaction = transaction;
    }

    public async Task<Account> GetById(Guid id)
    {
        var sql = "SELECT Id, Username, Balance  " +
                    "FROM Accounts " +
                    "WHERE Id=@id";
        var result = await _connection
        .QueryFirstOrDefaultAsync<Account>(sql, new { id },
                transaction: _transaction);
        return result;
    }
}
```

Although this works well with Dapper, but we still cannot find a way to make it to work with EF. This, at first, seems very annoying. What seems to be the problem?

Well, back with Mukesh's implementation, the first thing we can improve is merge this back to just use 1 abstraction, one **unit of work** that encapsulate all database operations and that is what we will first do now:

``` C#
public class UnitOfWork : IDisposable
{
    private readonly IDbTransaction _transaction;
    private readonly DbContext _context;

    public UnitOfWork(IDbTransaction transaction, DbContext context,
        AccountRepository accounts)
    {
        // Baseline
        _transaction = transaction;
        _context = context;
        
        // Inject repositories
        Accounts = accounts;
    }

    public AccountRepository Accounts { get; }

    public async Task CommitAsync()
    {
        try
        {
            await _context.SaveChangesAsync();
            _transaction.Commit();
        }
        catch
        {
            _transaction.Rollback();
        }
    }

    public void Dispose()
    {
        // Close the SQL Connection and dispose the objects
        _transaction.Connection?.Close();
        _transaction.Connection?.Dispose();
        _transaction.Dispose();
        GC.SuppressFinalize(this);
    }
}
```

I used Unit of Work pattern because it makes the most sense when it comes to sharing a transaction across repositories.

We first have to inject the `DbContext`, `IDbTransaction` and also the `AccountRepository` which is also Scoped registered. Remember, with the transaction injected only once (in this case in `UnitOfWork`) we enable transaction by default.

With the `CommitAsync` method, the reason why I have to call both `_context.SaveChangesAsync()` and `_transaction.Commit()` is because when transaction is enabled, without any of these calls the commit will not be completed. I will talk about why in a later article.

And finally goes the `Dispose()` method of `IDisposable`

At this point, if you are wondering something like "Hey Chien, where are those EF's operations you told" then stay calm, we shall dive deep.

In EF Core, the `DbContext` itself is an implementation of Unit of Work pattern, and the `DbSet` class is an implementation of Repository, so in this case, it is best if we can combine `DbSet` with our already defined `AccountRepository` right?

The real problem with this approach is [this issue](https://github.com/dotnet/efcore/issues/12422), EF Core team does not support and does not plan to support extending from `DbSet` itself. So the only choice left is to either find a way to work around this, or to scrap it.

But thankfully, after some hours of trials and errors, I found a way to workaround it:

``` C#
public abstract class BaseRepository<TEntity> : DbSet<TEntity> where TEntity : class
{
    protected BaseRepository(EfContext context)
    {
        DbSet = context.Set<TEntity>();
    }

    private DbSet<TEntity> DbSet { get; }

    #region DbContext methods

    public override IEntityType EntityType
        => DbSet.EntityType;

    public override IAsyncEnumerable<TEntity> AsAsyncEnumerable()
        => DbSet.AsAsyncEnumerable();

    public override IQueryable<TEntity> AsQueryable()
        => DbSet.AsQueryable();

    public override LocalView<TEntity> Local
        => DbSet.Local;
    public override TEntity? Find(params object?[]? keyValues)
        => DbSet.Find(keyValues);

    public override ValueTask<TEntity?> FindAsync(params object?[]? keyValues)
        => DbSet.FindAsync(keyValues);

    public override ValueTask<TEntity?> FindAsync(object?[]? keyValues, CancellationToken cancellationToken)
        => DbSet.FindAsync(keyValues, cancellationToken);

    public override EntityEntry<TEntity> Add(TEntity entity)
        => DbSet.Add(entity);

    public override ValueTask<EntityEntry<TEntity>> AddAsync(
        TEntity entity,
        CancellationToken cancellationToken = default)
        => DbSet.AddAsync(entity, cancellationToken);

    public override EntityEntry<TEntity> Attach(TEntity entity)
        => DbSet.Attach(entity);

    public override EntityEntry<TEntity> Remove(TEntity entity)
        => DbSet.Remove(entity);

    public override EntityEntry<TEntity> Update(TEntity entity)
        => DbSet.Update(entity);
    
    public override void AddRange(params TEntity[] entities)
        => DbSet.AddRange(entities);
    
    public override void AddRange(IEnumerable<TEntity> entities)
        => DbSet.AddRange(entities);

    public override Task AddRangeAsync(params TEntity[] entities)
        => DbSet.AddRangeAsync(entities);

    public override Task AddRangeAsync(
        IEnumerable<TEntity> entities,
        CancellationToken cancellationToken = default)
        => DbSet.AddRangeAsync(entities, cancellationToken);

    public override void AttachRange(params TEntity[] entities)
        => DbSet.AttachRange(entities);

    public override void AttachRange(IEnumerable<TEntity> entities)
        => DbSet.UpdateRange(entities);

    public override void RemoveRange(params TEntity[] entities)
        => DbSet.RemoveRange(entities);

    public override void RemoveRange(IEnumerable<TEntity> entities)
        => DbSet.UpdateRange(entities);

    public override void UpdateRange(params TEntity[] entities)
        => DbSet.UpdateRange(entities);

    public override void UpdateRange(IEnumerable<TEntity> entities)
        => DbSet.UpdateRange(entities);

    public override EntityEntry<TEntity> Entry(TEntity entity)
        => DbSet.Entry(entity);

    public override IAsyncEnumerator<TEntity> GetAsyncEnumerator(CancellationToken cancellationToken = default)
        => DbSet.GetAsyncEnumerator(cancellationToken);

    #endregion
}
```

This is quite some work, but essentially what this does is overriding `DbSet` by injecting an `EfContext` and use an instance of `DbSet` this context holds. Then we just override and delegate the responsibilities to the `DbSet` that we inject. I know that this implementation is a bit loose in terms of type generic because any class can be substitute in `TEntity` and thus can cause a runtime exception. We can address this by provide a base model for entities to extend from, this make sure that all entities should be persisted by EF Core's migration and they can be type argument for `BaseRepository` without any problems.

But after all this, we can very elegantly do this:

``` C#
public class AccountRepository : BaseRepository<Account>
{
    private readonly IDbConnection _connection;
    private readonly IDbTransaction _transaction;

    public AccountRepository(
        DbContext context,
        IDbConnection connection,
        IDbTransaction transaction) : base(context)
    {
        _connection = connection;
        _transaction = transaction;
    }
    ...
}
```

The change in `AccountRepository` is minimal: we let it inherit from `BaseRepository<Account>` and inject an `DbContext` then pass it to the base constructor. With this setup the `AccountRepository` can achieve [true repository](https://martinfowler.com/eaaCatalog/repository.html) and has both the functionalities that EF Core and Dapper has to offer.

Now we do a demo of writing a `TransferService`, with one method for a transfer operation that involves all 3 tables.

``` C#
public class TransferService
{
    private readonly UnitOfWork _uow;

    public TransferService(UnitOfWork uow)
    {
        _uow = uow;
    }

    public async Task<Transfer> CreateTransfer(Guid fromAccountId, Guid toAccountId, int amount)
    {
        var fromAccount = await _uow.Accounts.GetById(fromAccountId);
        if (fromAccount is null)
        {
            throw new KeyNotFoundException(nameof(fromAccountId));
        }
        var toAccount = await _uow.Accounts.GetById(toAccountId);
        if (toAccount is null)
        {
            throw new KeyNotFoundException(nameof(toAccountId));
        }

        var transfer = new Transfer
        {
            FromAccount = fromAccount,
            ToAccount = toAccount,
            Amount = amount,
            CreatedAt = DateTime.Now,
        };
        var entries = new List<Entry>
        {
            new()
            {
                Account = fromAccount,
                Amount = amount * -1,
                CreatedAt = DateTime.Now,
            },
            new()
            {
                Account = toAccount,
                Amount = amount,
                CreatedAt = DateTime.Now,
            },
        };
        await _uow.Entries.AddRangeAsync(entries);
        fromAccount.Balance -= amount;
        toAccount.Balance += amount;
        _uow.Accounts.Update(fromAccount);
        _uow.Accounts.Update(toAccount);
        var result = await _uow.Transfers.AddAsync(transfer);

        await _uow.CommitAsync();

        return result.Entity;
    }
}
```

With our setup, the `UnitOfWork` can use both Dapper's and EF Core's operations in the same method, transaction is enabled transparently and we use it the way we would in a normal project that uses only EF Core

Instead of writing an extension method for `DbSet<>`, doing it like this enable the ability to do testing by being able to mock the methods.

## Conclusion
This implementation satisfies most of our requirements. Using 2 ORMs together enable flexibility between being feature-rich and being fast. I will try to do benchmark for it in comparison with using EF or Dapper alone.

However, some of you might have spotted the problem of this implementation.

By default, the `BeginTransaction()` call default the transaction's isolation level to ReadCommitted. The above `TransferService` will be proned to read phenomenas when only about 10 requests per second and can lead to data inconsistency. Only an isolation level of Snapshot or above can solve this inconsistency hazard.

With current DI configuration, there is no way we can dynamically choose one `IsolationLevel` at runtime, at our will that'll work without any hassle right? Or can we? I will leave it as an exercise for the readers.
 
