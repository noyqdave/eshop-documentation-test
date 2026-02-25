# eShopOnWeb: 4+1 Architectural Views

> **Reverse-engineered** from the codebase. Describes the architecture of the eShopOnWeb ASP.NET Core reference application—a monolithic e-commerce web app demonstrating traditional web development patterns.

**Related documentation**:
- [Use Case Specifications](use-case-specifications.md) – UC-001 through UC-015 with flows and code mapping
- [Practices Guide](practices-guide.md) – BDD, TDD, and use case documentation practices

---

## Overview

eShopOnWeb is a single-process (monolithic) ASP.NET Core application with:
- **Web**: MVC + Razor Pages for storefront and identity
- **PublicApi**: REST API (minimal API endpoints) for Blazor Admin
- **BlazorAdmin**: Blazor WebAssembly admin UI (catalog CRUD)
- **ApplicationCore**: Domain entities, interfaces, services
- **Infrastructure**: Data access, identity, external services
- **BlazorShared**: Shared DTOs and interfaces for Blazor/API

---

## 1. Logical View

The logical view describes the key abstractions and domain model.

### Domain Entities (ApplicationCore)

```mermaid
classDiagram
    class CatalogItem {
        <<IAggregateRoot>>
        +Id
        +Name
        +Description
        +Price
        +PictureUri
        +CatalogBrandId
        +CatalogTypeId
    }
    class CatalogBrand {
        +Id
        +Brand
    }
    class CatalogType {
        +Id
        +Type
    }
    class Basket {
        <<IAggregateRoot>>
        +Id
        +BuyerId
        +Items IReadOnlyCollection
        +AddItem()
        +RemoveEmptyItems()
    }
    class BasketItem {
        +Id
        +CatalogItemId
        +Quantity
        +UnitPrice
        +AddQuantity()
        +SetQuantity()
    }
    class Order {
        <<IAggregateRoot>>
        +Id
        +BuyerId
        +ShipToAddress
        +OrderItems
    }
    class OrderItem {
        +ItemOrdered
        +UnitPrice
        +Units
    }
    class CatalogItemOrdered {
        +CatalogItemId
        +ProductName
        +PictureUri
    }
    class Address {
        <<Value Object>>
        +Street
        +City
        +State
        +Country
        +ZipCode
    }
    CatalogItem --> CatalogBrand : CatalogBrandId
    CatalogItem --> CatalogType : CatalogTypeId
    Basket "1" --> "*" BasketItem : Items
    BasketItem --> CatalogItem : CatalogItemId
    Order "1" --> "*" OrderItem : OrderItems
    OrderItem --> CatalogItemOrdered : ItemOrdered
    Order --> Address : ShipToAddress
```

### Key Interfaces

| Interface | Purpose |
|-----------|---------|
| `IRepository<T>` | Generic CRUD for aggregates |
| `IReadRepository<T>` | Read-only repository |
| `IBasketService` | Add item, set quantities, transfer, delete basket |
| `IOrderService` | Create order from basket |
| `ICatalogViewModelService` | Catalog with filters, pagination, brands, types |
| `IBasketViewModelService` | Basket view model mapping |
| `IUriComposer` | Compose picture URIs with base URL |
| `IEmailSender` | Send email (e.g., confirmation) |
| `ITokenClaimsService` | JWT claim generation |

### Specifications (Query Objects)

- `CatalogFilterSpecification`, `CatalogFilterPaginatedSpecification` – filter by brand/type, paginate
- `CatalogItemsSpecification` – load catalog items by IDs
- `BasketWithItemsSpecification` – basket by buyer ID or basket ID with items
- `CustomerOrdersSpecification` – orders by buyer ID
- `OrderWithItemsByIdSpec` – order with items by order ID

### Key Domain Services

- **BasketService**: AddItemToBasket, SetQuantities, TransferBasketAsync, DeleteBasketAsync
- **OrderService**: CreateOrderAsync (from basket + address)

### Logical Interaction Flows

The following sequence diagrams show domain-level interactions between entities and services. These complement the process-view diagrams, which describe physical request flows.

#### Add Item to Basket

```mermaid
sequenceDiagram
    participant Caller
    participant BasketService
    participant Basket
    participant BasketItem

    Caller->>+BasketService: AddItemToBasket(username, catalogItemId, price, quantity)
    BasketService->>BasketService: Load or create Basket (by buyer ID)
    BasketService->>+Basket: AddItem(catalogItemId, price, quantity)
    alt Item not in basket
        Basket->>BasketItem: new BasketItem(catalogItemId, quantity, unitPrice)
    else Item already in basket
        Basket->>BasketItem: AddQuantity(quantity)
    end
    Basket-->>-BasketService: (void)
    BasketService->>BasketService: Persist basket
    BasketService-->>-Caller: Basket
```

#### Checkout (Create Order)

```mermaid
sequenceDiagram
    participant Caller
    participant OrderService
    participant Basket
    participant Order
    participant OrderItem
    participant CatalogItemOrdered

    Caller->>+OrderService: CreateOrderAsync(basketId, address)
    OrderService->>OrderService: Load Basket (BasketWithItemsSpecification)
    OrderService->>OrderService: Load CatalogItems (CatalogItemsSpecification)
    loop For each basket item
        OrderService->>CatalogItemOrdered: new (from CatalogItem snapshot)
        OrderService->>OrderItem: new (itemOrdered, unitPrice, quantity)
    end
    OrderService->>Order: new Order(buyerId, address, orderItems)
    OrderService->>OrderService: Persist order
    OrderService-->>-Caller: (void)
```

#### Basket Transfer on Login

```mermaid
sequenceDiagram
    participant Caller
    participant BasketService
    participant AnonymousBasket as Basket (anonymous)
    participant UserBasket as Basket (user)
    participant BasketItem

    Caller->>+BasketService: TransferBasketAsync(anonymousId, userName)
    BasketService->>AnonymousBasket: Load (BasketWithItemsSpecification)
    opt Anonymous basket exists
        BasketService->>UserBasket: Load or create (BasketWithItemsSpecification)
        loop For each item in anonymous basket
            BasketService->>UserBasket: AddItem(catalogItemId, unitPrice, quantity)
            UserBasket->>BasketItem: new or AddQuantity
        end
        BasketService->>UserBasket: Persist
        BasketService->>AnonymousBasket: Delete
    end
    BasketService-->>-Caller: (void)
```

#### Update Basket Quantities

```mermaid
sequenceDiagram
    participant Caller
    participant BasketService
    participant Basket
    participant BasketItem

    Caller->>+BasketService: SetQuantities(basketId, quantities)
    BasketService->>Basket: Load (BasketWithItemsSpecification)
    loop For each basket item
        BasketService->>BasketItem: SetQuantity(quantity)
    end
    BasketService->>Basket: RemoveEmptyItems()
    Basket->>Basket: Remove items with quantity 0
    BasketService->>BasketService: Persist basket
    BasketService-->>-Caller: Basket
```

### Exceptions

- `EmptyBasketOnCheckoutException` – basket has no items at checkout
- `DuplicateException` – (defined, usage varies)

---

## 2. Process View

The process view describes runtime behavior, concurrency, and communication.

### Request Flow: Storefront (Web)

```mermaid
sequenceDiagram
    participant Browser
    participant Web as ASP.NET Core Web
    participant VM as ViewModel Services
    participant Core as ApplicationCore
    participant DB as SQL Server

    Browser->>+Web: HTTP Request
    Web->>Web: Cookie Auth (Identity)
    Web->>Web: Razor Pages / MVC Controllers
    Web->>+VM: Get catalog or basket (ICatalogViewModelService, IBasketViewModelService)
    VM->>+Core: BasketService, OrderService
    Core->>+DB: EF Core → CatalogContext
    DB-->>-Core: Data
    Core-->>-VM: Result
    VM-->>-Web: View model
    Note over Web: MediatR (OrderController)
    Web-->>-Browser: HTTP Response
```

### Request Flow: Admin (Blazor + PublicApi)

```mermaid
sequenceDiagram
    participant Browser
    participant Web as Web (serves Blazor WASM)
    participant Blazor as Blazor WASM
    participant API as PublicApi
    participant DB as SQL Server

    Browser->>Web: HTTP GET /admin
    Web-->>Browser: BlazorAdmin WASM app
    Browser->>+Blazor: Run app
    Blazor->>API: POST /api/authenticate (login)
    API->>API: AuthEndpoints → JWT
    API-->>Blazor: JWT token
    Blazor->>+API: fetch (Bearer token) CRUD, CatalogType, CatalogBrand
    API->>API: JWT Bearer Auth
    API->>API: CatalogItemEndpoints, CatalogTypeEndpoints, CatalogBrandEndpoints
    API->>+DB: EF Core → CatalogContext
    DB-->>-API: Data
    API-->>-Blazor: JSON Response
    Blazor-->>Browser: Update UI
```

### Concurrency Model

- **Web**: Single process, request-scoped DbContext per HTTP request
- **PublicApi**: Single process, request-scoped DbContext
- **Database**: Shared SQL Server; CatalogContext (catalog, basket, orders) and AppIdentityDbContext (identity) are separate databases
- No explicit background jobs or message queues in the codebase
- In-memory cache (IMemoryCache) for catalog brands/types/items—shared within process

### Basket Transfer on Login

When anonymous user logs in, `TransferBasketAsync` is invoked from `LoginModel.OnPostAsync` (after successful sign-in) via `TransferAnonymousBasketToUserAsync` to merge anonymous basket into user basket and delete anonymous basket.

```mermaid
sequenceDiagram
    participant User
    participant Login as LoginModel
    participant Transfer as TransferAnonymousBasketToUserAsync
    participant Basket as IBasketService
    participant DB as SQL Server

    User->>+Login: POST login (credentials)
    Login->>Login: SignInManager sign-in
    Login->>+Transfer: TransferAnonymousBasketToUserAsync
    Transfer->>+Basket: TransferBasketAsync(anonymousId, userId)
    Basket->>DB: Load anonymous basket
    Basket->>DB: Load user basket
    Basket->>Basket: Merge items (AddItem)
    Basket->>DB: Delete anonymous basket
    Basket->>DB: Save user basket
    DB-->>Basket: OK
    Basket-->>-Transfer: Done
    Transfer-->>-Login: Done
    Login-->>-User: Redirect
```

---

## 3. Development View

The development view describes the module structure and dependencies.

### Solution Structure

```mermaid
C4Container
    title Solution Structure - eShopOnWeb
    Container_Boundary(sln, "eShopOnWeb.sln") {
        Container_Boundary(src, "src/") {
            Container(web, "Web", "ASP.NET Core MVC + Razor Pages", "Storefront")
            Container(publicapi, "PublicApi", "ASP.NET Core Web API", "For Blazor Admin")
            Container(blazoradmin, "BlazorAdmin", "Blazor WebAssembly", "Admin UI")
            Container(blazorshared, "BlazorShared", "Shared DTOs", "Blazor + API")
            Container(appcore, "ApplicationCore", "Domain layer", "Entities, interfaces, services")
            Container(infra, "Infrastructure", "Data access", "EF Core, Identity")
        }
        Container_Boundary(tests, "tests/") {
            Container(unit, "UnitTests")
            Container(integration, "IntegrationTests")
            Container(functional, "FunctionalTests", "WebApplicationFactory")
            Container(apiint, "PublicApiIntegrationTests")
        }
    }
```

### Dependency Graph

```mermaid
C4Component
    title Project Dependencies
    Container_Boundary(sln, "eShopOnWeb") {
        Component(web, "Web", "ASP.NET Core")
        Component(blazoradmin, "BlazorAdmin", "Blazor WASM")
        Component(publicapi, "PublicApi", "ASP.NET Core API")
        Component(infra, "Infrastructure", "EF Core, Identity")
        Component(appcore, "ApplicationCore", "Domain")
        Component(blazorshared, "BlazorShared", "DTOs")
    }
    Rel(web, blazoradmin, "references")
    Rel(web, infra, "references")
    Rel(web, appcore, "references")
    Rel(blazoradmin, blazorshared, "references")
    Rel(infra, appcore, "references")
    Rel(appcore, blazorshared, "references")
    Rel(publicapi, infra, "references")
    Rel(publicapi, appcore, "references")
    Rel(publicapi, blazorshared, "references")
```

- **ApplicationCore**: No dependencies on Web, Infrastructure, or Blazor. References BlazorShared for shared types.
- **Infrastructure**: Depends on ApplicationCore. Implements IRepository, identity, EmailSender.
- **Web**: Depends on ApplicationCore, Infrastructure, BlazorAdmin, BlazorShared. Hosts storefront and serves Blazor Admin.
- **PublicApi**: Depends on ApplicationCore, Infrastructure, BlazorShared. Standalone API for Blazor Admin.
- **BlazorAdmin**: Depends on BlazorShared. Calls PublicApi via HttpClient.

### Key Technologies

| Layer | Technologies |
|-------|--------------|
| Web | ASP.NET Core 8, MVC, Razor Pages, Cookie Auth, MediatR, Blazor Server (admin host) |
| PublicApi | ASP.NET Core, Minimal API (MinimalApi.Endpoint), JWT Bearer, Swagger, AutoMapper |
| BlazorAdmin | Blazor WebAssembly, Blazored.LocalStorage, HttpClient |
| ApplicationCore | Ardalis.GuardClauses, Ardalis.Result, Ardalis.Specification |
| Infrastructure | EF Core, SQL Server, ASP.NET Core Identity |

### Configuration

- **Web**: appsettings.json, optional Azure Key Vault in production, BaseUrlConfiguration for PublicApi base URL
- **PublicApi**: appsettings.json, CORS from Web base URL, JWT secret (AuthorizationConstants.JWT_SECRET_KEY)

---

## 4. Physical View

The physical view describes deployment topology.

### Docker Compose (Local/Dev)

```mermaid
C4Deployment
    title Docker Compose - Local/Dev
    Deployment_Node(docker, "Docker Host", "Docker") {
        Deployment_Node(webnode, "eshopwebmvc", "ASP.NET Core", "port 5106") {
            Container(web, "Web", "ASP.NET Core MVC + Razor Pages")
        }
        Deployment_Node(apinode, "eshoppublicapi", "ASP.NET Core", "port 5200") {
            Container(api, "PublicApi", "REST API")
        }
        Deployment_Node(dbnode, "sqlserver", "Azure SQL Edge", "port 1433") {
            ContainerDb(db, "Database", "SQL Server", "CatalogContext + AppIdentityDbContext")
        }
    }
    Rel(web, db, "connects to")
    Rel(api, db, "connects to")
```

### Local Development (without Docker)

- **Web**: `dotnet run` from Web folder (or `--launch-profile Web`)
- **PublicApi**: `dotnet run` from PublicApi folder (required for admin)
- **SQL Server**: Local or SQL Server Express; two databases: catalog + identity
- **BlazorAdmin**: Served by Web at `/admin`; fetches from PublicApi

### Azure Deployment (azd)

- Uses `azd up` to provision and deploy
- Configuration from Azure Key Vault (connection strings)
- Resource group: `rg-{env name}`

### Data Stores

| Context | Database | Contents |
|---------|----------|----------|
| CatalogContext | SQL Server | CatalogItem, CatalogBrand, CatalogType, Basket, BasketItem, Order, OrderItem |
| AppIdentityDbContext | SQL Server | AspNetUsers, AspNetRoles, AspNetUserClaims, etc. |

---

## 5. Scenarios (+1)

Scenarios tie the four views together by showing how key use cases traverse the system.

### Scenario 1: Shopper Browses Catalog and Adds to Basket

```mermaid
sequenceDiagram
    participant Shopper
    participant Web
    participant CatalogVM as ICatalogViewModelService
    participant BasketSvc as IBasketService
    participant DB as SQL Server

    Shopper->>+Web: GET /
    Web->>+CatalogVM: Get catalog (filters, pagination)
    CatalogVM->>DB: CatalogFilterPaginatedSpecification
    DB-->>CatalogVM: CatalogItem[]
    CatalogVM-->>-Web: CatalogIndexViewModel
    Web-->>-Shopper: Products page
    Shopper->>+Web: POST add to basket
    Web->>+BasketSvc: AddItemToBasket
    BasketSvc->>DB: BasketWithItemsSpecification, Save
    DB-->>BasketSvc: OK
    BasketSvc-->>-Web: OK
    Web-->>-Shopper: Redirect to basket
```

| Step | Logical | Process | Development | Physical |
|------|---------|---------|-------------|----------|
| 1 | Shopper requests home page | HTTP GET / | Web Index page, ICatalogViewModelService | Browser → Web |
| 2 | Load catalog with filters | CatalogFilterPaginatedSpecification, IRepository<CatalogItem> | ApplicationCore + Infrastructure | Web → SQL Server |
| 3 | Display products | CatalogIndexViewModel, Razor | Web | Web → Browser |
| 4 | Add to basket | IBasketService.AddItemToBasket | ApplicationCore BasketService | Browser POST → Web |
| 5 | Persist basket | BasketWithItemsSpecification, IRepository<Basket> | Infrastructure EfRepository | Web → SQL Server |
| 6 | Redirect to basket | BasketViewModelService | Web | Web → Browser |

### Scenario 2: Authenticated Shopper Checks Out

```mermaid
sequenceDiagram
    participant Shopper
    participant Web
    participant BasketVM as IBasketViewModelService
    participant OrderSvc as IOrderService
    participant BasketSvc as IBasketService
    participant DB as SQL Server

    Shopper->>+Web: GET checkout (Authorize)
    Web->>BasketVM: Get basket
    BasketVM->>DB: Load basket
    DB-->>BasketVM: Basket
    BasketVM-->>Web: BasketViewModel
    Web-->>Shopper: Checkout form
    Shopper->>+Web: POST Pay Now
    Web->>+OrderSvc: CreateOrderAsync
    OrderSvc->>DB: Create order, CatalogItemsSpecification
    DB-->>OrderSvc: Order
    OrderSvc-->>-Web: Order
    Web->>BasketSvc: DeleteBasketAsync
    BasketSvc->>DB: Delete basket
    Web-->>-Shopper: Redirect to success
```

| Step | Logical | Process | Development | Physical |
|------|---------|---------|-------------|----------|
| 1 | Navigate to checkout | Authorize → redirect if not logged in | Web Basket/Checkout | Browser → Web |
| 2 | Load basket | IBasketViewModelService | Web | Web → SQL Server |
| 3 | Submit "Pay Now" | IOrderService.CreateOrderAsync | ApplicationCore OrderService | Browser POST → Web |
| 4 | Create order | Order aggregate, CatalogItemsSpecification | ApplicationCore + Infrastructure | Web → SQL Server |
| 5 | Delete basket | IBasketService.DeleteBasketAsync | ApplicationCore | Web → SQL Server |
| 6 | Redirect to success | Basket/Success page | Web | Web → Browser |

### Scenario 3: Admin Creates Catalog Item

```mermaid
sequenceDiagram
    participant Admin
    participant Blazor
    participant API as PublicApi
    participant DB as SQL Server

    Admin->>+Blazor: Login
    Blazor->>+API: POST /api/authenticate
    API->>API: ITokenClaimsService, SignInManager
    API-->>-Blazor: JWT
    Admin->>Blazor: Create catalog item form
    Admin->>+Blazor: Submit
    Blazor->>+API: POST /api/catalog-items (Bearer)
    API->>API: CreateCatalogItemEndpoint
    API->>DB: IRepository.SaveAsync(CatalogItem)
    DB-->>API: OK
    API-->>-Blazor: CatalogItemDto
    Blazor-->>-Admin: Show created item
```

| Step | Logical | Process | Development | Physical |
|------|---------|---------|-------------|----------|
| 1 | Admin logs in via Blazor | POST /api/authenticate | PublicApi AuthenticateEndpoint | Blazor → PublicApi |
| 2 | Receive JWT | ITokenClaimsService, SignInManager | Infrastructure | PublicApi → Blazor |
| 3 | Create catalog item form | ICatalogItemService | BlazorAdmin | Browser |
| 4 | Submit create | POST /api/catalog-items (Bearer token) | PublicApi CreateCatalogItemEndpoint | Blazor → PublicApi |
| 5 | Validate & persist | IRepository<CatalogItem>, CatalogItem | Infrastructure | PublicApi → SQL Server |
| 6 | Return created item | CatalogItemDto | PublicApi + BlazorShared | PublicApi → Blazor |

### Scenario 4: Anonymous User Logs In (Basket Transfer)

```mermaid
sequenceDiagram
    participant User
    participant Web
    participant Login as LoginModel
    participant BasketSvc as IBasketService
    participant DB as SQL Server

    User->>Web: Add items (anonymous, GUID buyerId cookie)
    Web->>BasketSvc: AddItemToBasket
    BasketSvc->>DB: Save anonymous basket
    User->>+Login: POST login
    Login->>Login: SignInManager sign-in
    Login->>+BasketSvc: TransferBasketAsync(anonymousId, userId)
    BasketSvc->>DB: Load both baskets
    BasketSvc->>BasketSvc: Merge items
    BasketSvc->>DB: Delete anonymous, save user basket
    DB-->>BasketSvc: OK
    BasketSvc-->>-Login: Done
    Login-->>-User: Redirect
```

| Step | Logical | Process | Development | Physical |
|------|---------|---------|-------------|----------|
| 1 | Anonymous adds items | Basket with GUID buyer ID (cookie) | Web, IBasketService | Browser → Web |
| 2 | User logs in | Identity/Account/Login | Web, SignInManager | Browser POST → Web |
| 3 | Sign-in succeeds | LoginModel.OnPostAsync → TransferAnonymousBasketToUserAsync | Web Identity/Account/Login | Web |
| 4 | Transfer basket | IBasketService.TransferBasketAsync | ApplicationCore BasketService | Web → SQL Server |
| 5 | Merge baskets | Anonymous basket items → user basket | Basket.AddItem | In-process |
| 6 | Delete anonymous basket | IRepository.DeleteAsync | Infrastructure | Web → SQL Server |

---

## Summary

| View | Summary |
|------|---------|
| **Logical** | Domain aggregates (Catalog, Basket, Order), services (Basket, Order), specifications, value objects |
| **Process** | Request/response over HTTP; Web + PublicApi as separate processes; shared SQL Server; in-memory cache |
| **Development** | Clean architecture: Core → Infrastructure; Web and PublicApi as delivery mechanisms; BlazorAdmin as SPA |
| **Physical** | Docker: Web (5106), PublicApi (5200), SQL Server (1433); optional Azure deployment |
| **Scenarios** | Browse → Add to basket → Checkout; Admin CRUD via PublicApi; basket transfer on login |
