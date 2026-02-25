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

### Basic Flow

1. Shopper navigates to the store home page
2. System loads catalog items with pagination
3. System loads available brands and types for filter dropdowns
4. System displays the catalog with product cards (name, image, price), filter controls, and pagination

### Alternative Flows

#### A1: Shopper Applies Filters

- **Trigger**: After step 4, shopper selects brand and/or type filters and submits
- **Steps**:
  1. Shopper selects filters and submits
  2. System reloads catalog with applied filters (resume basic flow at step 2)
  3. System displays filtered results; if no items match, system displays "THERE ARE NO RESULTS THAT MATCH YOUR SEARCH"

#### A2: Shopper Navigates to Another Page

- **Trigger**: After step 4, shopper requests a different page of results
- **Steps**:
  1. Shopper selects page
  2. System reloads catalog for requested page (resume basic flow at step 2)
  3. System displays the requested page of results

### Notes

- Code mapping: see Summary table

---

## UC-002: Add Item to Basket

**Actor**: Shopper (anonymous or authenticated)  
**Description**: A shopper adds a catalog item to their shopping basket.

### Preconditions

- System is operational and database is accessible

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

- Code mapping: see Summary table

---

## UC-003: View Basket

**Actor**: Shopper (anonymous or authenticated)  
**Description**: A shopper views the contents of their shopping basket.

### Preconditions

- System is operational and database is accessible

### Basic Flow

1. Shopper navigates to the basket page
2. System retrieves or creates a basket for the shopper
3. System loads basket items with catalog details (name, image, price)
4. System displays basket with items, quantities, line totals, and grand total

### Notes

- Code mapping: see Summary table

---

## UC-004: Update Basket Quantities

**Actor**: Shopper (anonymous or authenticated)  
**Description**: A shopper updates the quantity of items in their basket.

### Preconditions

- System is operational and database is accessible

### Basic Flow

1. Shopper modifies quantity inputs for one or more basket items
2. Shopper submits the update form
3. System validates submitted data
4. System retrieves the basket
5. System updates quantities for each item; items with quantity 0 are removed
6. System persists the basket
7. System displays the updated basket

### Alternative Flows

#### A1: Validation Fails

- **Trigger**: In step 3, submitted data is invalid
- **Steps**:
  1. System does not persist changes
  2. System re-displays basket with validation errors

#### A2: Basket Not Found

- **Trigger**: In step 4, basket does not exist
- **Steps**:
  1. System does not persist changes; no update is performed

### Notes

- Code mapping: see Summary table

---

## UC-005: Checkout (Create Order)

**Actor**: Authenticated shopper  
**Description**: A shopper completes checkout by converting their basket into an order.

### Preconditions

- System is operational and database is accessible

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

- **Trigger**: In step 6, basket has no items
- **Steps**:
  1. System redirects shopper to basket page

#### A2: Unauthenticated Access

- **Trigger**: In step 1, shopper navigates to checkout without being signed in
- **Steps**:
  1. System redirects shopper to login page
  2. After login, shopper may return to checkout

#### A3: Validation Fails

- **Trigger**: In step 5, submitted data is invalid
- **Steps**:
  1. System re-displays checkout form with errors

### Exception Flows

- **Database Connection Lost**: If database fails during order creation or basket deletion, exception propagates; user sees error

### Notes

- Code mapping: see Summary table

---

## UC-006: View My Orders

**Actor**: Authenticated shopper  
**Description**: A shopper views a list of their past orders.

### Preconditions

- System is operational and database is accessible

### Basic Flow

1. Shopper navigates to "My Orders"
2. System retrieves orders for the authenticated user
3. System displays list of orders with order details (date, items, total)

### Alternative Flows

#### A1: Unauthenticated Access

- **Trigger**: In step 1, shopper navigates without being signed in
- **Steps**:
  1. System redirects shopper to login page

### Notes

- Code mapping: see Summary table

---

## UC-007: View Order Details

**Actor**: Authenticated shopper  
**Description**: A shopper views details of a specific order.

### Preconditions

- System is operational and database is accessible

### Basic Flow

1. Shopper navigates to order detail page
2. System retrieves order with items for the authenticated user
3. System displays order details (address, items, quantities, prices, total)

### Alternative Flows

#### A1: Order Not Found or Unauthorized

- **Trigger**: In step 2, order does not exist or does not belong to the user
- **Steps**:
  1. System displays error message "No such order found for this user."

### Notes

- Code mapping: see Summary table

---

## UC-008: Sign In

**Actor**: Shopper  
**Description**: A shopper signs in to the application using email and password.

### Preconditions

- System is operational and identity database is accessible

### Basic Flow

1. Shopper navigates to login page
2. System displays sign-in form with email, password, and remember-me options
3. Shopper enters credentials and submits
4. System validates credentials against identity store
5. System signs in user and redirects to home page

### Alternative Flows

#### A1: Invalid Credentials

- **Trigger**: In step 4, credentials are invalid
- **Steps**:
  1. System redisplays login form with error message

#### A2: Two-Factor Authentication Enabled

- **Trigger**: In step 4, user has 2FA enabled
- **Steps**:
  1. System prompts for authenticator code
  2. User provides code
  3. System validates and completes sign-in

#### A3: Anonymous Basket Transfer

- **Trigger**: In step 5, user had items in an anonymous basket before signing in
- **Steps**:
  1. System merges anonymous basket items into user basket
  2. System removes anonymous basket
  3. System redirects user to home page

### Notes

- Code mapping: see Summary table

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
5. System redirects user to login or home page

### Alternative Flows

#### A1: Email Confirmation Required

- **Trigger**: In step 4, system is configured to require email confirmation
- **Steps**:
  1. System sends confirmation email
  2. System redirects user with message to check email

### Notes

- Code mapping: see Summary table

---

## UC-010: Manage Account Profile

**Actor**: Authenticated user  
**Description**: A user updates their profile information (email, phone number).

### Preconditions

- System is operational and identity database is accessible

### Basic Flow

1. User navigates to account profile page
2. System loads current user profile (username, email, phone, email confirmed status)
3. System displays profile form
4. User updates email and/or phone number and submits
5. System validates and persists changes
6. System displays success message and updated profile

### Alternative Flows

#### A1: Email Update Fails

- **Trigger**: In step 5, email update cannot be completed
- **Steps**:
  1. System redisplays form with error message

#### A2: Phone Update Fails

- **Trigger**: In step 5, phone update cannot be completed
- **Steps**:
  1. System redisplays form with error message

### Notes

- Code mapping: see Summary table

---

## UC-011: Create Catalog Item (Admin)

**Actor**: Admin  
**Description**: An administrator creates a new catalog item via the Admin UI.

### Preconditions

- System is operational and database is accessible

### Basic Flow

1. Admin navigates to admin catalog list
2. Admin clicks Create
3. Admin enters catalog item details (name, description, price, brand, type, picture URI)
4. Admin submits
5. Admin application submits create request to system
6. System validates request (name, price, image if provided)
7. System creates catalog item
8. System returns created item with ID
9. Admin sees updated catalog list

### Alternative Flows

#### A1: Unauthorized

- **Trigger**: In step 6, user does not have admin role
- **Steps**:
  1. System denies request; admin sees unauthorized message

#### A2: Validation Failure

- **Trigger**: In step 6, required fields missing or invalid (e.g., negative price)
- **Steps**:
  1. System returns validation errors; form redisplays with errors

### Notes

- Code mapping: see Summary table

---

## UC-012: Update Catalog Item (Admin)

**Actor**: Admin  
**Description**: An administrator updates an existing catalog item.

### Preconditions

- System is operational and database is accessible

### Basic Flow

1. Admin navigates to admin catalog list
2. Admin clicks Edit on a catalog item
3. Admin modifies fields (name, description, price, brand, type, picture)
4. Admin submits
5. Admin application submits update request to system
6. System loads existing item, applies updates, persists
7. System returns updated item
8. Admin sees updated catalog list

### Alternative Flows

#### A1: Catalog Item Not Found

- **Trigger**: In step 6, catalog item does not exist
- **Steps**:
  1. System returns not found; admin sees error message

### Notes

- Code mapping: see Summary table

---

## UC-013: Delete Catalog Item (Admin)

**Actor**: Admin  
**Description**: An administrator deletes a catalog item.

### Preconditions

- System is operational and database is accessible

### Basic Flow

1. Admin navigates to admin catalog list
2. Admin clicks Delete on a catalog item
3. Admin confirms deletion
4. Admin application submits delete request to system
5. System deletes catalog item
6. Admin sees updated catalog list

### Alternative Flows

#### A1: Catalog Item Not Found

- **Trigger**: In step 5, catalog item does not exist
- **Steps**:
  1. System returns not found; admin sees error message

### Notes

- Code mapping: see Summary table

---

## UC-014: List Catalog Items (Admin API)

**Actor**: Admin (via Admin application)  
**Description**: Retrieve paginated catalog items for admin display.

### Preconditions

- System is operational and database is accessible

### Basic Flow

1. Admin application requests catalog list with optional page index and page size
2. System returns paged list of catalog items (ID, name, description, price, picture URI, brand, type)
3. Admin application displays catalog list

### Notes

- Code mapping: see Summary table

---

## UC-015: Authenticate (Public API)

**Actor**: Admin application (or API client)  
**Description**: Obtain authentication token for Admin API access.

### Preconditions

- System is operational and identity database is accessible

### Basic Flow

1. Client submits email and password
2. System validates credentials
3. System generates token with user claims (including admin role if applicable)
4. System returns token and user info
5. Client uses token for subsequent API calls

### Alternative Flows

#### A1: Invalid Credentials

- **Trigger**: In step 2, credentials are invalid
- **Steps**:
  1. System denies request; client receives unauthorized response

### Notes

- Code mapping: see Summary table

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
