# ShardCore
ShardCore is a library that provides horizontal sharding functionality based on EntityFramework Core. 
It allows developers to abstract from the complexity of storing and retrieving information to or from multiple database shards using a normal repository class.

## Glossary
To better understand how to use ShardCore we start by defining a few terms.

**GenericRepository -** This is your standard generic entity repository.
If you wanted to store, let's say, products information we would create a ProductsRepository class that inherits from a GenericRepository\<T\> class, where T is of type Product.

**ShardInformationRepository -** This is a repository that ShardCore will use to store and retrieve information about the shards, like database connections, which shards are enabled for read and write operations, etc.

**GenericShardedRepository -** This is the repository that a developer will use to access multiple shards transparently. 
This repository will make use of the previously mentioned **ShardInformationRepository** and **GenericRepository**, to resolve the correct shard, and store/retrieve the information respectively. Using the previous example, if you wanted to store products information in multiple database shards you would create a ProductsShardedRepository class that inherits from a GenericShardedRepository\<T,R\> class, where T is of type Product and R is of type GenericRepository\<T\>.
  
## Installation
You can install the latest ShardCore version by running the following command in the Package Manager Console:

```Install-Package Particle.ShardCore```

## Setup
Using ShardCore requires taking the following steps:
1. Add the entity that will be sharded. That entity must extend ShardCore's ShardedEntity class.
2. Add a database mapping for that entity.
3. Add a ShardDbContext class that will be used to access each of the shards repositories, this class must extend Entity Framework's DbContext class.
4. Add a ShardUnitOfWork that will be used by the shards repositories, this class must extend ShardCore's UnitOfWork class.
5. Add a regular, non sharded, repository for that particular entity, this class must extend ShardCore's GenericRepository\<T\> class.
6. Add a sharded repository for that same entity, this class must extend ShardCore's GenericShardedRepository\<T,R\> class. 
7. Setting up the ConfigureServices and Configure methods on the Startup class to initialize ShardCore.
8. Setting up migrations (optional).

### Real World Example

Let's keep using the same example where you want to provide product information. 

#### 1. Adding the Entity
The first step is to create the Product class and extend ShardCore's ShardedEntity class.

```
...
using Particle.Framework.ShardCore.Models;
...

public class Product : ShardedEntity
{
    public Product()
    {
        this.ProductProperties = new List<ProductProperty>();
    }

    public string Model { get; set; }
        
    public string Description { get; set; }

    public float Price { get; set; }

    public List<ProductProperty> ProductProperties { get; set; }
}
```

#### 2. Adding the Database Mapping
Add a class with the following mapping for the Product sharded entity.

```
public class ProductMap
{
    public static void RegisterMapping(ModelBuilder builder)
    {
        var entityBuilder = builder.Entity<Product>();

        entityBuilder.ToTable("Products");

        // Primary Key
        entityBuilder.HasKey(e => e.ShardId).HasName("ShardId");
        entityBuilder.Property(e => e.ShardId).HasColumnName("ShardId").ValueGeneratedOnAdd();
        entityBuilder.Property(e => e.Id).HasColumnName("GloballyUniqueId").IsRequired();
        entityBuilder.Property(e => e.Model).HasColumnName("Model");
        entityBuilder.Property(e => e.Description).HasColumnName("Description");
        entityBuilder.Property(e => e.Price).HasColumnName("Price");

        // Relationships
        entityBuilder
            .HasMany(e => e.ProductProperties)
            .WithOne(e => e.Product)
            .HasForeignKey(e => e.ProductId);

        // Indexes
        entityBuilder
            .HasIndex(e => e.Id)
            .IsUnique()
            .IsClustered(false);

        entityBuilder
            .HasIndex(e => e.Model)
            .IsUnique(false)
            .IsClustered(false);

        entityBuilder
            .HasIndex(e => e.Description)
            .IsUnique(false)
            .IsClustered(false);

        entityBuilder
            .HasIndex(e => e.Price)
            .IsUnique(false)
            .IsClustered(false);
    }
```

#### 3. Add a ShardDbContext
Create a ShardDbContext class with the following structure and add any mappings required in the OnModelCreating method. 

```
public class ShardDbContext : DbContext
{
    private readonly string connectionString;

    // This constructor is required to prevent an exception in bulk operations.
    public ShardDbContext() : base()
    {
    }
    
    public ShardDbContext(string connectionString) : base()
    {
        this.connectionString = connectionString;
    }

    public ShardDbContext(DbContextOptions<ShardDbContext> options)
        : base(options)
    {
    }

    protected override void OnModelCreating(ModelBuilder builder)
    {
        base.OnModelCreating(builder);

        // Register Mappings
        ProductMap.RegisterMapping(builder);
        ProductPropertyMap.RegisterMapping(builder);
    }

    protected override void OnConfiguring(DbContextOptionsBuilder optionsBuilder)
    {
        if (!string.IsNullOrEmpty(connectionString))
        {
            optionsBuilder.UseSqlServer(connectionString);
        }
    }
}
```

#### 4. Add a ShardUnitOfWork
Add a ShardUnitOfWork class that extends ShardCore's UnitOfWork class and receives a previously created ShardDbContext instance in its constructor.

```
...
using Particle.Framework.ShardCore.Implementations;
...
public class ShardUnitOfWork : UnitOfWork
{
    public ShardUnitOfWork(ShardDbContext context) : base(context)
    {
    }
}
```

#### 5. Add a non sharded repository for the Entity
Add a class that extends GenericRepository\<T\> and receives an instance of the previously created ShardUnitOfWork class.

```
...
using Particle.Framework.ShardCore.Implementations;
...
public class ProductsRepository : GenericRepository<Product>, IProductsRepository
{
    public ProductsRepository(ShardUnitOfWork unitOfWork)
        : base(unitOfWork)
    {
    }
}
```

#### 6. Add a sharded repository for the Entity
Add a class that extends GenericShardedRepository\<T,R\> class, where T is of your entity type R is of type GenericRepository\<T\>.
This class receives ShardCore's  ShardInformationRepository and IOptions\<ShardedRepositoryOptions\> as parameters, in the next section it will be shown how to configure these parameters.
  
```
...
using Particle.Framework.ShardCore.Implementations;
using Particle.Framework.ShardCore.Models;
...

public class ProductsShardedRepository : GenericShardedRepository<Product, ProductsRepository>, IProductsShardedRepository
{
    public ProductsShardedRepository(ShardInformationRepository shardInformationRepository, IServiceProvider serviceProvider, IOptions<ShardedRepositoryOptions> options) : base(shardInformationRepository, serviceProvider, options)
    {
    }

    public Product GetByShardKeyWithIncludes(Guid id)
    {
        var include = this.GetIncludes();
        return this.GetByShardKey(id, include);
    }
 
    private Func<IQueryable<Product>, IIncludableQueryable<Product, object>> GetIncludes()
    {
        Func<IQueryable<Product>, IIncludableQueryable<Product, object>> include = entity =>
            entity.Include(e => e.ProductProperties);

        return include;
    }
  
    ...
}
```
  
  
#### 7. Setting up Startup class
In the ConfigureServices method add the following code.

What each line does is described in the comments.

```
public void ConfigureServices(IServiceCollection services)
{
    // Sets the DbContext to access information about the shards.
    // Don't forget to set the migrations assembly.
    services.AddDbContext<ShardInformationDbContext>(options =>
       options.UseSqlServer("Server=YOURSQLSERVER\\SQLEXPRESS;Database=SHARDTEST_INFORMATION;Trusted_Connection=True;MultipleActiveResultSets=true",
       b => b.MigrationsAssembly(System.Reflection.Assembly.GetExecutingAssembly().FullName)));

    // Sets the DbContext used to access the shards.
    // A connection must be provided to prevent an exception but the connection for each shard will be resolved during runtime.
    services.AddDbContext<ShardDbContext>(options =>
       options.UseSqlServer("Server=YOURSQLSERVER\\SQLEXPRESS;Database=SHARDTEST_SHARD1;Trusted_Connection=True;MultipleActiveResultSets=true"),
       ServiceLifetime.Transient);

    // Sets the unit of work used for the shard information repository.
    services.AddScoped<ShardInformationUnitOfWork>();
    
    // Sets the unit of work used for the sharded repositories.
    services.AddTransient<ShardUnitOfWork>();
    
    // Sets the options for the sharded repositories.
    services.Configure<ShardedRepositoryOptions>(e =>
    {
        // ShardedEntities use a GUID as a unique identifier accross shards, this sets how many characters are reserved to identify the shard where it is stored.
        e.ShardIdLength = 8;
        
        // This option sets how data is stored accross shards. Random does a good job at keeping data well distributed and is the preferred choice.
        e.DataBalancingStrategy = DataBalancingStrategy.Random;
        
        // This option sets if shard information should be fetched from the database every time it is needed or if should be cached.
        e.ShardInformationCacheEnabled = true;
        
        // This option sets the shard information cache duration.
        e.ShardInformationCacheDurationInSeconds = 600;
        
        // Sets the connection timeout when accessing the shards.
        e.ShardsConnectionTimeoutInSeconds = 10;
        
        // This option allows to seed shard information when using database migrations.
        e.SeedShardsIfDontExist = true;
        
        // This specifies data about the shards
        e.Shards = new List<ShardInformation>()
        {
            new ShardInformation()
            {
                // A human friendly name for the shard.
                ShardFriendlyName = "Shard1",
                
                // The prefix that will be added in an entity GUID stored in this particular shard.
                ShardIdPrefix = "00000001",
                
                // The database connection string for the shard.
                ConnectionString = "Server=YOURSQLSERVER\\SQLEXPRESS;Database=SHARDTEST_SHARD1;Trusted_Connection=True;MultipleActiveResultSets=true",

                // The server name where this particular shard is located.
                // This parameter is required for cross shard querying.
                DatabaseServerName = "YOURSQLSERVER\\SQLEXPRESS",

                // The database name.
                // This parameter is required for cross shard querying.
                DatabaseName = "SHARDTEST_SHARD1",

                // The database schema.
                // This parameter is required for cross shard querying.
                DatabaseSchema = "dbo"
            },
            new ShardInformation()
            {
                ShardFriendlyName = "Shard2",
                ShardIdPrefix = "00000002",
                ConnectionString = "Server=YOURSQLSERVER\\SQLEXPRESS;Database=SHARDTEST_SHARD2;Trusted_Connection=True;MultipleActiveResultSets=true",
                DatabaseServerName = "YOURSQLSERVER\\SQLEXPRESS",
                DatabaseName = "SHARDTEST_SHARD2",
                DatabaseSchema = "dbo"
            }
        };
    });

    // Sets the shard information repository.
    services.AddTransient<ShardInformationRepository>();
    
    // Sets the non sharded repository.
    services.AddTransient<ProductsRepository>();
    
    // Sets the sharded repository.
    services.AddTransient<IProductsShardedRepository, ProductsShardedRepository>();
}
```

On the Configure method add the following code.

```
public void Configure(IApplicationBuilder app, IWebHostEnvironment env)
{
    app.UseShardCore();
    ...
}
```

#### 8. Setting up Migrations (optional)

If you require automatic migrations create the following class.

```
public static class IHostExtensions
{
    public static IHost MigrateDatabase(this IHost webHost)
    {
        var serviceScopeFactory = (IServiceScopeFactory)webHost.Services.GetService(typeof(IServiceScopeFactory));

        using (var scope = serviceScopeFactory.CreateScope())
        {
            var services = scope.ServiceProvider;
            var shardInformationDbContext = services.GetRequiredService<ShardInformationDbContext>();

            shardInformationDbContext.Database.Migrate();
                        
            var shardsConfiguration = services.GetRequiredService<IOptions<ShardedRepositoryOptions>>().Value;

            if (shardsConfiguration.SeedShardsIfDontExist)
            {
                var objectSet = shardInformationDbContext.Set<ShardInformation>();

                foreach (var shard in shardsConfiguration.Shards)
                {
                    var existingShard = objectSet.FirstOrDefault(e => e.ShardIdPrefix == shard.ShardIdPrefix);

                    if (existingShard == null)
                    {
                        shard.Enabled = true;
                        shard.ReadEnabled = true;
                        shard.WriteEnabled = true;
                        shard.UpdateEnabled = true;
                        shard.InsertDate = DateTime.UtcNow;
                        shardInformationDbContext.Add(shard);
                    }
                }

                shardInformationDbContext.SaveChanges();
            }

            foreach (var shard in shardsConfiguration.Shards)
            {     
                var shardDbContext = new ShardDbContext(shard.ConnectionString);
                shardDbContext.Database.Migrate();
            }
        }

        return webHost;
    }
}
```

And on the Program class use the MigrateDatabase extension.

```
public class Program
{
    public static void Main(string[] args)
    {
        CreateHostBuilder(args)
            .Build()
            .MigrateDatabase()
            .Run();
    }
    ...
}
```

## Important Notes

### Linked Servers
If you are using shards in different servers you have to configure them as linked servers. You can read how to configure linked servers [here](https://docs.microsoft.com/en-us/sql/relational-databases/linked-servers/create-linked-servers-sql-server-database-engine?view=sql-server-ver15).

One shard must be able to query any other shard using the following synthax.

```
select * from [ServerName].[DatabaseName].[DatabaseSchema].TableName with (nolock)
```
If all your shards are in the same servers this isn't required.

### Indexing
Indexing is very important for database searches, this is particularly true for sharded databases.

Pay a lot of attention to which columns will be used to search for information, and by which order, and add the respective indexes.

### Example Project
We have an example project with everything you need to know to start using ShardCore, you can find it [here](https://github.com/SoftParticle/ShardCoreTest).

## Pricing and Availability
ShardCore can be used up to three shards for free, if you require more shards please contact us at our business email on the profile page for pricing and availability.
