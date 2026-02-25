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

1. Shopper goes to the store
2. System displays products with name, image, and price
3. System displays available brands and types for filtering
4. Shopper sees products, one page at a time

### Alternative Flows

#### A1: Shopper Narrows Results by Brand or Type

- **Trigger**: After step 4, shopper applies filters
- **Steps**:
  1. Shopper chooses brand and/or type and applies
  2. System displays filtered results; if no products match, system displays "THERE ARE NO RESULTS THAT MATCH YOUR SEARCH"

#### A2: Shopper Views Another Page of Results

- **Trigger**: After step 4, shopper wants to see more products
- **Steps**:
  1. Shopper goes to another page of results
  2. System displays the requested page of products

### Notes

- Code mapping: see Summary table

---

## UC-002: Add Item to Basket

**Actor**: Shopper (anonymous or authenticated)  
**Description**: A shopper adds a product to their shopping basket.

### Preconditions

- System is operational and database is accessible

### Basic Flow

1. Shopper looks at a product
2. Shopper requests to add the product to their basket
3. System retrieves the shopper's basket
4. System adds the product to the basket at its current price (or increases the quantity if the product is already in the basket)
5. System saves the basket
6. System takes shopper to their basket

### Alternative Flows

#### A1: Shopper Has No Basket Yet

- **Trigger**: In step 3, shopper has no basket
- **Steps**:
  1. System creates a new basket for the shopper
  2. Resume basic flow at step 4

#### A2: Product Not Found

- **Trigger**: In step 4, product does not exist or cannot be found
- **Steps**:
  1. System takes shopper back to the store

#### A3: Product Cannot Be Identified

- **Trigger**: In step 2, the product selection is invalid
- **Steps**:
  1. System takes shopper back to the store

### Notes

- Code mapping: see Summary table

---

## UC-003: View Basket

**Actor**: Shopper (anonymous or authenticated)  
**Description**: A shopper views the contents of their shopping basket.

### Preconditions

- System is operational and database is accessible

### Basic Flow

1. Shopper goes to their basket
2. System retrieves the shopper's basket
3. System displays the basket with each item (name, image, price), quantities, line totals, and grand total

### Alternative Flows

#### A1: Shopper Has No Basket Yet

- **Trigger**: In step 2, shopper has no basket
- **Steps**:
  1. System creates a new empty basket for the shopper
  2. System displays the empty basket with option to continue shopping

### Notes

- Code mapping: see Summary table

---

## UC-004: Update Basket Quantities

**Actor**: Shopper (anonymous or authenticated)  
**Description**: A shopper changes the quantity of items in their basket.

### Preconditions

- System is operational and database is accessible

### Basic Flow

1. Shopper changes the quantity of one or more items in their basket
2. Shopper confirms the changes
3. System checks that the quantities are valid
4. System retrieves the basket
5. System updates the quantities; any item set to zero is removed from the basket
6. System saves the basket
7. System displays the updated basket

### Alternative Flows

#### A1: Quantities Invalid

- **Trigger**: In step 3, the quantities entered are not valid
- **Steps**:
  1. System does not save changes
  2. System redisplays the basket and shows what needs to be corrected

#### A2: Basket Not Found

- **Trigger**: In step 4, basket does not exist
- **Steps**:
  1. System does not save changes; no update is performed

### Notes

- Code mapping: see Summary table

---

## UC-005: Checkout (Create Order)

**Actor**: Authenticated shopper  
**Description**: A shopper completes checkout by turning their basket into an order.

### Preconditions

- System is operational and database is accessible

### Basic Flow

1. Shopper goes to checkout
2. System displays basket contents for review with total
3. Shopper confirms the order and completes payment
4. System applies any last-minute quantity changes
5. System creates the order from the basket contents
6. System records the order and clears the basket
7. System takes shopper to the order confirmation

### Alternative Flows

#### A1: Empty Basket on Checkout

- **Trigger**: In step 5, basket has no items
- **Steps**:
  1. System takes shopper back to their basket

#### A2: Shopper Not Signed In

- **Trigger**: In step 1, shopper tries to checkout without being signed in
- **Steps**:
  1. System takes shopper to sign in
  2. After signing in, shopper may return to checkout

#### A3: Details Invalid

- **Trigger**: In step 4, the quantity or other details are not valid
- **Steps**:
  1. System redisplays checkout and shows what needs to be corrected

### Exception Flows

- **System Unable to Complete Order**: If a technical failure occurs during order creation, the shopper sees an error and the order is not completed

### Notes

- Code mapping: see Summary table

---

## UC-006: View My Orders

**Actor**: Authenticated shopper  
**Description**: A shopper views a list of their past orders.

### Preconditions

- System is operational and database is accessible

### Basic Flow

1. Shopper goes to their order history
2. System retrieves the shopper's orders
3. System displays the list of orders with date, items, and total for each

### Alternative Flows

#### A1: Shopper Not Signed In

- **Trigger**: In step 1, shopper is not signed in
- **Steps**:
  1. System takes shopper to sign in

### Notes

- Code mapping: see Summary table

---

## UC-007: View Order Details

**Actor**: Authenticated shopper  
**Description**: A shopper views details of a specific order.

### Preconditions

- System is operational and database is accessible

### Basic Flow

1. Shopper selects an order to view
2. System retrieves the order and its items
3. System displays order details (address, items, quantities, prices, total)

### Alternative Flows

#### A1: Order Not Found or Not Theirs

- **Trigger**: In step 2, order does not exist or does not belong to the shopper
- **Steps**:
  1. System displays message that the order could not be found

### Notes

- Code mapping: see Summary table

---

## UC-008: Sign In

**Actor**: Shopper  
**Description**: A shopper signs in using email and password.

### Preconditions

- System is operational and identity database is accessible

### Basic Flow

1. Shopper goes to sign in
2. System displays the sign-in page
3. Shopper enters email and password and signs in
4. System checks the email and password
5. System signs in the shopper and takes them to the store

### Alternative Flows

#### A1: Invalid Email or Password

- **Trigger**: In step 4, email or password is incorrect
- **Steps**:
  1. System redisplays sign-in page with error message

#### A2: Two-Factor Authentication Required

- **Trigger**: In step 4, shopper has two-factor authentication enabled
- **Steps**:
  1. System prompts for verification code
  2. Shopper provides code
  3. System verifies and completes sign-in

#### A3: Basket Transfer on Sign-In

- **Trigger**: In step 5, shopper had items in their basket before signing in (from a previous visit)
- **Steps**:
  1. System combines those items with the shopper's account basket
  2. System clears the temporary basket
  3. System takes shopper to the store

### Notes

- Code mapping: see Summary table

---

## UC-009: Register

**Actor**: Shopper  
**Description**: A new user creates an account.

### Preconditions

- System is operational and identity database is accessible

### Basic Flow

1. Shopper goes to create an account
2. System displays the registration page
3. Shopper enters their details and creates their account
4. System creates the account
5. System takes user to sign in or to the store

### Alternative Flows

#### A1: Email Confirmation Required

- **Trigger**: In step 4, the store requires email confirmation before signing in
- **Steps**:
  1. System sends a confirmation email
  2. System shows a message asking the user to check their email

### Notes

- Code mapping: see Summary table

---

## UC-010: Manage Account Profile

**Actor**: Authenticated user  
**Description**: A user updates their profile information (email, phone number).

### Preconditions

- System is operational and identity database is accessible

### Basic Flow

1. User goes to their account settings
2. System displays the user's current profile information
3. User changes email and/or phone number and saves
4. System saves the changes
5. System displays a success message and the updated profile

### Alternative Flows

#### A1: Email Update Cannot Be Completed

- **Trigger**: In step 4, the email change cannot be saved
- **Steps**:
  1. System redisplays the page with an error message

#### A2: Phone Update Cannot Be Completed

- **Trigger**: In step 4, the phone change cannot be saved
- **Steps**:
  1. System redisplays the page with an error message

### Notes

- Code mapping: see Summary table

---

## UC-011: Create Catalog Item (Admin)

**Actor**: Admin  
**Description**: An administrator adds a new product to the catalog.

### Preconditions

- System is operational and database is accessible

### Basic Flow

1. Admin goes to the product catalog
2. Admin requests to add a new product
3. Admin enters product details (name, description, price, brand, type, image)
4. Admin saves the new product
5. System checks that required fields are present and valid
6. System adds the product to the catalog
7. Admin sees the updated catalog with the new product

### Alternative Flows

#### A1: Not Authorized

- **Trigger**: In step 5, the user does not have permission to add products
- **Steps**:
  1. System denies the request and shows an error message

#### A2: Details Invalid

- **Trigger**: In step 5, required fields are missing or invalid (e.g., negative price)
- **Steps**:
  1. System redisplays the form and shows what needs to be corrected

### Notes

- Code mapping: see Summary table

---

## UC-012: Update Catalog Item (Admin)

**Actor**: Admin  
**Description**: An administrator updates an existing product in the catalog.

### Preconditions

- System is operational and database is accessible

### Basic Flow

1. Admin goes to the product catalog
2. Admin selects a product to edit
3. Admin changes the product details (name, description, price, brand, type, image)
4. Admin saves the changes
5. System saves the updated product
6. Admin sees the updated catalog

### Alternative Flows

#### A1: Product Not Found

- **Trigger**: In step 5, the product no longer exists
- **Steps**:
  1. System shows an error message that the product could not be found

### Notes

- Code mapping: see Summary table

---

## UC-013: Delete Catalog Item (Admin)

**Actor**: Admin  
**Description**: An administrator removes a product from the catalog.

### Preconditions

- System is operational and database is accessible

### Basic Flow

1. Admin goes to the product catalog
2. Admin selects a product to remove
3. Admin confirms removal
4. System removes the product from the catalog
5. Admin sees the updated catalog

### Alternative Flows

#### A1: Product Not Found

- **Trigger**: In step 4, the product no longer exists
- **Steps**:
  1. System shows an error message that the product could not be found

### Notes

- Code mapping: see Summary table

---

## UC-014: List Catalog Items (Admin API)

**Actor**: Admin (via Admin application)  
**Description**: Retrieve the list of products for the admin catalog view.

### Preconditions

- System is operational and database is accessible

### Basic Flow

1. Admin application requests the product list (optionally for a specific page)
2. System returns the list of products with details (name, description, price, image, brand, type)
3. Admin application displays the catalog

### Notes

- Code mapping: see Summary table

---

## UC-015: Authenticate (Public API)

**Actor**: Admin application (or API client)  
**Description**: Sign in to access the Admin API.

### Preconditions

- System is operational and identity database is accessible

### Basic Flow

1. Client provides email and password
2. System checks the email and password
3. System issues an access token (including admin permissions when applicable)
4. System returns the token and user information
5. Client uses the token for subsequent requests

### Alternative Flows

#### A1: Invalid Email or Password

- **Trigger**: In step 2, email or password is incorrect
- **Steps**:
  1. System denies the request

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
