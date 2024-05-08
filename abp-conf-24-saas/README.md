# Presentation

## SaaS

- **On-site** means on-premise hosting, when users download desktop applications to their computers. That means we're giving on-site application services. The other three are considered as cloud service models.
- **Infrastructure as service**, all computing resource in a virtual environment. It is used by system administrators.
- **Platform as Service**, encapsulate the environment where users can run their application without worrying of the underlying infrastructure. It is used by developers. For instance when we deploy our application to existing kubernetes cluster we need to only manage the deployment yaml files.
- Lastly **SaaS** or **software as service** it's a cloud based software delivery model. The users only need to open and use the application without worry about anything else. This is for end customers.

Basically, we can say Infrastructure as service target audiences for DevOps, Platform as Service for Developers and Software as Service for end customers.
## Key Concepts

In a multi-tenant application, there is a term for the tenancy side, which could be host or tenant.
- Tenants are users who access the application and service.
- The host is the organization that is responsible for providing the service and managing all the tenants.

Additionally, other terms related to the application:
- Feature a functionality of the application.
- Edition or Package a group of application features.
- Subscription assigning a package to a tenant for a period of time. 
- Payment a subscription price.
## Sample Pricing Table

We can look to GitHub pricing table for an example.

- Free, Team and Enterprise are the editions.
- In the below of a package information we can see the payment and subcription model.
- Additionally, we can see the features in green highlights.

## Multi-Tenancy

A software architecture where a single codebase of the application serves multiple customers. It's a sub-term for SaaS. The main difference between SaaS and multi-tenancy is the single codebase. Each URL that we give to our customers is considered as SaaS and it could connect to different servers and load balancers with different application codebases. However, for the multi-tenant application, we can still give different URLs, but all URLs connect to the same server or load balancer with a single application codebase. In such a scenario, there are some pros and cons.

The main pros are cost efficiency, simple maintenance, and faster deployment. On the other hand, the main cons are that since we have only one code base, it limits customization, and there might be some security concerns.

## Deployment & Database Architectures

There are various of types we can deploy our SaaS applications. 

- **Separate Codebase and Separate DB**: We can deploy the application directly to a server and provide a URL to each tenant. However, this approach requires us to repeat the deployment process for each tenant. It's still SaaS, but it does not use the multi-tenancy system, making it less convenient solution.
The other three are considered as multi-tenant application and ABP supports each scenario.
- **Shared Codebase and Separate DB**: We deploy our application to a server and provide the URL to all our tenants. However, we maintain separate databases for each tenant.
- **Shared Codebase and Shared DB**: This system is similar to the separate database system, but it involves sharing all tenants data in a single database. The distinction between tenants is determined by the _TenantId_ column in multi-tenant tables.
- **Shared Codebase and Hybrid DB**: Some tenants use for a shared database, while others may prefer to use their own databases.

## ABP Tenant Connection String Edit

In the ABP Framework's SaaS module, we can edit tenant connection strings in the _Database Connection Strings_ tab. Additionally, we can use a different database for each module within a single tenant. While this approach can be more challenging to maintain, it provides flexibility if needed.

## Tenant Identification in Multi-Tenant Environments

When all our tenants request the same application codebase, we need to determine the current tenant, and each tenant should only have the capability to edit their own data. ABP Framework provides 6 pre-built tenant resolvers for this purpose.

- **Current User**: If a user is logged into the system, we should always check their claims for the tenant id, as it could cause security issues if we don't take precautions.
- **Domain Resolver**: This resolver is optional; we can use it to determine the tenant based on the domain name in the request URL.
- **Query String**: We can include the tenant id in the query string of the request URL to specify the intended tenant.
- **Route**: The tenant id can be extracted from the route parameters of the request URL to identify the corresponding tenant.
- **Header**: Tenant information can be passed in the request header to designate the specific tenant for the request.
- **Cookie**: Tenants can be identified using cookies, where the tenant id is stored and retrieved from the client's browser.

When one of the tenant resolvers determines the current tenant, it should stop the resolving process from other resolvers. For instance, when a user is logged in, we always retrieve their tenant information from their claims. Otherwise, there might be a risk of adding a different tenant id in the header and accessing other tenants data. Lastly, we can also create our own custom resolvers if we need.

## Let's See It in Action: The Demonstration

If you have an ABP commercial license, you can download the sample from the following [link](https://abp.io/api/download/samples/conf-24-saas ).

---
# Code Example
## Project Structure

In this example, we're using a different project structure than our existing templates. There are 3 host application. 

- AuthServer for authentication.
- Admin.Host => System administration operations.
- Tenant.Host => Tenants or customers gonna use this application.

Additionally there is a **ProductManagement** module and Tenant.Host depended this module. Admin host only depended the EfCore package because of migration operations. 

- Acme.CloudCrm.Admin.HttpApi.Host.csproj#47
- Acme.CloudCrm.HttpApi.Host#27

## Database Migration

We have 3 DbContext classes: a base DbContext for general mapping, an Admin DbContext for host-side migration where we explicitly set the tenancy side to host. We're following the same approach for the tenant DbContext as well.

- CloudCrmDbContextBase#37
- CloudCrmAdminDbContext#16
- CloudCrmDbContext#16

We're utilizing the runtime migration approach. The host-side database is initialized on host application start, and tenant databases are migrated upon tenant creation, connection string change, or applying a database request.

- CloudCrmAdminHostModule#236
- CloudCrmAdminRuntimeDatabaseMigrator  
	- It takes inhertiance from [EfCoreRuntimeDatabaseMigratorBase](https://github.com/abpframework/abp/blob/4103618f861fe692b0ab56572daa0a9385ec4759/framework/src/Volo.Abp.EntityFrameworkCore/Volo/Abp/EntityFrameworkCore/Migrations/EfCoreRuntimeDatabaseMigratorBase.cs#L16)
	- It is used for host side migration
- CloudCrmDatabaseMigrationEventHandler
	- It takes inheritance from [EfCoreDatabaseMigrationEventHandlerBase](https://github.com/abpframework/abp/blob/4103618f861fe692b0ab56572daa0a9385ec4759/framework/src/Volo.Abp.EntityFrameworkCore/Volo/Abp/EntityFrameworkCore/Migrations/EfCoreDatabaseMigrationEventHandlerBase.cs#L18)
	- It is used for tenant side migration

## Why We Need Shared Project ?

Since we have different host applications, we can define common things in shared project. Such as infrastructure packages, cache and rabbitmq settings etc.
If we don't wanna use the multi tenancy we can disable with **AbpMultiTenancyOptions** option. After we change IsEnable to false we should also remove the UseMultiTenancy middleware from host applications. However application templates already add this check.
## Creating a Multi Tenant Entity

We have **Product** aggregate root implements the [IMultiTenant](https://github.com/abpframework/abp/blob/4103618f861fe692b0ab56572daa0a9385ec4759/framework/src/Volo.Abp.MultiTenancy.Abstractions/Volo/Abp/MultiTenancy/IMultiTenant.cs#L5) interface and it forces the create **TenantId** property. It's nullable guid since host side could also use this entity. If you want to use it only for tenants, you can add a **tenant id** property in the constructor parameters like this example. If we don't provide the tenant id parameter in constructer, it attempts to set the tenant id instance creation on the entity base.

- Product#18
- [Volo.Abp.Domain.Entities.Entity](https://github.com/abpframework/abp/blob/4103618f861fe692b0ab56572daa0a9385ec4759/framework/src/Volo.Abp.Ddd.Domain/Volo/Abp/Domain/Entities/Entity.cs#L12)

We have basic business rule for this domain. The Product entity has code and name properties, with the code required to be unique within each tenant. While a code can exist in one tenant, other tenants may also have the same code. However, if a tenant attempts to create a product with a code that already exists within that tenant, it should throw an exception.

- ProductManager#24,#41

In the EfCore package, we create an implementation for the product repository and also map the product entity with database tenant side check in db context model creating extension. If we wanna add only host side entity we can use IsHostDatabase method.

- EfCoreProductRepository
- ProductManagementDbContextModelCreatingExtensions#17

Lastly, we create related app service and HTTP API controller for the product entity. 
When we create permissions, we explicitly set the tenancy side to tenant, ensuring that the host side cannot use these controller methods.

- ProductAppService
- ProductController
- ProductManagementPermissionDefinitionProvider#19

### ProductAppService usage

First we need to create some tenants for demonstration. 

- Currently we haven't any tenants. ==GET host/tenants/list==.
	- Create t-1, t-2, t-3-shared, t-4-shared. Don't forget the change **connection strings**.
	- You can list users in shared database which has two admin user but has different tenant id for t-3 and t-4.
- Create P-1 for each tenant ==GET tenant/products/list==.
	- We can create P-1 for shared db context for all tenants however when we try to create P-1 in same tenant we get business exception.

## Determining Current Tenant

When we send a request to the tenant application, it uses the [Multitenancy](https://github.com/abpframework/abp/blob/4103618f861fe692b0ab56572daa0a9385ec4759/framework/src/Volo.Abp.AspNetCore.MultiTenancy/Volo/Abp/AspNetCore/MultiTenancy/MultiTenancyMiddleware.cs#L19) middleware. This middleware attempts to find the requested tenant and checks if the tenant exists and active. If it exists, it changes the current tenant ([ICurrentTenant.Change](https://github.com/abpframework/abp/blob/4103618f861fe692b0ab56572daa0a9385ec4759/framework/src/Volo.Abp.AspNetCore.MultiTenancy/Volo/Abp/AspNetCore/MultiTenancy/MultiTenancyMiddleware.cs#L61)) to the requested one.

- CloudCrmHttpApiHostModule#148
- [MultiTenancyMiddleware](https://github.com/abpframework/abp/blob/4103618f861fe692b0ab56572daa0a9385ec4759/framework/src/Volo.Abp.AspNetCore.MultiTenancy/Volo/Abp/AspNetCore/MultiTenancy/MultiTenancyMiddleware.cs#L19)
- [TenantConfigurationProvider](https://github.com/abpframework/abp/blob/4103618f861fe692b0ab56572daa0a9385ec4759/framework/src/Volo.Abp.MultiTenancy/Volo/Abp/MultiTenancy/TenantConfigurationProvider.cs#L33)
- [TenantResolver](https://github.com/abpframework/abp/blob/4103618f861fe692b0ab56572daa0a9385ec4759/framework/src/Volo.Abp.MultiTenancy/Volo/Abp/MultiTenancy/TenantResolver.cs#L29)
	- Resolvers added in [AbpAspNetCoreMultiTenancyModule](https://github.com/abpframework/abp/blob/4103618f861fe692b0ab56572daa0a9385ec4759/framework/src/Volo.Abp.AspNetCore.MultiTenancy/Volo/Abp/AspNetCore/MultiTenancy/AbpAspNetCoreMultiTenancyModule.cs#L17), [AbpMultiTenancyModule](https://github.com/abpframework/abp/blob/4103618f861fe692b0ab56572daa0a9385ec4759/framework/src/Volo.Abp.MultiTenancy/Volo/Abp/MultiTenancy/AbpMultiTenancyModule.cs#L35), [AbpMultiTenancyOptionsExtensions](https://github.com/abpframework/abp/blob/4103618f861fe692b0ab56572daa0a9385ec4759/framework/src/Volo.Abp.AspNetCore.MultiTenancy/Volo/Abp/MultiTenancy/AbpMultiTenancyOptionsExtensions.cs#L11).

After we resolve the tenant we're looking for tenant exists and active in [TenantConfigurationProvider](https://github.com/abpframework/abp/blob/4103618f861fe692b0ab56572daa0a9385ec4759/framework/src/Volo.Abp.MultiTenancy/Volo/Abp/MultiTenancy/TenantConfigurationProvider.cs#L45). Afterwards we change the current tenant in the ambient context.

## Current Tenant

We can get current tenant informations with [ICurrentTenant](https://github.com/abpframework/abp/blob/4103618f861fe692b0ab56572daa0a9385ec4759/framework/src/Volo.Abp.MultiTenancy.Abstractions/Volo/Abp/MultiTenancy/ICurrentTenant.cs#L6) service and change the current tenant using the ambient context pattern. However, when we change the current tenant, it doesn't check whether the tenant exists or not.

- TenantExamplesController#40
- [CurrentTenant](https://github.com/abpframework/abp/blob/4103618f861fe692b0ab56572daa0a9385ec4759/framework/src/Volo.Abp.MultiTenancy/Volo/Abp/MultiTenancy/CurrentTenant.cs#L23)
- [AsyncLocalCurrentTenantAccessor](https://github.com/abpframework/abp/blob/4103618f861fe692b0ab56572daa0a9385ec4759/framework/src/Volo.Abp.MultiTenancy/Volo/Abp/MultiTenancy/AsyncLocalCurrentTenantAccessor.cs#L15) It uses the static async local [BasicTenantInfo](https://github.com/abpframework/abp/blob/4103618f861fe692b0ab56572daa0a9385ec4759/framework/src/Volo.Abp.MultiTenancy.Abstractions/Volo/Abp/MultiTenancy/BasicTenantInfo.cs#L6) property.

## Creating Custom Tenant Resolver

To create a custom tenant resolver, inherit from [TenantResolveContributorBase](https://github.com/abpframework/abp/blob/4103618f861fe692b0ab56572daa0a9385ec4759/framework/src/Volo.Abp.MultiTenancy/Volo/Abp/MultiTenancy/TenantResolveContributorBase.cs#L5). In the **ResolveAsync** method, set **context.Handled** to true or **context.TenantIdOrName** to break the loop. For instance, in this example, a custom resolver is created where if the current user tenancy side is host and a tenant id is set in the header, it is accepted as the requested tenant.

- HostUserHeaderTenantResolveContributor#40
- Retrive a token from host side with ==POST token==.
- We can list products with ==GET tenant/products/list== after we add tenant id or name in the header. We've defined the permission for only tenants that's why we should change the tenancy side for both.

## Data Isolation

### Connection String Resolver

ABP uses the [MultiTenantConnectionStringResolver](https://github.com/abpframework/abp/blob/4103618f861fe692b0ab56572daa0a9385ec4759/framework/src/Volo.Abp.MultiTenancy/Volo/Abp/MultiTenancy/MultiTenantConnectionStringResolver.cs#L28) to find the connection string for the current tenant and requested module database name. This lets us share connection strings between tenants and modules, or have different databases for each tenant and module. If the tenancy side is host, it gets the connection string from the configuration. Otherwise, it tries to get it from the database.

- TenantExamplesController#102
- [MultiTenantConnectionStringResolver](https://github.com/abpframework/abp/blob/4103618f861fe692b0ab56572daa0a9385ec4759/framework/src/Volo.Abp.MultiTenancy/Volo/Abp/MultiTenancy/MultiTenantConnectionStringResolver.cs#L28)
### Data Filter

When tenants share the connection string, we use the [IDataFilter](https://github.com/abpframework/abp/blob/4103618f861fe692b0ab56572daa0a9385ec4759/framework/src/Volo.Abp.Data/Volo/Abp/Data/IDataFilter.cs#L5) for data isolation. In EF Core, we apply a global query filter so that each database request automatically adds the tenant id from the current tenant.

- [AbpDbContext](https://github.com/abpframework/abp/blob/4103618f861fe692b0ab56572daa0a9385ec4759/framework/src/Volo.Abp.EntityFrameworkCore/Volo/Abp/EntityFrameworkCore/AbpDbContext.cs#L755)
- ProductAppService#87 => if we send request to ==GET tenant/products/disabled-data-filter== with shared one of the tenant, we're gonna see all the products.

### Ignoring Multi-tenancy

If your db context or cache item hasn't tenancy side we can ignore with [IgnoreMultiTenancyAttribute](https://github.com/abpframework/abp/blob/4103618f861fe692b0ab56572daa0a9385ec4759/framework/src/Volo.Abp.MultiTenancy.Abstractions/Volo/Abp/MultiTenancy/IgnoreMultiTenancyAttribute.cs#L6). For instance you can look to [FeatureManagementDbContext](https://github.com/abpframework/abp/blob/4103618f861fe692b0ab56572daa0a9385ec4759/modules/feature-management/src/Volo.Abp.FeatureManagement.EntityFrameworkCore/Volo/Abp/FeatureManagement/EntityFrameworkCore/FeatureManagementDbContext.cs#L8) and [FeatureValueCacheItem](https://github.com/abpframework/abp/blob/4103618f861fe692b0ab56572daa0a9385ec4759/modules/feature-management/src/Volo.Abp.FeatureManagement.Domain/Volo/Abp/FeatureManagement/FeatureValueCacheItem.cs#L7)
## Miscellaneous

- We can use the [ITenantStore](https://github.com/abpframework/abp/blob/4103618f861fe692b0ab56572daa0a9385ec4759/framework/src/Volo.Abp.MultiTenancy.Abstractions/Volo/Abp/MultiTenancy/ITenantStore.cs#L7) from other modules to find a tenant by id or name. Additionally, we can list all tenants using the **GetListAsync** method.
	- TenantExamplesController#64
- When we enqueue a background job if job args take implementation from IMultiTenant interface we don't need to change current tenant in job handler.
	- [BackgroundJobExecuter](https://github.com/abpframework/abp/blob/4103618f861fe692b0ab56572daa0a9385ec4759/framework/src/Volo.Abp.BackgroundJobs.Abstractions/Volo/Abp/BackgroundJobs/BackgroundJobExecuter.cs#L48)#48
- Just like background job system if our event class take implementation from **IMultiTenant** interface we don't need to change current tenant in event handler.
	- [EventBusBase](https://github.com/abpframework/abp/blob/4103618f861fe692b0ab56572daa0a9385ec4759/framework/src/Volo.Abp.EventBus/Volo/Abp/EventBus/EventBusBase.cs#L214)
## Conclusion

Overall, this example shows how easy it is to make multi-tenant application with ABP. By following good coding practices, developers can handle the complexity of multi-tenancy and create strong, flexible solutions. You can use IMultiTenant interface and everything else handled by ABP framework.