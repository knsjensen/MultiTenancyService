![Build & Publish to NuGet](https://github.com/MultiTenancyServer/MultiTenancyServer/workflows/Build%20&%20Publish%20to%20NuGet/badge.svg)

# MultiTenancyServer

MultiTenancyServer aims to be a lightweight package for adding multi-tenancy support to any codebase easily. 
Its design is heavily influenced from ASP.NET Core Identity. You can add multi-tenancy support to your model without adding any tenant key properties to any classes or entities. 
Using ASP.NET Core, the current tenant can be retrieved by a custom domain name, sub-domain, partial hostname, HTTP request header, 
child or partial URL path, query string parameter, authenticated user claim, or a custom request parser implementation. 
Using Entity Framework Core, tenant keys are added as shadow properties (or optionally concrete properties) and enforced through global query filters, 
all configurable options can be set from defaults or overidden per entity. 
The below example highlights how to use MultiTenancyServer with ASP.NET Core Identity and IdentityServer4 together. 
You can find many full working samples integrated with IdentityServer4, ASP.NET Core Identity (using different key types such as String and Int64), 
and  Entity Framework Core in the [samples repo](https://github.com/MultiTenancyServer/MultiTenancyServer.Samples).

## Define Model
Define your own tenant model, or inherit from TenancyTenant, or just use TenancyTenant as is. In this example we will inherit from TenancyTenant and add a display name.

``` csharp
public class Tenant : TenancyTenant
{
    // Custom property for display name of tenant.
    public string Name { get; set; }
}
```

## Register Services
Example of adding multi-tenancy support to ASP.NET Core.
``` csharp
public void ConfigureServices(IServiceCollection services)
{
    var connectionString = Configuration.GetConnectionString("DefaultConnection");
    var migrationsAssembly = typeof(AppDbContext).GetTypeInfo().Assembly.GetName().Name;

    services.AddDbContext<AppDbContext>(options =>
    {
        options.UseSqlServer(connectionString, sql => sql.MigrationsAssembly(migrationsAssembly));
    });

    // Add Multi-Tenancy Server defining TTenant<TKey> as type Tenant with an ID (key) of type string.
    services.AddMultiTenancy<Tenant, string>()
        // Add one or more IRequestParser (MultiTenancyServer.AspNetCore).
        .AddRequestParsers(parsers =>
        {
            // Parsers are processed in the order they are added,
            // typically 1 or 2 parsers should be all you need.
            parsers
                // www.tenant1.com
                .AddDomainParser()
                // tenant1.tenants.multitenancyserver.io
                .AddSubdomainParser(".tenants.multitenancyserver.io")
                // from partial hostname
                .AddHostnameParser("^(regular_expression)$")
                // HTTP header X-TENANT = tenant1
                .AddHeaderParser("X-TENANT")
                // /tenants/tenant1
                .AddChildPathParser("/tenants/")
                // from partial path
                .AddPathParser("^(regular_expression)$")
                // ?tenant=tenant1
                .AddQueryParser("tenant")
                // Claim from authenticated user principal.
                .AddClaimParser("http://schemas.microsoft.com/identity/claims/tenantid")
                // Add custom request parser with lambda.
                .AddCustomParser(httpContext => "tenant1");
                // Add custom request parser implementation.
                .AddMyCustomParser();
        })
        // Use in memory tenant store for development (MultiTenancyServer.Stores)
        .AddInMemoryStore(new Tenant[] 
        { 
            new Tenant() 
            { 
                Id = "TENANT_1", 
                CanonicalName = "Tenant1", 
                NormalizedCanonicalName = "TENANT1"
            }
        })
        // Use EF Core store for production (MultiTenancyServer.EntityFrameworkCore).
        .AddEntityFrameworkStore<AppDbContext, Tenant, string>()
        // Use custom store.
        .AddMyCustomStore();
        
    // Add ASP.NET Core Identity
    services.AddIdentity<User, Role>()
        .AddEntityFrameworkStores<AppDbContext>()
        .AddDefaultTokenProviders();

    // Add Identity Server 4
    var builder = services.AddIdentityServer()
        .AddAspNetIdentity<User>()
        // Add the config data from DB (clients, resources)
        .AddConfigurationStore<AppDbContext>(options =>
        {
            options.ConfigureDbContext = b =>
                b.UseSqlServer(connectionString,
                    sql => sql.MigrationsAssembly(migrationsAssemblyName));
        })
        // Add the operational data from DB (codes, tokens, consents)
        .AddOperationalStore<AppDbContext>(options =>
        {
            options.ConfigureDbContext = b =>
                b.UseSqlServer(connectionString,
                    sql => sql.MigrationsAssembly(migrationsAssemblyName));
        });

    if (Environment.IsDevelopment())
    {
        builder.AddDeveloperSigningCredential();
    }
    else
    {
        throw new Exception("Key not configured.");
    }
}    
```

## Add Middleware
Example of configuring application with multi-tenancy support for ASP.NET Core MVC and IdentityServer4.
``` csharp
public void Configure(IApplicationBuilder app)
{
    // other code removed for brevity

    app.UseStaticFiles();
    app.UseMultiTenancy<Tenant>();
    app.UseIdentityServer();
    app.UseAuthentication();
    app.UseMvcWithDefaultRoute();
}
```

## Configure Entities
Example of DbContext with multi-tenancy support for ASP.NET Core Identity and IdentityServer4.
``` csharp
    public class AppDbContext : 
        // ASP.NET Core Identity EF Core
        IdentityDbContext<User, Role, string, UserClaim, UserRole, UserLogin, RoleClaim, UserToken>, 
        // IdentityServer4 EF Core
        IConfigurationDbContext, IPersistedGrantDbContext,
        // MultiTenancyServer EF Core
        ITenantDbContext<Tenant, string>
    {
        private static object _tenancyModelState;
        private readonly ITenancyContext<Tenant> _tenancyContext;

        public AppDbContext(
            DbContextOptions<AppDbContext> options, 
            ITenancyContext<Tenant> tenancyContext)
            : base(options)
        {
            // The request scoped tenancy context.
            // Should not access the tenancyContext.Tenant property in the constructor yet,
            // as the request pipeline has not finished running yet and it will likely be null.
            _tenancyContext = tenancyContext;
        }

        // IdentityServer4 implementation.
        public DbSet<Client> Clients { get; set; }
        public DbSet<IdentityResource> IdentityResources { get; set; }
        public DbSet<ApiResource> ApiResources { get; set; }
        public DbSet<PersistedGrant> PersistedGrants { get; set; }

        // MultiTenancyServer implementation.
        public DbSet<Tenant> Tenants { get; set; }

        protected override void OnModelCreating(ModelBuilder builder)
        {
            base.OnModelCreating(builder);

            // IdentityServer4 configuration.
            var configurationStoreOptions = new ConfigurationStoreOptions();
            builder.ConfigureClientContext(configurationStoreOptions);
            builder.ConfigureResourcesContext(configurationStoreOptions);
            var operationalStoreOptions = new OperationalStoreOptions();
            builder.ConfigurePersistedGrantContext(operationalStoreOptions);

            // MultiTenancyServer configuration.
            var tenantStoreOptions = new TenantStoreOptions();
            builder.ConfigureTenantContext<Tenant, string>(tenantStoreOptions);

            // Add multi-tenancy support to model.
            var tenantReferenceOptions = new TenantReferenceOptions();
            builder.HasTenancy<string>(tenantReferenceOptions, out _tenancyModelState);

            // Configure custom properties on Tenant (MultiTenancyServer).
            builder.Entity<Tenant>(b =>
            {
                b.Property(t => t.Name).HasMaxLength(256);
            });

            // Configure properties on User (ASP.NET Core Identity).
            builder.Entity<User>(b =>
            {
                // Add multi-tenancy support to entity.
                b.HasTenancy(() => _tenantId, _tenancyModelState, hasIndex: false);
                // Remove unique index on NormalizedUserName.
                b.HasIndex(u => u.NormalizedUserName).HasName("UserNameIndex").IsUnique(false);
                // Add unique index on TenantId and NormalizedUserName.
                b.HasIndex(tenantReferenceOptions.ReferenceName, nameof(User.NormalizedUserName))
                    .HasName("TenantUserNameIndex").IsUnique();
            });

            // Configure properties on Role (ASP.NET Core Identity).
            builder.Entity<Role>(b =>
            {
                // Add multi-tenancy support to entity.
                b.HasTenancy(() => _tenancyContext.Tenant.Id, _tenancyModelState, hasIndex: false);
                // Remove unique index on NormalizedName.
                b.HasIndex(r => r.NormalizedName).HasName("RoleNameIndex").IsUnique(false);
                // Add unique index on TenantId and NormalizedName.
                b.HasIndex(tenantReferenceOptions.ReferenceName, nameof(Role.NormalizedName))
                    .HasName("TenantRoleNameIndex").IsUnique();
            });

            // Configure properties on Client (IdentityServer4).
            builder.Entity<Client>(b =>
            {
                // Add multi-tenancy support to entity.
                b.HasTenancy(() => _tenancyContext.Tenant.Id, _tenancyModelState, hasIndex: false);
                // Remove unique index on ClientId.
                b.HasIndex(c => c.ClientId).IsUnique(false);
                // Add unique index on TenantId and ClientId.
                b.HasIndex(tenantReferenceOptions.ReferenceName, nameof(Client.ClientId)).IsUnique();
            });

            // Configure properties on IdentityResource (IdentityServer4).
            builder.Entity<IdentityResource>(b =>
            {
                // Add multi-tenancy support to entity.
                b.HasTenancy(() => _tenancyContext.Tenant.Id, _tenancyModelState, hasIndex: false);
                // Remove unique index on Name.
                b.HasIndex(r => r.Name).IsUnique(false);
                // Add unique index on TenantId and Name.
                b.HasIndex(tenantReferenceOptions.ReferenceName, nameof(IdentityResource.Name)).IsUnique();
            });

            // Configure properties on ApiResource (IdentityServer4).
            builder.Entity<ApiResource>(b =>
            {
                // Add multi-tenancy support to entity.
                b.HasTenancy(() => _tenancyContext.Tenant.Id, _tenancyModelState, hasIndex: false);
                // Remove unique index on Name.
                b.HasIndex(r => r.Name).IsUnique(false);
                // Add unique index on TenantId and Name.
                b.HasIndex(tenantReferenceOptions.ReferenceName, nameof(ApiResource.Name)).IsUnique();
            });

            // Configure properties on ApiScope (IdentityServer4).
            builder.Entity<ApiScope>(b =>
            {
                // Add multi-tenancy support to entity.
                b.HasTenancy(() => _tenancyContext.Tenant.Id, _tenancyModelState, hasIndex: false);
                // Remove unique index on Name.
                b.HasIndex(s => s.Name).IsUnique(false);
                // Add unique index on TenantId and Name.
                b.HasIndex(tenantReferenceOptions.ReferenceName, nameof(ApiScope.Name)).IsUnique();
            });

            // Configure properties on PersistedGrant (IdentityServer4).
            builder.Entity<PersistedGrant>(b =>
            {
                // Add multi-tenancy support to entity.
                b.HasTenancy(() => _tenancyContext.Tenant.Id, _tenancyModelState);
            });
        }

        public override int SaveChanges(bool acceptAllChangesOnSuccess)
        {
            // Ensure multi-tenancy for all tenantable entities.
            this.EnsureTenancy(_tenancyContext?.Tenant?.Id, _tenancyModelState, _logger);
            return base.SaveChanges(acceptAllChangesOnSuccess);
        }

        public override Task<int> SaveChangesAsync(bool acceptAllChangesOnSuccess, CancellationToken cancellationToken = default)
        {
            // Ensure multi-tenancy for all tenantable entities.
            this.EnsureTenancy(_tenancyContext?.Tenant?.Id, _tenancyModelState, _logger);
            return base.SaveChangesAsync(acceptAllChangesOnSuccess, cancellationToken);
        }
    }
```

## Additional Options

``` csharp
public class TenantReferenceOptions
{
    // Summary:
    //     If set to a non-null value, the store will use this value as the name for the
    //     tenant's reference property. The default is "TenantId".
    public string ReferenceName { get; set; }

    // Summary:
    //     True to enable indexing of tenant reference properties in the store, otherwise
    //     false. The default is true.
    public bool IndexReferences { get; set; }

    // Summary:
    //     If set to a non-null value, the store will use this value as the name of the
    //     index for any tenant references. The name is also a format pattern of {0:PropertyName}.
    //     The default is "{0}Index", eg. "TenantIdIndex".
    public string IndexNameFormat { get; set; }

    // Summary:
    //     Determines if a null tenant reference is allowed for entities and how querying
    //     for null tenant references is handled.
    public NullTenantReferenceHandling NullTenantReferenceHandling { get; set; }
}

public enum NullTenantReferenceHandling
{
    // Summary:
    //     A null tenant reference is NOT allowed for the entity, where possible a NOT NULL
    //     or REQUIRED constraint should be set on the tenant reference, querying for entities
    //     with a null tenant reference will match NO entities.
    //     This is the default option.
    NotNullDenyAccess = 0,

    // Summary:
    //     A null tenant reference is allowed for the entity, where possible an NULLABLE
    //     or OPTIONAL constraint should be set on the tenant reference, querying for entities
    //     with a null tenant reference will match those expected results.
    //     This may be useful where globally defined system entities are set with a null
    //     tenant reference.
    NullableEntityAccess = 1,

    // Summary:
    //     A null tenant reference is NOT allowed for the entity, where possible a NOT NULL
    //     or REQUIRED constraint should be set on the tenant reference, querying for entities
    //     with a null tenant reference will match ALL entities across all tenants.
    //     For obvious security reasons, this is typically not recommended; however, this
    //     can be useful for admin reporting across all tenants.
    NotNullGlobalAccess = 2
}
