---
title: Dynamic Models - EF Core
author: ansvyryd

uid: core/modeling/dynamic-model
---
# Dynamic Models

The model built in `OnModelCreating` could use a property on the context to change how the model is built. For example it could be used exclude a certain property:

<!-- [!code-csharp[Main](samples/core/DynamicModel/DynamicContext.cs?range=6-31)] -->
``` csharp
public class DynamicContext : DbContext
{
    public bool? IgnoreIntProperty { get; set; }

    public DbSet<ConfigurableEntity> Entities { get; set; }

    protected override void OnConfiguring(DbContextOptionsBuilder optionsBuilder)
        => optionsBuilder
            .UseInMemoryDatabase("DynamicContext")
            .ReplaceService<IModelCacheKeyFactory, DynamicModelCacheKeyFactory>();

    protected override void OnModelCreating(ModelBuilder modelBuilder)
    {
        if (IgnoreIntProperty.HasValue)
        {
            if (IgnoreIntProperty.Value)
            {
                modelBuilder.Entity<ConfigurableEntity>().Ignore(e => e.IntProperty);
            }
            else
            {
                modelBuilder.Entity<ConfigurableEntity>().Ignore(e => e.StringProperty);
            }
        }
    }
}
```

## IModelCacheKeyFactory
However if you tried doing the above without additional changes you would get the same model every time a new context is created for any value of `IgnoreIntProperty`. This is caused by the model caching mechanism EF uses to improve the performance.

By default EF assumes that for any given context type the model will be the same. To change this you need to replace the `IModelCacheKeyFactory` service. The new implementation needs to return an object that can be compared to other model keys using the `Equals` method that takes into account all the variables that affect the model:

<!-- [!code-csharp[Main](samples/core/DynamicModel/DynamicModelCacheKeyFactory.cs?range=6-16)] -->
``` csharp
public class DynamicModelCacheKeyFactory : IModelCacheKeyFactory
{
    public object Create(DbContext context)
    {
        if (context is DynamicContext dynamicContext)
        {
            return (context.GetType(), dynamicContext.IgnoreIntProperty);
        }
        return context.GetType();
    }
}
```
