# Use Case Specifications

> **Reverse-engineered** from the eShopOnWeb codebase. These specifications document observed behavior, flows, terminology, and exception handling as implemented.

**Related documentation**:
- [4+1 Architecture Views](architecture-4plus1.md) – Logical, process, development, and physical views
- [Practices Guide](practices-guide.md) – BDD, TDD, and use case documentation practices

---

## UC-001: Browse Catalog

**Actor**: Shopper (anonymous or authenticated)  
**Description**: A shopper views the product catalog with optional filtering and pagination.

### Preconditions

- System is operational and catalog database is accessible
- Application server is running

### Basic Flow

1. Shopper navigates to the store home page
2. System loads catalog items from the catalog repository, filtered by brand and type if specified, with pagination
3. System loads available brands and types for filter dropdowns
4. System displays the catalog with product cards (name, image, price), filter controls, and pagination
5. Shopper may optionally select brand and/or type filters and submit to refine results
6. Shopper may navigate between pages of results

### Alternative Flows

#### A1: No Results Match Filters

- **Trigger**: In step 2, no catalog items match the applied filters
- **Steps**:
  1. System returns empty catalog list
  2. System displays "THERE ARE NO RESULTS THAT MATCH YOUR SEARCH"

### Notes

- Catalog uses in-memory caching when configured (CachedCatalogViewModelService)
- Items per page is configurable (Constants.ITEMS_PER_PAGE)
- Picture URIs are composed via IUriComposer (base URL + path)

---

## UC-002: Add Item to Basket

**Actor**: Shopper (anonymous or authenticated)  
**Description**: A shopper adds a catalog item to their shopping basket.

### Preconditions

- System is operational and database is accessible
- Catalog item exists and is available
- Shopper has a basket identifier (cookie for anonymous users, username for authenticated users)

### Basic Flow

1. Shopper views a product on the catalog page
2. Shopper clicks "Add to Basket" on the product
3. System retrieves or creates a basket for the shopper (by buyer ID from cookie or username)
4. System loads the catalog item to obtain its price
5. System adds the item to the basket (or increments quantity if item already in basket)
6. System persists the basket
7. System redirects shopper to the basket page

### Alternative Flows

#### A1: Catalog Item Not Found

- **Trigger**: In step 4, catalog item does not exist
- **Steps**:
  1. System redirects shopper to the catalog home page

#### A2: Invalid Product Details

- **Trigger**: In step 2, product ID is null or missing
- **Steps**:
  1. System redirects shopper to the catalog home page

### Notes

- Anonymous shoppers receive a GUID-based basket cookie; authenticated users use their username as buyer ID
- Basket transfer occurs when an anonymous user logs in (IBasketService.TransferBasketAsync)
- Quantity defaults to 1 when adding

---

## UC-003: View Basket

**Actor**: Shopper (anonymous or authenticated)  
**Description**: A shopper views the contents of their shopping basket.

### Preconditions

- System is operational and database is accessible
- Shopper has a basket identifier (cookie or authenticated session)

### Basic Flow

1. Shopper navigates to the basket page
2. System retrieves or creates a basket for the shopper
3. System loads basket items with catalog details (name, image, price)
4. System displays basket with items, quantities, line totals, and grand total
5. Shopper may update quantities or proceed to checkout

### Notes

- Empty basket displays with option to continue shopping
- Basket component also appears in layout for quick summary

---

## UC-004: Update Basket Quantities

**Actor**: Shopper (anonymous or authenticated)  
**Description**: A shopper updates the quantity of items in their basket.

### Preconditions

- System is operational and database is accessible
- Shopper has a non-empty basket
- Shopper is on the basket page

### Basic Flow

1. Shopper modifies quantity inputs for one or more basket items
2. Shopper submits the update form
3. System validates model state
4. System retrieves the basket
5. System updates quantities for each item; items with quantity 0 are removed
6. System persists the basket
7. System displays the updated basket

### Alternative Flows

#### A1: Model State Invalid

- **Trigger**: In step 3, model validation fails
- **Steps**:
  1. System does not persist changes
  2. System re-displays basket with validation errors

#### A2: Basket Not Found

- **Trigger**: In step 4, basket does not exist
- **Steps**:
  1. System returns Result.NotFound; no update is performed

### Notes

- Quantity of 0 removes the item (RemoveEmptyItems)
- Unit tests verify update to 0 empties basket (IndexTest.OnPostUpdateTo0EmptyBasket)

---

## UC-005: Checkout (Create Order)

**Actor**: Authenticated shopper  
**Description**: A shopper completes checkout by converting their basket into an order.

### Preconditions

- System is operational and database is accessible
- Shopper is authenticated
- Shopper has a non-empty basket

### Basic Flow

1. Shopper navigates to checkout page from basket
2. System loads basket for authenticated user
3. System displays basket contents for review with total
4. Shopper confirms and clicks "Pay Now"
5. System updates basket quantities from form (if any changes)
6. System creates order from basket: loads catalog items, builds order items with snapshot data (name, picture URI, price, quantity)
7. System adds order to order repository
8. System deletes the basket
9. System redirects shopper to order success page

### Alternative Flows

#### A1: Empty Basket on Checkout

- **Trigger**: In step 6, basket has no items (EmptyBasketOnCheckoutException)
- **Steps**:
  1. System catches EmptyBasketOnCheckoutException
  2. System logs warning
  3. System redirects shopper to basket page

#### A2: Unauthenticated Access

- **Trigger**: Shopper navigates to checkout without being signed in
- **Steps**:
  1. System redirects to login page (Authorize attribute on Checkout page)
  2. After login, shopper may return to checkout

#### A3: Model State Invalid

- **Trigger**: In step 5, model validation fails
- **Steps**:
  1. System returns BadRequest
  2. Checkout page re-displays with errors

### Exception Flows

- **Database Connection Lost**: If database fails during order creation or basket deletion, exception propagates; user sees error

### Notes

- Shipping address is currently hardcoded in implementation ("123 Main St.", Kent, OH, etc.)—simplification for reference app
- No payment processing is implemented; "Pay Now" completes the order immediately
- Order is created with buyer ID from authenticated user

---

## UC-006: View My Orders

**Actor**: Authenticated shopper  
**Description**: A shopper views a list of their past orders.

### Preconditions

- System is operational and database is accessible
- Shopper is authenticated

### Basic Flow

1. Shopper navigates to "My Orders" (OrderController.MyOrders)
2. System retrieves orders for the authenticated user (CustomerOrdersSpecification)
3. System displays list of orders with order details (date, items, total)
4. Shopper may select an order to view details

### Alternative Flows

#### A1: Unauthenticated Access

- **Trigger**: Shopper navigates without being signed in
- **Steps**:
  1. System redirects to login page (Authorize on OrderController)

### Notes

- Uses MediatR (GetMyOrders / GetMyOrdersHandler)
- Guard.Against.Null ensures User.Identity.Name is present

---

## UC-007: View Order Details

**Actor**: Authenticated shopper  
**Description**: A shopper views details of a specific order.

### Preconditions

- System is operational and database is accessible
- Shopper is authenticated
- Order exists and belongs to the shopper

### Basic Flow

1. Shopper navigates to order detail (OrderController.Detail with orderId)
2. System retrieves order with items for the authenticated user
3. System displays order details (address, items, quantities, prices, total)
4. If order not found or does not belong to user, system returns BadRequest

### Alternative Flows

#### A1: Order Not Found or Unauthorized

- **Trigger**: In step 2, order does not exist or buyer ID does not match
- **Steps**:
  1. System returns BadRequest with message "No such order found for this user."

---

## UC-008: Sign In

**Actor**: Shopper  
**Description**: A shopper signs in to the application using email and password.

### Preconditions

- System is operational and identity database is accessible
- User account exists with confirmed email (for standard login)

### Basic Flow

1. Shopper navigates to login page
2. System displays sign-in form with email, password, and remember-me options
3. Shopper enters credentials and submits
4. System validates credentials against identity store
5. System signs in user and redirects to home page
6. If anonymous basket exists, system transfers it to user basket (on next basket operation)

### Alternative Flows

#### A1: Invalid Credentials

- **Trigger**: In step 4, credentials are invalid
- **Steps**:
  1. System redisplays login form with error message

#### A2: Two-Factor Authentication Enabled

- **Trigger**: User has 2FA enabled
- **Steps**:
  1. System prompts for authenticator code
  2. User provides code
  3. System validates and completes sign-in

### Notes

- Uses ASP.NET Core Identity with cookie authentication
- Default demo credentials: demouser@microsoft.com / Pass@word1
- Functional tests cover ReturnsSignInScreenOnGet, ReturnsSuccessfulSignInOnPostWithValidCredentials

---

## UC-009: Register

**Actor**: Shopper  
**Description**: A new user creates an account.

### Preconditions

- System is operational and identity database is accessible

### Basic Flow

1. Shopper navigates to register page
2. System displays registration form (email, password, confirm password)
3. Shopper submits valid registration data
4. System creates user account
5. System may require email confirmation depending on configuration
6. System redirects user accordingly (login or confirmation message)

### Notes

- Uses ASP.NET Core Identity RegisterModel
- Email confirmation flow via ConfirmEmail page

---

## UC-010: Manage Account Profile

**Actor**: Authenticated user  
**Description**: A user updates their profile information (email, phone number).

### Preconditions

- System is operational and identity database is accessible
- User is authenticated

### Basic Flow

1. User navigates to Manage/MyAccount
2. System loads current user profile (username, email, phone, email confirmed status)
3. System displays profile form
4. User updates email and/or phone number and submits
5. System validates and persists changes
6. System displays success message and updated profile

### Alternative Flows

#### A1: Email Update Fails

- **Trigger**: In step 5, SetEmailAsync fails
- **Steps**:
  1. System throws ApplicationException with error details

#### A2: Phone Update Fails

- **Trigger**: In step 5, SetPhoneNumberAsync fails
- **Steps**:
  1. System throws ApplicationException with error details

### Notes

- Functional test UpdatePhoneNumberProfile verifies phone number update
- User may also send verification email, change password, manage 2FA, external logins (ManageController)

---

## UC-011: Create Catalog Item (Admin)

**Actor**: Admin  
**Description**: An administrator creates a new catalog item via the Blazor Admin UI.

### Preconditions

- System is operational and database is accessible
- Admin is authenticated with admin role (JWT token with admin claim)
- Public API is running (Blazor Admin communicates with PublicApi)

### Basic Flow

1. Admin navigates to admin catalog list
2. Admin clicks Create
3. Admin enters catalog item details (name, description, price, brand, type, picture URI)
4. Admin submits
5. Blazor Admin sends POST to PublicApi /api/catalog-items
6. PublicApi validates request (name, price, image file if provided)
7. System creates catalog item in repository
8. System returns created item with ID
9. Admin sees updated catalog list

### Alternative Flows

#### A1: Unauthorized (Normal User)

- **Trigger**: In step 6, user token does not have admin role
- **Steps**:
  1. PublicApi returns 401 Unauthorized

#### A2: Validation Failure

- **Trigger**: In step 6, required fields missing or invalid (e.g., negative price)
- **Steps**:
  1. PublicApi returns 400 Bad Request with validation errors

### Notes

- CreateCatalogItemEndpoint in PublicApi
- Blazor Admin uses ICatalogItemService (HttpService to PublicApi)
- Integration test: ReturnsSuccessGivenValidNewItemAndAdminUserToken

---

## UC-012: Update Catalog Item (Admin)

**Actor**: Admin  
**Description**: An administrator updates an existing catalog item.

### Preconditions

- System is operational and database is accessible
- Admin is authenticated with admin role
- Catalog item exists

### Basic Flow

1. Admin navigates to admin catalog list
2. Admin clicks Edit on a catalog item
3. Admin modifies fields (name, description, price, brand, type, picture)
4. Admin submits
5. Blazor Admin sends PUT to PublicApi /api/catalog-items
6. PublicApi loads existing item, applies updates, persists
7. System returns updated item
8. Admin sees updated catalog list

### Alternative Flows

#### A1: Catalog Item Not Found

- **Trigger**: In step 6, catalog item ID does not exist
- **Steps**:
  1. PublicApi returns 404 Not Found

---

## UC-013: Delete Catalog Item (Admin)

**Actor**: Admin  
**Description**: An administrator deletes a catalog item.

### Preconditions

- System is operational and database is accessible
- Admin is authenticated with admin role
- Catalog item exists

### Basic Flow

1. Admin navigates to admin catalog list
2. Admin clicks Delete on a catalog item
3. Admin confirms deletion
4. Blazor Admin sends DELETE to PublicApi /api/catalog-items/{id}
5. PublicApi deletes catalog item from repository
6. Admin sees updated catalog list

### Alternative Flows

#### A1: Catalog Item Not Found

- **Trigger**: In step 5, catalog item ID does not exist
- **Steps**:
  1. PublicApi returns 404 Not Found

### Notes

- Integration tests: ReturnsSuccessGivenValidIdAndAdminUserToken, ReturnsNotFoundGivenInvalidIdAndAdminUserToken

---

## UC-014: List Catalog Items (Admin API)

**Actor**: Admin (Blazor Admin application)  
**Description**: Retrieve paginated catalog items for admin display.

### Preconditions

- System is operational and database is accessible
- Public API is running

### Basic Flow

1. Blazor Admin requests GET /api/catalog-items with optional page index and page size
2. PublicApi applies CatalogFilterPaginatedSpecification
3. System returns paged list of catalog items (ID, name, description, price, picture URI, brand, type)
4. Blazor Admin displays catalog list

### Notes

- CatalogItemListPagedEndpoint
- Integration test: ReturnsFirst10CatalogItems, ReturnsCorrectCatalogItemsGivenPageIndex1

---

## UC-015: Authenticate (Public API)

**Actor**: Blazor Admin (or API client)  
**Description**: Obtain JWT token for Public API authentication.

### Preconditions

- System is operational and identity database is accessible
- Public API is running
- User account exists

### Basic Flow

1. Client sends POST to /api/authenticate with email and password
2. PublicApi validates credentials via SignInManager
3. System generates JWT token with user claims (including admin role if applicable)
4. System returns token and user info
5. Client uses token in Authorization header for subsequent API calls

### Alternative Flows

#### A1: Invalid Credentials

- **Trigger**: In step 2, credentials are invalid
- **Steps**:
  1. PublicApi returns 401 Unauthorized

### Notes

- AuthenticateEndpoint in PublicApi
- Integration test: AuthenticateEndpointTest with DataRow for valid/invalid credentials

---

## Summary: Use Case to Code Mapping

| Use Case | Primary Entry Point | Key Services |
|----------|---------------------|--------------|
| UC-001 Browse Catalog | Index.cshtml.cs, ICatalogViewModelService | CatalogViewModelService, IRepository\<CatalogItem\> |
| UC-002 Add to Basket | Basket/Index OnPost | IBasketService, BasketViewModelService |
| UC-003 View Basket | Basket/Index OnGet | IBasketViewModelService |
| UC-004 Update Basket | Basket/Index OnPostUpdate | IBasketService.SetQuantities |
| UC-005 Checkout | Basket/Checkout OnPost | IOrderService, IBasketService |
| UC-006 View My Orders | OrderController.MyOrders | GetMyOrdersHandler |
| UC-007 View Order Details | OrderController.Detail | GetOrderDetailsHandler |
| UC-008 Sign In | Identity/Account/Login | SignInManager |
| UC-009 Register | Identity/Account/Register | UserManager |
| UC-010 Manage Profile | ManageController.MyAccount | UserManager |
| UC-011–013 Catalog CRUD (Admin) | Blazor Admin → PublicApi | CreateCatalogItemEndpoint, UpdateCatalogItemEndpoint, DeleteCatalogItemEndpoint |
| UC-014 List Catalog (API) | PublicApi CatalogItemListPagedEndpoint | IRepository\<CatalogItem\> |
| UC-015 Authenticate (API) | PublicApi AuthenticateEndpoint | SignInManager, ITokenClaimsService |
