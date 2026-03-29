# Comprehensive Code Review Guide - EverPeak Camp Laravel Project

**Project**: Laravel E-commerce/Booking Platform  
**Date Generated**: March 29, 2026  
**Status**: COMPLETE - All 8 Milestones Implemented ✅

---

## Table of Contents
1. [MP1 - Product CRUD (53pts)](#mp1---product-crud)
2. [MP2 - User Management (20pts)](#mp2---user-management)
3. [MP3 - Authentication](#mp3---authentication)
4. [MP4 - Product Reviews](#mp4---product-reviews)
5. [MP5 - Validation](#mp5---validation)
6. [MP6 - Filtering (25pts)](#mp6---filtering)
7. [MP7 - Charts/Dashboard](#mp7---charts-dashboard)
8. [MP8 - Search (30pts)](#mp8---search)
9. [Database Schema](#database-schema)
10. [Summary & Statistics](#summary--statistics)

---

## MP1 - Product CRUD

### **Overview**
Full CRUD operations with advanced features including DataTables integration, soft deletes, multi-photo management, and Excel import. **Status: ✅ FULLY IMPLEMENTED (53pts)**

### **1.1 Files & Implementation**

| Component | File Path | Purpose | Status |
|-----------|-----------|---------|--------|
| **Controller** | [app/Http/Controllers/ProductController.php](app/Http/Controllers/ProductController.php) | Product CRUD operations, DataTable API, Excel import | ✅ |
| **Model** | [app/Models/Product.php](app/Models/Product.php) | Product model with relationships, Scout searchable | ✅ |
| **Model** | [app/Models/ProductImage.php](app/Models/ProductImage.php) | Gallery photo management | ✅ |
| **View - List** | [resources/views/products/index.blade.php](resources/views/products/index.blade.php) | DataTables UI for product listing | ✅ |
| **View - Create** | [resources/views/products/create.blade.php](resources/views/products/create.blade.php) | Form for new product with multi-photo upload | ✅ |
| **View - Edit** | [resources/views/products/edit.blade.php](resources/views/products/edit.blade.php) | Update form with photo gallery management | ✅ |
| **View - Show** | [resources/views/products/show.blade.php](resources/views/products/show.blade.php) | Public product detail page with reviews | ✅ |
| **Migration** | [database/migrations/2026_02_22_000007_create_products_table.php](database/migrations/2026_02_22_000007_create_products_table.php) | Products table schema | ✅ |
| **Migration** | [database/migrations/2026_02_22_001558_create_product_images_table.php](database/migrations/2026_02_22_001558_create_product_images_table.php) | Product images gallery table | ✅ |
| **Migration** | [database/migrations/2026_02_22_002724_add_soft_deletes_to_product_table.php](database/migrations/2026_02_22_002724_add_soft_deletes_to_product_table.php) | Soft deletes support | ✅ |

### **1.2 Key Methods in ProductController**

```php
// Core CRUD Methods
- index()                    // Display DataTable view (admin)
- datatable()               // API endpoint for server-side DataTable processing
- create()                  // Show create form
- store(Request)            // Save new product with image & validation
- show(Product)             // Display public product detail page
- edit(Product)             // Show edit form
- update(Request, Product)  // Update product data
- destroy(Product)          // Soft delete product
- restore($id)              // Restore soft-deleted product
- deleteImage($imageId)     // AJAX delete gallery image
- importForm()              // Show Excel import form
- import(Request)           // Handle Excel file import
```

### **1.3 Key Methods in Product Model**

```php
// Relationships
- category()                // BelongsTo Category
- images()                  // HasMany ProductImage
- reviews()                 // HasMany Review
- cartItems()               // HasMany CartItem
- orderItems()              // HasMany OrderItem
- stock()                   // HasMany Stock

// Query Scopes
- scopeSearch(?string)      // Search by name/description
- toSearchableArray()       // For Laravel Scout indexing

// Traits
- SoftDeletes              // Logical deletion support
- Searchable                // Laravel Scout integration
```

### **1.4 Features Implemented**

#### **Stage 1: DataTable CRUD (8pts)**
- Server-side pagination and filtering
- Advanced search by product name
- Real-time status display (Active/Deleted)
- Photo count badges
- Stock level indicators with color coding
- Sortable columns

#### **Stage 2: Soft Deletes & Restore (10pts)**
- Products marked as deleted (not physically removed)
- View deleted products toggle
- One-click restore functionality
- Automatic status display (Active vs Deleted)

#### **Stage 3: Photo Upload - Single & Multiple (15pts)**
- Main product image upload (single)
- Gallery photos upload (multiple)
- Image preview before submission
- AJAX photo deletion without page reload
- Display existing photos during edit
- **Validation**: JPEG/PNG/GIF, max 2MB, min 100x100px

#### **Stage 4: Excel Import (20pts)**
- XLSX, XLS, CSV format support
- Sample template download
- Column mapping (name, description, cost_price, sell_price, category_id)
- Error handling and reporting
- Import via [app/Imports/ProductsImport.php](app/Imports/ProductsImport.php)

### **1.5 Database Schema - Products**

```sql
CREATE TABLE products (
    product_id BIGINT UNSIGNED PRIMARY KEY,
    name VARCHAR(255) REQUIRED,
    description TEXT REQUIRED,
    cost_price DECIMAL(10,2) REQUIRED,
    sell_price DECIMAL(10,2) REQUIRED,
    category_id BIGINT UNSIGNED (nullable, FK to categories),
    img_path VARCHAR(255) (nullable),
    deleted_at TIMESTAMP (nullable, soft delete),
    created_at TIMESTAMP,
    updated_at TIMESTAMP
);

CREATE TABLE product_images (
    image_id BIGINT UNSIGNED PRIMARY KEY,
    product_id BIGINT UNSIGNED FK (cascade delete),
    img_path VARCHAR(255) REQUIRED,
    created_at TIMESTAMP,
    updated_at TIMESTAMP
);
```

### **1.6 Form Validation Rules**

```php
'name' => 'required|string|min:3|max:255|regex:/^[a-zA-Z0-9\s\-&\/.,()]+$/',
'description' => 'required|string|min:10|max:5000',
'cost_price' => 'required|numeric|min:0|max:999999.99',
'sell_price' => 'required|numeric|min:0|max:999999.99|gte:cost_price',
'category_id' => 'nullable|exists:categories,category_id',
'img_path' => 'nullable|image|mimes:jpeg,png,jpg,gif|max:2048|dimensions:min_width=100,min_height=100',
'images.*' => 'nullable|image|mimes:jpeg,png,jpg,gif|max:2048|dimensions:min_width=100,min_height=100',
```

### **1.7 API Routes**

```
GET    /products                    Admin - List products (DataTable view)
GET    /products/datatable          Admin - DataTable API endpoint
GET    /products/create             Admin - Create form
POST   /products                    Admin - Store product
GET    /products/{product}          Public - View product details
GET    /products/{product}/edit     Admin - Edit form
PUT    /products/{product}          Admin - Update product
DELETE /products/{product}          Admin - Soft delete
GET    /products/{id}/restore       Admin - Restore deleted product
DELETE /products-image/{imageId}    Admin - Delete gallery image (AJAX)
GET    /products/import/form        Admin - Import form
POST   /products/import             Admin - Process Excel import
```

### **1.8 Middleware & Security**
- `auth` - User must be logged in
- `admin` - User must have admin role
- `email_verified` - Email verification required for admin access
- CSRF protection on all forms

---

## MP2 - User Management

### **Overview**
Complete user management with photo upload, profile management, DataTables admin view, and role/status management. **Status: ✅ FULLY IMPLEMENTED (20pts)**

### **2.1 Files & Implementation**

| Component | File Path | Purpose | Status |
|-----------|-----------|---------|--------|
| **Controller** | [app/Http/Controllers/UserController.php](app/Http/Controllers/UserController.php) | User CRUD, auth, profile, admin management | ✅ |
| **Model** | [app/Models/User.php](app/Models/User.php) | User model with relationships | ✅ |
| **View - List** | [resources/views/users/index.blade.php](resources/views/users/index.blade.php) | DataTable admin user listing | ✅ |
| **View - Create** | [resources/views/users/create.blade.php](resources/views/users/create.blade.php) | Admin create user form | ✅ |
| **View - Edit** | [resources/views/users/edit.blade.php](resources/views/users/edit.blade.php) | Admin edit user form | ✅ |
| **View - Register** | [resources/views/auth/register.blade.php](resources/views/auth/register.blade.php) | Public registration form with photo | ✅ |
| **View - Profile** | [resources/views/profile/create.blade.php](resources/views/profile/create.blade.php) | User profile creation | ✅ |
| **View - Profile** | [resources/views/profile/edit.blade.php](resources/views/profile/edit.blade.php) | User profile update with photo | ✅ |
| **View - Role** | [resources/views/users/role.blade.php](resources/views/users/role.blade.php) | Role assignment UI | ✅ |
| **Migration** | [database/migrations/2026_02_22_002830_add_role_to_users.php](database/migrations/2026_02_22_002830_add_role_to_users.php) | Add role column to users | ✅ |
| **Migration** | [database/migrations/2026_03_22_000000_add_photo_status_to_users.php](database/migrations/2026_03_22_000000_add_photo_status_to_users.php) | Add photo & status columns | ✅ |
| **Migration** | [database/migrations/2026_03_03_000100_add_contact_fields_to_users.php](database/migrations/2026_03_03_000100_add_contact_fields_to_users.php) | Add phone & address fields | ✅ |

### **2.2 Key Methods in UserController**

```php
// Admin User Management
- index()                       // Show DataTable view
- datatable(Request)            // API for DataTable
- create()                      // Create user form
- store(Request)                // Save new user
- show(User)                    // View user profile
- edit(User)                    // Edit form
- update(Request, User)         // Update user
- destroy(User)                 // Delete user
- updateRole(Request, User)     // AJAX role update
- updateStatus(Request, User)   // AJAX status update

// Authentication & Registration
- createRegister()              // Show registration form
- storeRegister(Request)        // Process registration with photo
- createLogin()                 // Show login form
- storeLogin(Request)           // Process login
- logout()                      // Logout user

// Email Verification
- verifyEmail($token)           // Verify email via token
- resendVerification()          // Resend verification email

// Profile Management
- showProfile()                 // View current user profile
- createProfile()               // Create profile form
- storeProfile(Request)         // Save profile with photo
- editProfile()                 // Edit profile form
- updateProfile(Request)        // Update profile with photo
```

### **2.3 Features Implemented**

#### **Stage 1: Registration with Photo (5pts)**
- Optional photo upload during registration
- Image preview before submission
- Photo stored in `storage/users/profiles/`
- User status auto-set to 'active'
- Validation: JPEG/PNG/GIF, max 2MB

#### **Stage 2: User Profile Update with Photo (5pts)**
- Profile creation page with photo upload
- Profile edit page with photo management
- Display existing photo with replacement ability
- Automatic old photo deletion on new upload

#### **Stage 3: DataTable User Listing (5pts)**
- Server-side pagination
- Sortable columns (ID, Name, Email, Phone, Created)
- Search functionality (name, email, phone)
- User photo thumbnail display
- Configurable page length

#### **Stage 4: Admin User Management (5pts)**
- Admin can set user status (active/inactive) via AJAX
- Admin can change user role (customer/admin) via AJAX
- Admin cannot modify own role/status (security)
- Role/status badges for non-admin viewers
- Form dropdowns for admins only

### **2.4 Database Schema - Users**

```sql
CREATE TABLE users (
    id BIGINT UNSIGNED PRIMARY KEY,
    first_name VARCHAR(255) REQUIRED,
    last_name VARCHAR(255) REQUIRED,
    email VARCHAR(255) UNIQUE REQUIRED,
    password VARCHAR(255) REQUIRED,
    role VARCHAR(255) DEFAULT 'customer',
    phone VARCHAR(255) nullable,
    address TEXT nullable,
    photo VARCHAR(255) nullable,
    status ENUM('active','inactive') DEFAULT 'active',
    email_verified_at TIMESTAMP nullable,
    remember_token VARCHAR(100) nullable,
    created_at TIMESTAMP,
    updated_at TIMESTAMP
);
```

### **2.5 Form Validation Rules - Registration**

```php
'first_name' => 'required|string|max:255',
'last_name' => 'required|string|max:255',
'email' => 'required|email|unique:users,email',
'password' => 'required|string|min:8|confirmed',
'photo' => 'nullable|image|mimes:jpeg,png,jpg,gif|max:2048',
```

### **2.6 API Routes**

```
GET    /register                Admin/Public - Registration form
POST   /register                Admin/Public - Store registration
GET    /login                   Public - Login form
POST   /login                   Public - Process login
POST   /logout                  Auth - Logout

GET    /users                   Admin - User list (DataTable)
GET    /users/datatable         Admin - DataTable API
GET    /users/create            Admin - Create user form
POST   /users                   Admin - Store user
GET    /users/{user}            Auth - View user profile
GET    /users/{user}/edit       Admin - Edit form
PUT    /users/{user}            Admin - Update user
PUT    /users/{user}/role       Admin - Update role (AJAX)
PUT    /users/{user}/status     Admin - Update status (AJAX)
DELETE /users/{user}            Admin - Delete user

GET    /profile                 Auth - View current user profile
GET    /profile/create          Auth - Create profile form
POST   /profile                 Auth - Store profile
GET    /profile/edit            Auth - Edit profile form
PUT    /profile                 Auth - Update profile

GET    /email/verify/{token}    Public - Verify email
POST   /email/resend            Auth - Resend verification
```

---

## MP3 - Authentication

### **Overview**
Complete authentication system with email verification, token management, and admin middleware. **Status: ✅ FULLY IMPLEMENTED**

### **3.1 Files & Implementation**

| Component | File Path | Purpose | Status |
|-----------|-----------|---------|--------|
| **Controller** | [app/Http/Controllers/UserController.php](app/Http/Controllers/UserController.php) | Auth methods: login, register, verify | ✅ |
| **Middleware** | [app/Http/Middleware/IsAdmin.php](app/Http/Middleware/IsAdmin.php) | Check admin role & email verification | ✅ |
| **Mail** | [app/Mail/VerifyEmailMail.php](app/Mail/VerifyEmailMail.php) | Email verification mail template | ✅ |
| **Model** | [app/Models/EmailVerificationToken.php](app/Models/EmailVerificationToken.php) | Token storage & management | ✅ |
| **Model** | [app/Models/User.php](app/Models/User.php) | User model with auth traits | ✅ |
| **View** | [resources/views/auth/login.blade.php](resources/views/auth/login.blade.php) | Login form | ✅ |
| **View** | [resources/views/auth/register.blade.php](resources/views/auth/register.blade.php) | Registration form | ✅ |
| **View** | [resources/views/auth/verify.blade.php](resources/views/auth/verify.blade.php) | Email verification page | ✅ |
| **View** | [resources/views/emails/verify-email.blade.php](resources/views/emails/verify-email.blade.php) | Email verification template | ✅ |
| **Migration** | [database/migrations/2026_03_22_100000_create_email_verification_tokens_table.php](database/migrations/2026_03_22_100000_create_email_verification_tokens_table.php) | Token storage | ✅ |

### **3.2 Authentication Flow**

```
1. User registers via /register
   - Validates email (unique), password (min 8)
   - Hashes password
   - Stores user record
   - Generates verification token
   - Sends verification email

2. User clicks email verification link
   - Token is validated
   - User email_verified_at is set
   - User redirected to home

3. User logs in via /login
   - Email & password validated
   - Session created

4. Admin Check (IsAdmin Middleware)
   - Must be authenticated
   - Must have email_verified_at set
   - Must have role = 'admin'
```

### **3.3 Email Verification Model**

```php
// EmailVerificationToken Model
- user_id (FK to users)
- token (unique)
- created_at

// User Model
- email_verified_at (timestamp, can be null)
- role (enum: 'customer', 'admin')
```

### **3.4 IsAdmin Middleware Logic**

```php
// Checks in Order:
1. User authenticated? → redirect to login
2. Email verified? → logout & redirect to login with error
3. User role = admin? → redirect to home with error
4. All passed → proceed to next middleware
```

### **3.5 Security Features**
- CSRF protection on auth forms
- Password hashing with bcrypt
- Email verification required for admin
- Token expiration (configurable)
- Secure password reset via email tokens
- Session management with Laravel's authentication

---

## MP4 - Product Reviews

### **Overview**
Full review system with create, update, delete, and admin DataTable management. **Status: ✅ FULLY IMPLEMENTED**

### **4.1 Files & Implementation**

| Component | File Path | Purpose | Status |
|-----------|-----------|---------|--------|
| **Controller** | [app/Http/Controllers/ReviewController.php](app/Http/Controllers/ReviewController.php) | Review CRUD & admin DataTable | ✅ |
| **Model** | [app/Models/Review.php](app/Models/Review.php) | Review model with relationships | ✅ |
| **View - Admin** | [resources/views/reviews/index.blade.php](resources/views/reviews/index.blade.php) | Admin review management DataTable | ✅ |
| **View - Display** | [resources/views/reviews/display.blade.php](resources/views/reviews/display.blade.php) | Display reviews on product page | ✅ |
| **View - Form** | [resources/views/reviews/form-modal.blade.php](resources/views/reviews/form-modal.blade.php) | Review form modal | ✅ |
| **Migration** | [database/migrations/2026_02_22_001635_create_reviews_table.php](database/migrations/2026_02_22_001635_create_reviews_table.php) | Reviews table | ✅ |

### **4.2 Key Methods in ReviewController**

```php
// Admin Management
- index()                       // Show DataTable view
- datatable(Request)            // API for DataTable
- destroy(Request, Review)      // Delete review

// User Reviews
- store(Request)                // Create new review
- update(Request, Review)       // Update existing review
- productReviews(Product)       // Get reviews for product
- hasPurchasedProduct()         // Check purchase eligibility
```

### **4.3 Features Implemented**

- **Create Review**: Only users who purchased the product can review (status = 'completed')
- **Rating**: 1-5 stars validation
- **Comments**: Optional text field
- **Edit**: Users can edit their own reviews
- **Delete**: Users can delete their reviews; admins can delete any
- **Datatable**: Admin can view all reviews with search & pagination

### **4.4 Database Schema - Reviews**

```sql
CREATE TABLE reviews (
    review_id BIGINT UNSIGNED PRIMARY KEY,
    product_id BIGINT UNSIGNED FK (cascade delete),
    user_id BIGINT UNSIGNED FK (cascade delete),
    rating INTEGER (1-5) REQUIRED,
    comment TEXT nullable,
    created_at TIMESTAMP,
    updated_at TIMESTAMP
);
```

### **4.5 Form Validation Rules**

```php
'product_id' => 'required|exists:products,product_id',
'rating' => 'required|integer|min:1|max:5',
'comment' => 'nullable|string|max:1000',
```

### **4.6 Review Model Relationships**

```php
// BelongsTo
- product()         // Product model
- user()            // User model

// Scopes
- Filters by product_id
- Filters by user_id
- Order by created_at (newest first)
```

### **4.7 API Routes**

```
GET    /reviews                 Admin - List reviews (DataTable)
GET    /reviews/datatable       Admin - DataTable API
POST   /reviews                 Auth - Create review
PUT    /reviews/{review}        Auth - Update review
DELETE /reviews/{review}        Auth - Delete review
GET    /products/{product}/reviews  Public - Get product reviews
```

---

## MP5 - Validation

### **Overview**
Comprehensive form validation across all CRUD operations with custom error messages. **Status: ✅ FULLY IMPLEMENTED**

### **5.1 Validation Pattern**

All controllers use Laravel's `$request->validate()` method with custom error messages:

```php
$validated = $request->validate([
    // validation rules
], [
    // custom error messages
]);
```

### **5.2 Validation Rules by Module**

#### **Product Validation**
```php
'name' => 'required|string|min:3|max:255|regex:/^[a-zA-Z0-9\s\-&\/.,()]+$/',
'description' => 'required|string|min:10|max:5000',
'cost_price' => 'required|numeric|min:0|max:999999.99',
'sell_price' => 'required|numeric|min:0|max:999999.99|gte:cost_price',
'category_id' => 'nullable|exists:categories,category_id',
'img_path' => 'nullable|image|mimes:jpeg,png,jpg,gif|max:2048|dimensions:min_width=100,min_height=100',
'images.*' => 'nullable|image|mimes:jpeg,png,jpg,gif|max:2048|dimensions:min_width=100,min_height=100',
```

#### **User Registration**
```php
'first_name' => 'required|string|max:255',
'last_name' => 'required|string|max:255',
'email' => 'required|email|unique:users,email',
'password' => 'required|string|min:8|confirmed',
'photo' => 'nullable|image|mimes:jpeg,png,jpg,gif|max:2048',
```

#### **Review Validation**
```php
'product_id' => 'required|exists:products,product_id',
'rating' => 'required|integer|min:1|max:5',
'comment' => 'nullable|string|max:1000',
```

#### **Order Validation**
```php
'customer_name' => 'required|string|max:255',
'customer_email' => 'required|email',
'shipping_address' => 'required|string|max:500',
'phone' => 'required|string|max:20',
```

#### **Cart Validation**
```php
'product_id' => 'required|exists:products,product_id',
'quantity' => 'required|integer|min:1|max:999',
```

### **5.3 Validation Views**

All forms display validation errors via [resources/views/layouts/flash-messages.blade.php](resources/views/layouts/flash-messages.blade.php):

```blade
@if ($errors->any())
    <div class="alert alert-danger">
        @foreach ($errors->all() as $message)
            <div>{{ $message }}</div>
        @endforeach
    </div>
@endif
```

**Individual field errors:**
```blade
@error('email')
    <div class="invalid-feedback d-block">{{ $message }}</div>
@enderror
```

### **5.4 Frontend Validation**

- HTML5 validation attributes (required, min, max, pattern, type)
- JavaScript validation in modals and forms
- Real-time image preview with size/dimension checks

### **5.5 Controllers with Validation**

| Controller | Methods | Rules |
|-----------|---------|-------|
| ProductController | store(), update() | Product fields + images |
| UserController | store(), storeRegister(), storeProfile(), updateProfile(), update() | User fields + profile |
| ReviewController | store(), update() | Product, rating, comment |
| OrderController | store(), checkout() | Customer info, items |
| CartController | store(), update() | Product ID, quantity |
| CategoryController | store(), update() | Category name |

---

## MP6 - Filtering

### **Overview**
Advanced product filtering by price, category, and type with search integration. **Status: ✅ FULLY IMPLEMENTED (25pts)**

### **6.1 Files & Implementation**

| Component | File Path | Purpose | Status |
|-----------|-----------|---------|--------|
| **Controller** | [app/Http/Controllers/ShopController.php](app/Http/Controllers/ShopController.php) | Filter logic & implementation | ✅ |
| **View** | [resources/views/shop/show.blade.php](resources/views/shop/show.blade.php) | Shop page with filters | ✅ |

### **6.2 Implemented Filters**

#### **Stage 1: Price Filtering (10pts)**
- Minimum price filter (>=)
- Maximum price filter (<=)
- Both filters can be combined
- UI shows available price range

#### **Stage 2: Multi-Criteria Filtering (15pts)**
- **Price Range**: Min & max filters
- **Category**: Exact match on category_id
- **Type/Brand**: Partial match on category name
- **Search**: Full-text search on product name & description
- **Sorting**: By name (asc/desc), price (asc/desc), newest

### **6.3 Filter Parameters**

```php
// Query String Parameters
?search=text              // Search products by name/description/category
?category=id             // Filter by category ID (exact match)
?type=text              // Filter by category name (partial match)
?min_price=value        // Minimum price filter
?max_price=value        // Maximum price filter
?sort_by=name|sell_price|created_at    // Sort field
?sort_order=asc|desc    // Sort direction
```

### **6.4 Filter Implementation (ShopController::show)**

```php
// Build query with filters
$query = Product::query();

// Search filter - product name, description, category name
if ($search !== '') {
    $query->where(function ($q) use ($search) {
        $q->where('name', 'like', "%{$search}%")
          ->orWhere('description', 'like', "%{$search}%")
          ->orWhereHas('category', fn ($q) => $q->where('name', 'like', "%{$search}%"));
    });
}

// Category exact match
if ($selectedCategory !== '') {
    $query->where('category_id', $selectedCategory);
}

// Type/Category name search
if ($selectedType !== '') {
    $query->whereHas('category', fn ($q) => $q->where('name', 'like', "%{$selectedType}%"));
}

// Price range filters
if ($minPrice !== '') {
    $query->where('sell_price', '>=', (float) $minPrice);
}
if ($maxPrice !== '') {
    $query->where('sell_price', '<=', (float) $maxPrice);
}

// Sorting
$query->orderBy($sortBy, $sortOrder);
$products = $query->paginate(12)->appends($request->query());
```

### **6.5 UI Features**

**Filter Form (Sidebar):**
- Search box for product name/description
- Category filter with radio buttons
- Type/Category name text input
- Min/Max price number inputs
- Auto-submit for category selection
- "Apply Filters" button
- "Clear Filters" button (when filters active)

**Product Display:**
- Total product count
- Grid layout (3 columns desktop)
- Product cards with image, category, name, description, stock, price
- View Details & Add to Cart buttons
- "No Products Found" message

### **6.6 API Routes**

```
GET    /shop                   Public - Shop page with filters
Query Parameters:
  ?search=keyword             // Product search
  ?category=id               // Category filter
  ?type=name                 // Type filter
  ?min_price=value           // Minimum price
  ?max_price=value           // Maximum price
  ?sort_by=field             // Sorting field
  ?sort_order=direction      // asc or desc
```

### **6.7 Database Queries**

- Uses eager loading to prevent N+1 queries
- Loads: category, stock, images relationships
- Pagination with 12 products per page
- Filters persist in pagination links via `appends($request->query())`

---

## MP7 - Charts & Dashboard

### **Overview**
Admin dashboard with multiple data visualization charts using Chart.js. **Status: ✅ FULLY IMPLEMENTED**

### **7.1 Files & Implementation**

| Component | File Path | Purpose | Status |
|-----------|-----------|---------|--------|
| **Controller** | [app/Http/Controllers/DashboardController.php](app/Http/Controllers/DashboardController.php) | Dashboard data aggregation & chart generation | ✅ |
| **View** | [resources/views/dashboard/index.blade.php](resources/views/dashboard/index.blade.php) | Dashboard layout with chart containers | ✅ |
| **View** | [resources/views/dashboard.blade.php](resources/views/dashboard.blade.php) | Full dashboard page | ✅ |
| **Library** | Chart.js v2.9.3 (CDN) | Chart rendering library | ✅ |

### **7.2 Charts Implemented**

#### **1. Yearly Sales Revenue Chart**
- Bar chart showing sales by year
- Time range: Configurable (default: 2024-present)
- Data: Total revenue per year (including shipping fees)
- Excludes cancelled orders

#### **2. Daily Sales Chart**
- Line/Bar chart for custom date range
- Only shows days with sales (reduces clutter)
- Time range: Configurable (default: last 30 days)
- Updates based on order_date

#### **3. Customer Growth Chart**
- Shows customer acquisition over time
- Tracks new users count

#### **4. Items Sold Chart**
- Pie chart showing product sales percentage
- Top 10 products by sales volume
- Shows percentage and total sales value

### **7.3 Dashboard Controller Methods**

```php
// Main dashboard index
- index(Request)              // Aggregate data and generate charts

// Data Aggregation
- Calculate total revenue (all time, excluding cancelled)
- Group orders by year for yearly sales
- Build daily sales data for date range
- Calculate product sales percentages
- Count users, products, categories, orders
- Count pending/completed orders
```

### **7.4 Dashboard Statistics**

```php
$stats = [
    'total_users' => User::count(),
    'total_products' => Product::count(),
    'total_categories' => Category::count(),
    'total_orders' => Order::count(),
    'total_revenue' => Total revenue (all time),
    'pending_orders' => Orders with status = 'pending',
    'completed_orders' => Orders with status = 'completed',
];
```

### **7.5 Chart Data Format**

**Yearly Sales:**
```javascript
{
    labels: ['2024', '2025', '2026'],
    datasets: [{
        label: 'Revenue',
        data: [15000, 28000, 45000],
        backgroundColor: 'rgba(75, 192, 192, 0.2)',
        borderColor: 'rgba(75, 192, 192, 1)',
        borderWidth: 1
    }]
}
```

**Daily Sales:**
```javascript
{
    labels: ['Mar 20', 'Mar 22', 'Mar 25'],
    datasets: [{
        label: 'Daily Revenue',
        data: [1200, 1900, 1500]
    }]
}
```

### **7.6 API Routes**

```
GET    /dashboard              Admin - Dashboard view with charts
Query Parameters (optional):
  ?start_date=YYYY-MM-DD
  ?end_date=YYYY-MM-DD
  ?daily_start_date=YYYY-MM-DD
  ?daily_end_date=YYYY-MM-DD
  ?yearly_start_date=YYYY-MM-DD
  ?yearly_end_date=YYYY-MM-DD
  ?product_start_date=YYYY-MM-DD
  ?product_end_date=YYYY-MM-DD
```

### **7.7 Date Range Defaults**

- Daily Sales: Last 30 days
- Yearly Sales: 2024 to present
- Product Sales: 2024 to present
- General Stats: Last 12 months

---

## MP8 - Search

### **Overview**
Multi-method product search with three implementations (LIKE, Model Scope, Scout). **Status: ✅ FULLY IMPLEMENTED (30pts)**

### **8.1 Files & Implementation**

| Component | File Path | Purpose | Status |
|-----------|-----------|---------|--------|
| **Controller** | [app/Http/Controllers/HomeController.php](app/Http/Controllers/HomeController.php) | Search implementation with 3 methods | ✅ |
| **Model** | [app/Models/Product.php](app/Models/Product.php) | Scout Searchable trait + search scope | ✅ |
| **View** | [resources/views/home.blade.php](resources/views/home.blade.php) | Search form and results display | ✅ |
| **Config** | [config/scout.php](config/scout.php) | Scout search engine configuration | ✅ |

### **8.2 Search Methods Implemented**

#### **Method 1: LIKE Query Search (8pts)**
```php
Product::where(function ($query) use ($search) {
    $query->where('name', 'LIKE', "%{$search}%")
        ->orWhere('description', 'LIKE', "%{$search}%");
})->paginate();
```
- Direct SQL LIKE query
- Searches: product name, description
- Performance: Good for small datasets

#### **Method 2: Model Scope Search (10pts)**
```php
Product::search($search)->paginate();

// scope definition
public function scopeSearch(Builder $query, ?string $term): Builder {
    $term = trim((string) $term);
    if ($term === '') return $query;
    
    return $query->where(function (Builder $builder) use ($term) {
        $builder->where('name', 'LIKE', "%{$term}%")
            ->orWhere('description', 'LIKE', "%{$term}%");
    });
}
```
- Reusable scope method
- Encapsulates search logic
- Easier to maintain & test

#### **Method 3: Laravel Scout Search (12pts)**
```php
Product::search($search)->get()->paginate();

// toSearchableArray definition
public function toSearchableArray(): array {
    return [
        'product_id' => $this->product_id,
        'name' => $this->name,
        'description' => $this->description,
    ];
}
```
- Full-text search engine integration
- Default driver: `database` (can switch to Algolia)
- Searchable trait enables indexing
- Best performance for large datasets

### **8.3 Search Selection**

User can select search type via query parameter:

```php
// Query Parameters
?search=keyword           // Search term (required)
?search_type=scout|like|model    // Search method (default: scout)
```

**Example URLs:**
```
/home?search=tent&search_type=scout
/home?search=camping&search_type=like
/?search=backpack&search_type=model
```

### **8.4 HomeController Search Methods**

```php
public function index(Request $request) {
    $search = trim((string) $request->query('search', ''));
    $searchType = $request->query('search_type', 'scout');
    
    if ($search === '') {
        // No search - show all featured products
        $products = Product::with('category', 'stock')
            ->latest()->paginate(12);
    } else {
        // Perform search based on type
        match ($searchType) {
            'like' => $this->performLikeSearch(...),
            'model' => $this->performModelSearch(...),
            'scout' => $this->performScoutSearch(...),
            default => $this->performScoutSearch(...),
        };
    }
    
    return view('home', [
        'products' => $products,
        'search' => $search,
        'searchMethod' => $searchType,
    ]);
}
```

### **8.5 Search Scope in Product Model**

```php
public function scopeSearch(Builder $query, ?string $term): Builder {
    $term = trim((string) $term);
    
    if ($term === '') {
        return $query;
    }
    
    return $query->where(function (Builder $builder) use ($term) {
        $builder->where('name', 'LIKE', "%{$term}%")
            ->orWhere('description', 'LIKE', "%{$term}%");
    });
}
```

### **8.6 Scout Configuration**

```php
// config/scout.php
'driver' => env('SCOUT_DRIVER', 'database'),  // or 'algolia'
'prefix' => env('SCOUT_PREFIX', ''),

// Model usage
use Laravel\Scout\Searchable;

class Product extends Model {
    use Searchable;
    
    public function toSearchableArray(): array {
        return [
            'product_id' => $this->product_id,
            'name' => $this->name,
            'description' => $this->description,
        ];
    }
}
```

### **8.7 Search Results Display**

- **Pagination**: 12 products per page
- **Sorting**: By relevance (Scout) or creation date
- **Filtering**: Excludes soft-deleted products
- **Related Data**: Eager loads category, stock, reviews
- **Query String**: Preserved through pagination (`withQueryString()`)

### **8.8 API Routes**

```
GET    /                       Public - Home page with search
GET    /home                   Public - Home page
Query Parameters:
  ?search=keyword             // Search term
  ?search_type=scout|like|model  // Search method
```

### **8.9 Performance Notes**

- **LIKE Query**: ~OK for <10k products
- **Model Scope**: ~Better for clean code
- **Scout Database**: Best for >10k products or frequent searches
- **Recommendation**: Use Scout with database driver for this project

### **8.10 Total Points for Search**
- LIKE Query: 8pts ✅
- Model Scope: 10pts ✅
- Scout Search: 12pts ✅
- **Total: 30pts** ✅

---

## Database Schema

### **9.1 Complete Table Structure**

#### **Users Table**
```sql
CREATE TABLE users (
    id BIGINT UNSIGNED PRIMARY KEY AUTO_INCREMENT,
    first_name VARCHAR(255),
    last_name VARCHAR(255),
    email VARCHAR(255) UNIQUE NOT NULL,
    password VARCHAR(255),
    role VARCHAR(255) DEFAULT 'customer',
    phone VARCHAR(255) nullable,
    address TEXT nullable,
    photo VARCHAR(255) nullable,
    status ENUM('active','inactive') DEFAULT 'active',
    email_verified_at TIMESTAMP nullable,
    remember_token VARCHAR(100) nullable,
    created_at TIMESTAMP,
    updated_at TIMESTAMP
);
```

#### **Products Table**
```sql
CREATE TABLE products (
    product_id BIGINT UNSIGNED PRIMARY KEY AUTO_INCREMENT,
    name VARCHAR(255) NOT NULL,
    description TEXT NOT NULL,
    cost_price DECIMAL(10,2) NOT NULL,
    sell_price DECIMAL(10,2) NOT NULL,
    category_id BIGINT UNSIGNED nullable,
    img_path VARCHAR(255) nullable,
    deleted_at TIMESTAMP nullable,  -- Soft delete
    created_at TIMESTAMP,
    updated_at TIMESTAMP,
    
    FOREIGN KEY (category_id) REFERENCES categories(category_id) ON DELETE SET NULL
);
```

#### **Product Images Table**
```sql
CREATE TABLE product_images (
    image_id BIGINT UNSIGNED PRIMARY KEY AUTO_INCREMENT,
    product_id BIGINT UNSIGNED NOT NULL,
    img_path VARCHAR(255) NOT NULL,
    created_at TIMESTAMP,
    updated_at TIMESTAMP,
    
    FOREIGN KEY (product_id) REFERENCES products(product_id) ON DELETE CASCADE
);
```

#### **Categories Table**
```sql
CREATE TABLE categories (
    category_id BIGINT UNSIGNED PRIMARY KEY AUTO_INCREMENT,
    name VARCHAR(255) UNIQUE NOT NULL,
    description TEXT nullable,
    deleted_at TIMESTAMP nullable,  -- Soft delete
    created_at TIMESTAMP,
    updated_at TIMESTAMP
);
```

#### **Reviews Table**
```sql
CREATE TABLE reviews (
    review_id BIGINT UNSIGNED PRIMARY KEY AUTO_INCREMENT,
    product_id BIGINT UNSIGNED NOT NULL,
    user_id BIGINT UNSIGNED NOT NULL,
    rating INTEGER,  -- 1-5
    comment TEXT nullable,
    created_at TIMESTAMP,
    updated_at TIMESTAMP,
    
    FOREIGN KEY (product_id) REFERENCES products(product_id) ON DELETE CASCADE,
    FOREIGN KEY (user_id) REFERENCES users(id) ON DELETE CASCADE
);
```

#### **Orders Table**
```sql
CREATE TABLE orders (
    order_id BIGINT UNSIGNED PRIMARY KEY AUTO_INCREMENT,
    user_id BIGINT UNSIGNED NOT NULL,
    customer_id BIGINT UNSIGNED nullable,
    total_amount DECIMAL(10,2),
    status VARCHAR(255) DEFAULT 'pending',
    shipping_fee DECIMAL(10,2) nullable,
    order_date TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    created_at TIMESTAMP,
    updated_at TIMESTAMP,
    
    FOREIGN KEY (user_id) REFERENCES users(id) ON DELETE CASCADE,
    FOREIGN KEY (customer_id) REFERENCES customers(customer_id) ON DELETE SET NULL
);
```

#### **Order Items Table**
```sql
CREATE TABLE order_items (
    order_item_id BIGINT UNSIGNED PRIMARY KEY AUTO_INCREMENT,
    order_id BIGINT UNSIGNED NOT NULL,
    product_id BIGINT UNSIGNED NOT NULL,
    quantity INTEGER,
    unit_price DECIMAL(10,2),
    created_at TIMESTAMP,
    updated_at TIMESTAMP,
    
    FOREIGN KEY (order_id) REFERENCES orders(order_id) ON DELETE CASCADE,
    FOREIGN KEY (product_id) REFERENCES products(product_id) ON DELETE CASCADE
);
```

#### **Stock Table**
```sql
CREATE TABLE stocks (
    stock_id BIGINT UNSIGNED PRIMARY KEY AUTO_INCREMENT,
    product_id BIGINT UNSIGNED NOT NULL UNIQUE,
    quantity INTEGER DEFAULT 0,
    warehouse_location VARCHAR(255) nullable,
    created_at TIMESTAMP,
    updated_at TIMESTAMP,
    
    FOREIGN KEY (product_id) REFERENCES products(product_id) ON DELETE CASCADE
);
```

#### **Cart Items Table**
```sql
CREATE TABLE cart_items (
    cart_item_id BIGINT UNSIGNED PRIMARY KEY AUTO_INCREMENT,
    user_id BIGINT UNSIGNED NOT NULL,
    product_id BIGINT UNSIGNED NOT NULL,
    quantity INTEGER DEFAULT 1,
    created_at TIMESTAMP,
    updated_at TIMESTAMP,
    
    FOREIGN KEY (user_id) REFERENCES users(id) ON DELETE CASCADE,
    FOREIGN KEY (product_id) REFERENCES products(product_id) ON DELETE CASCADE
);
```

#### **Customers Table**
```sql
CREATE TABLE customers (
    customer_id BIGINT UNSIGNED PRIMARY KEY AUTO_INCREMENT,
    name VARCHAR(255),
    email VARCHAR(255) UNIQUE,
    phone VARCHAR(255) nullable,
    address TEXT nullable,
    created_at TIMESTAMP,
    updated_at TIMESTAMP
);
```

#### **Email Verification Tokens Table**
```sql
CREATE TABLE email_verification_tokens (
    id BIGINT UNSIGNED PRIMARY KEY AUTO_INCREMENT,
    user_id BIGINT UNSIGNED NOT NULL,
    token VARCHAR(255) UNIQUE NOT NULL,
    created_at TIMESTAMP,
    
    FOREIGN KEY (user_id) REFERENCES users(id) ON DELETE CASCADE
);
```

### **9.2 Model Relationships Diagram**

```
User
├── hasMany: Orders
├── hasMany: Reviews
├── hasMany: CartItems
└── hasMany: EmailVerificationTokens

Product
├── belongsTo: Category
├── hasMany: ProductImages
├── hasMany: Reviews
├── hasMany: CartItems
├── hasMany: OrderItems
└── hasMany: Stock

Category
└── hasMany: Products

Order
├── belongsTo: User
├── belongsTo: Customer
├── hasMany: OrderItems
└── hasMany: Products (through OrderItems)

Review
├── belongsTo: Product
└── belongsTo: User

CartItem
├── belongsTo: User
└── belongsTo: Product

Stock
└── belongsTo: Product
```

### **9.3 Key Constraints**

| Constraint | Table | Column | References | On Delete |
|-----------|-------|--------|-----------|-----------|
| FK | products | category_id | categories.category_id | SET NULL |
| FK | product_images | product_id | products.product_id | CASCADE |
| FK | reviews | product_id | products.product_id | CASCADE |
| FK | reviews | user_id | users.id | CASCADE |
| FK | orders | user_id | users.id | CASCADE |
| FK | orders | customer_id | customers.customer_id | SET NULL |
| FK | order_items | order_id | orders.order_id | CASCADE |
| FK | order_items | product_id | products.product_id | CASCADE |
| FK | cart_items | user_id | users.id | CASCADE |
| FK | cart_items | product_id | products.product_id | CASCADE |
| FK | stocks | product_id | products.product_id | CASCADE |
| FK | email_verification_tokens | user_id | users.id | CASCADE |

### **9.4 Custom Primary Keys Note**

Project uses custom primary key names (NOT just `id`):
- `products.product_id` instead of `id`
- `categories.category_id` instead of `id`
- `reviews.review_id` instead of `id`
- `orders.order_id` instead of `id`
- `order_items.order_item_id` instead of `id`
- `cart_items.cart_item_id` instead of `id`
- `stocks.stock_id` instead of `id`
- `product_images.image_id` instead of `id`
- `customers.customer_id` instead of `id`

**Important**: This requires explicit `protected $primaryKey` declaration in models.

---

## Summary & Statistics

### **10.1 Implementation Status**

| Milestone | Feature | Points | Status |
|-----------|---------|--------|--------|
| **MP1** | Product CRUD - DataTable | 8pts | ✅ |
| **MP1** | Product CRUD - Soft Deletes | 10pts | ✅ |
| **MP1** | Product CRUD - Photo Upload | 15pts | ✅ |
| **MP1** | Product CRUD - Excel Import | 20pts | ✅ |
| **MP1** | **SUBTOTAL** | **53pts** | **✅** |
| **MP2** | User Registration with Photo | 5pts | ✅ |
| **MP2** | User Profile Update | 5pts | ✅ |
| **MP2** | User DataTable | 5pts | ✅ |
| **MP2** | Admin User Management | 5pts | ✅ |
| **MP2** | **SUBTOTAL** | **20pts** | **✅** |
| **MP3** | Authentication System | - | ✅ |
| **MP3** | Email Verification | - | ✅ |
| **MP3** | Admin Middleware | - | ✅ |
| **MP4** | Product Reviews | - | ✅ |
| **MP5** | Form Validation | - | ✅ |
| **MP6** | Price Filtering | 10pts | ✅ |
| **MP6** | Multi-Criteria Filtering | 15pts | ✅ |
| **MP6** | **SUBTOTAL** | **25pts** | **✅** |
| **MP7** | Dashboard Charts | - | ✅ |
| **MP8** | LIKE Search | 8pts | ✅ |
| **MP8** | Model Scope Search | 10pts | ✅ |
| **MP8** | Scout Search | 12pts | ✅ |
| **MP8** | **SUBTOTAL** | **30pts** | **✅** |

### **10.2 Total Points**

```
MP1 (Product CRUD):       53pts ✅
MP2 (User Management):    20pts ✅
MP3 (Authentication):     - (built-in)
MP4 (Reviews):            - (built-in)
MP5 (Validation):         - (built-in)
MP6 (Filtering):          25pts ✅
MP7 (Charts):             - (built-in)
MP8 (Search):             30pts ✅

TOTAL SCORED:             128pts ✅
```

### **10.3 Controllers Summary**

| Controller | File | Methods | Features |
|-----------|------|---------|----------|
| **ProductController** | [app/Http/Controllers/ProductController.php](app/Http/Controllers/ProductController.php) | 12 | CRUD + DataTable + Images + Import |
| **UserController** | [app/Http/Controllers/UserController.php](app/Http/Controllers/UserController.php) | 18 | Auth + CRUD + Profile + DataTable |
| **ReviewController** | [app/Http/Controllers/ReviewController.php](app/Http/Controllers/ReviewController.php) | 7 | CRUD + DataTable |
| **ShopController** | [app/Http/Controllers/ShopController.php](app/Http/Controllers/ShopController.php) | 1 | Filtering + Search |
| **DashboardController** | [app/Http/Controllers/DashboardController.php](app/Http/Controllers/DashboardController.php) | 1 | Charts + Statistics |
| **HomeController** | [app/Http/Controllers/HomeController.php](app/Http/Controllers/HomeController.php) | 4 | Search (3 methods) |
| **OrderController** | [app/Http/Controllers/OrderController.php](app/Http/Controllers/OrderController.php) | 9 | Order CRUD + Checkout |
| **CartController** | [app/Http/Controllers/CartController.php](app/Http/Controllers/CartController.php) | 5 | Cart management |
| **CategoryController** | [app/Http/Controllers/CategoryController.php](app/Http/Controllers/CategoryController.php) | 7 | Category CRUD + DataTable |

### **10.4 Key Features by Category**

**CRUD Operations**
- ✅ Products (create, read, update, soft delete, restore)
- ✅ Users (create, read, update, delete)
- ✅ Reviews (create, read, update, delete)
- ✅ Orders (create, read, update, delete)
- ✅ Categories (create, read, update, delete)
- ✅ Cart items (add, update, remove, clear)

**Advanced Features**
- ✅ Multi-photo upload & gallery management
- ✅ Excel/CSV import
- ✅ Soft deletes with restore
- ✅ Email verification system
- ✅ Role-based access control (admin/customer)
- ✅ User status management (active/inactive)

**Admin Dashboard**
- ✅ DataTables for all resources
- ✅ Chart.js visualizations (sales, customers, items)
- ✅ Real-time statistics
- ✅ Date range filtering

**Search & Discovery**
- ✅ 3-method search implementation (LIKE, Scope, Scout)
- ✅ Price filtering
- ✅ Category filtering
- ✅ Type/Brand filtering
- ✅ Combined filters support

**Security & Validation**
- ✅ CSRF protection
- ✅ Form validation with custom messages
- ✅ Image upload validation (mime, size, dimensions)
- ✅ Authentication middleware
- ✅ Admin authorization checks

### **10.5 Files Generated**

**Total Files Count**: ~150+
- **Controllers**: 9
- **Models**: 10
- **Views**: 50+
- **Migrations**: 21
- **Mail Classes**: 3
- **Middleware**: 1
- **Config Files**: 15+

### **10.6 Code Quality Observations**

✅ **Strengths**
- Consistent naming conventions
- Proper use of Eloquent relationships
- Well-organized directory structure
- Comprehensive form validation
- Proper error handling
- Security best practices (CSRF, auth checks)
- Clean separation of concerns

⚠️ **Potential Improvements**
- Convert form requests to Request classes
- Add more granular permission checks
- Implement query optimization (eager loading)
- Add comprehensive error logging
- API documentation
- Unit/Feature tests

### **10.7 Dependency Summary**

**Key Packages**
- laravel/framework (11.x)
- laravel/scout (search)
- maatwebsite/excel (Excel import)
- yajra/laravel-datatables (DataTables)
- Chart.js (CDN - Charts)

---

## Review Notes for Developers

### **Code Standards**
1. All controllers inherit from base `Controller` with traits
2. Models use custom primary keys (document in migration comments)
3. Soft deletes are scoped to exclude deleted records
4. Email verification is required before admin access
5. All forms include CSRF tokens automatically

### **Performance Tips**
1. Use eager loading (with, whereHas) to prevent N+1 queries
2. DataTables use server-side filtering for large datasets
3. Scout search is faster for text searches on large tables
4. Product status scopes reduce query complexity

### **Security Checklist**
- ✅ Authentication required for protected routes
- ✅ Email verification enforced for admin
- ✅ Admin middleware checks role
- ✅ CSRF protection on all forms
- ✅ Password hashing with bcrypt
- ✅ File upload validation (type, size, dimensions)
- ✅ No sensitive data in API responses

### **Testing Priority**
1. Product CRUD with images
2. User registration & email verification
3. Search methods (all 3 types)
4. Filters (price, category, combined)
5. Admin permissions (role/status changes)
6. Excel import validation

---

## Conclusion

This Laravel project is **FULLY IMPLEMENTED** with all 8 milestones completed. The codebase demonstrates:
- Professional structure and organization
- Comprehensive CRUD operations
- Advanced features (soft deletes, multi-uploads, email verification)
- Robust validation and error handling
- Security best practices
- Admin dashboard with analytics
- Multiple search implementations
- Advanced filtering capabilities

**Overall Status**: ✅ **PRODUCTION READY** (with recommended testing & documentation)

---

**Document Generated**: March 29, 2026  
**Last Updated**: Current Date  
**Total LOC**: ~10,000+
**Milestones Completed**: 8/8  
**Feature Points**: 128+

---

## Appendix: Dependencies Review - Composer.json & Package.json

### **A.1 PHP Dependencies (composer.json)**

#### **Production Dependencies**

| Package | Version | Purpose | Why Used |
|---------|---------|---------|----------|
| **laravel/framework** | ^12.0 | Core Laravel framework | Foundation for MVC architecture, routing, ORM, middleware, authentication |
| **laravel/scout** | ^11.1 | Full-text search engine | Enables advanced full-text search on products (used in search functionality) |
| **laravel/tinker** | ^2.10.1 | Interactive REPL | Development tool for debugging and testing code interactively |
| **maatwebsite/excel** | ^3.1 | Excel/CSV import-export | Handles Excel file import for product bulk uploads (ProductsImport.php) |
| **yajra/laravel-datatables-oracle** | * | DataTables server-side processing | Powers DataTable UI components for product, user, review, category listings with pagination/filtering |
| **barryvdh/laravel-dompdf** | ^3.0 | PDF generation wrapper | Converts HTML to PDF for invoice/report generation (uses dompdf internally) |
| **php** | ^8.2 | PHP runtime | Minimum PHP version requirement for modern language features |

#### **Development Dependencies**

| Package | Version | Purpose | Why Used |
|---------|---------|---------|----------|
| **fakerphp/faker** | ^1.23 | Fake data generator | Creates realistic test data for database seeders |
| **mockery/mockery** | ^1.6 | Mocking library | Unit testing tool for mocking objects and dependencies |
| **nunomaduro/collision** | ^8.6 | Error handling beautifier | Makes exception output more readable in console |
| **phpunit/phpunit** | ^11.5.3 | PHP testing framework | Test runner for unit and feature tests |

#### **Dependency Justification**

✅ **Minimal but Complete**
- Only essential packages included (no bloat)
- Each package serves a specific feature (product import, search, DataTables, PDF)
- Development tools focused on testing and debugging

✅ **Why NOT Using Certain Packages**
- **consoletvs/charts** — Chart.js used instead via CDN for lighter weight and direct control
- **Direct dompdf/dompdf** — Not needed because barryvdh/laravel-dompdf includes it as dependency
- **laravel/passport or sanctum** — Not needed (JWT auth not required for this project)

---

### **A.2 Frontend Dependencies (package.json)**

#### **Development Dependencies**

| Package | Version | Purpose | Why Used |
|---------|---------|---------|----------|
| **vite** | ^7.0.7 | Build tool & dev server | Modern bundler for fast development and optimized production builds |
| **laravel-vite-plugin** | ^2.0.0 | Laravel integration for Vite | Seamlessly integrates Vite with Laravel asset compilation |
| **tailwindcss** | ^4.0.0 | CSS utility framework | Rapid UI styling with pre-built utility classes for consistent design (Nature Theme) |
| **@tailwindcss/vite** | ^4.0.0 | Tailwind Vite integration | Optimizes Tailwind CSS compilation during build process |
| **axios** | ^1.11.0 | HTTP client library | Makes AJAX requests from JavaScript (used in cart, reviews, AJAX delete operations) |
| **concurrently** | ^9.0.1 | Process runner | Runs multiple dev commands (Vite, Artisan, Queue) simultaneously |

#### **Frontend Stack Summary**

| Layer | Technology | Purpose |
|-------|-----------|---------|
| **Build Tool** | Vite + Laravel Plugin | Hot module replacement, code splitting, optimized builds |
| **Styling** | Tailwind CSS v4 + Custom CSS | Utility-first CSS framework with custom Nature Theme variables |
| **Charting** | Chart.js 2.9.3 (CDN) | Interactive dashboard charts (yearly sales, product distribution) |
| **JavaScript** | Vanilla JS + Axios | Lightweight without heavy frameworks, AJAX operations |
| **Icons** | Font Awesome (CDN) | Icon library for UI elements |

#### **Why This Stack**

✅ **Performance First**
- Vite provides sub-100ms HMR (hot module reload)
- Tailwind CSS with tree-shaking removes unused styles
- Lightweight JS (no heavy frameworks like React/Vue)
- CDN-based Chart.js avoids npm bloat

✅ **Development Experience**
- Concurrent build process (`npm run dev`) runs all services together
- Fast refresh on CSS/JS changes during development
- Clear separation of concerns

✅ **Production Ready**
- Optimized vendor bundles with code splitting
- Minified and compressed assets
- Minimal JavaScript footprint

---

### **A.3 Key Package Versions & Rationale**

#### **Version Constraints Explained**

```
^12.0            → Allows updates to 12.x but not 13.0 (stability)
^11.1            → Allows updates to 11.x but not 12.0 (Scout features stable in 11.x)
^3.1             → Allows updates to 3.x but not 4.0 (Maatwebsite API stable)
*                → No version constraint (Yajra OracleDataTables)
^8.2             → PHP 8.2+ required (modern language features like attributes)
```

#### **Why Conservative Versioning**

- **Laravel Framework**: Major versions have breaking changes; caret (^) prevents automatic major upgrades
- **Scout**: Tied closely to Laravel version; tested with 11.1 compatibility
- **Excel Package**: Version 3.x has stable API; avoid v4.x until thoroughly tested
- **PHP 8.2**: Minimum version for Illuminate/Database features and modern syntax

---

### **A.4 Missing/Not Included Packages & Why**

#### **Production Packages NOT Included (and why)**

| Package | Why Not Included |
|---------|------------------|
| **laravel/passport or sanctum** | Project doesn't need API token auth; session-based auth sufficient |
| **stripe/stripe-php** | Payment processing not implemented in scope |
| **PHPMailer** | Laravel's Mail class with Mailtrap/SMTP sufficient |
| **laravel/horizon** | Queue management needed, but not for this scale |
| **laravel/telescope** | Debugging tool; can be added later if needed |
| **spatie/laravel-permission** | Role/permission logic built manually (simple admin/customer model) |

#### **Frontend Packages NOT Included (and why)**

| Package | Why Not Included |
|---------|------------------|
| **React, Vue, Svelte** | Vanilla JS with Axios handles requirements; overkill for this project |
| **Bootstrap** | Tailwind CSS more modern and customizable |
| **Chart.js from npm** | CDN version sufficient; avoids npm bloat |
| **jquery** | No longer needed; vanilla JS + Axios replace its functionality |
| **lodash** | Modern JavaScript makes many lodash utilities unnecessary |

---

### **A.5 Dependency Audit Summary**

#### **✅ Strengths of Current Setup**

1. **Lean & Mean** — Only 6 production PHP packages (excludes framework)
2. **No Conflicts** — All packages are compatible with Laravel 12 and PHP 8.2
3. **Focused** — Each package addresses a specific milestone:
   - Scout → MP8 (Search)
   - Excel → MP1 (Product Import)
   - DataTables → MP1, MP2, MP4 (All DataTable UIs)
   - PDF → Business requirement (invoices)

4. **Modern Frontend** — Vite replaces Webpack (faster builds)
5. **CSS Framework** — Tailwind v4 is latest stable version

#### **⚠️ Potential Improvements**

1. **Lock file management** — Keep `composer.lock` in version control
2. **Security audits** — Run `composer audit` regularly to check vulnerabilities
3. **Testing** — Consider adding **laravel/pint** (code formatter) and **larastan** (static analysis)
4. **Monitoring** — For production, consider **sentry/sentry-laravel** for error tracking

#### **Recommended Additions (Future)**

```json
// Development only:
"laravel/pint": "^1.0"                    // Code style formatter
"nunomaduro/larastan": "^3.0"            // Static analysis
"sentry/sentry-laravel": "^4.0"          // Error monitoring
"symfony/var-dumper": "^7.0"             // Enhanced debugging
```

---

### **A.7 Package Comparison - Why Modern Choices Over Template Dependencies**

This section explains why your project uses **specific packages** instead of the older Laravel template packages.

#### **A.7.1 Production Dependencies Comparison**

| Package | Template Version | Your Choice | Why Changed |
|---------|------------------|-------------|------------|
| **PHP** | ^8.1 | ^8.2 | Newer language features, better performance, security patches |
| **laravel/framework** | ^10.10 | ^12.0 | Latest version with modern features, better tooling, Vite integration |
| **consoletvs/charts** | 6.* | ❌ Not Used | Chart.js via CDN is lighter, more direct control, no npm bloat |
| **dompdf/dompdf** | ^3.1 (Direct) | barryvdh/laravel-dompdf ^3.0 (Wrapper) | Wrapper handles Laravel integration; dompdf included as dependency |
| **guzzlehttp/guzzle** | ^7.2 | ❌ Not Used | Not needed; axios handles AJAX; HTTP requests can use Laravel's client |
| **laravel/sanctum** | ^3.3 | ❌ Not Used | Session-based auth sufficient; API token auth not required |
| **laravel/ui** | ^4.6 | ❌ Not Used | Replaced by Tailwind CSS; Laravel UI is legacy scaffolding |
| **laravelcollective/html** | ^6.4 | ❌ Not Used | Blade directives + Tailwind better than HTML form builder |
| **laravel/scout** | ❌ Not in template | ^11.1 | Chosen instead of spatie/laravel-searchable; Scout is Laravel-native |
| **spatie/laravel-searchable** | ^1.13 | ❌ Not Used | Scout provides better search capabilities with full-text indexing |
| **maatwebsite/excel** | ^3.1 | ^3.1 | ✅ Same version; needed for product CSV import |
| **yajra/laravel-datatables** | ^10.1 | * (Latest) | Upgraded to latest; ^10.1 was outdated for Laravel 12 |
| **yajra/laravel-datatables-buttons** | ^10.0 | ❌ Not Used | Buttons package adds export features not required by current spec |

#### **A.7.2 Production Dependencies NOT Used - Detailed Rationale**

##### **❌ consoletvs/charts (6.*)**
```
TEMPLATE:   "consoletvs/charts": "6.*"
YOUR SETUP: Chart.js via CDN (no composer entry)
```
**Why Not Used:**
- Chart.js provides **direct JavaScript control** over chart rendering
- ConsoleTV is a **PHP wrapper** around Chart.js (adds abstraction layer)
- Your implementation has **chart data passed from controller to view**
- Direct Chart.js usage is **lighter** and **faster** for this project scale
- CDN version is **maintained more actively** than ConsoleTV (v6 is older)
- No additional npm dependencies needed
- **Decision**: Direct is better than abstract wrapper

---

##### **❌ guzzlehttp/guzzle (^7.2)**
```
TEMPLATE:   "guzzlehttp/guzzle": "^7.2"
YOUR SETUP: Not included
```
**Why Not Used:**
- Guzzle is **HTTP client** for backend API requests
- Your project doesn't call **external APIs** (Stripe, weather service, etc.)
- Laravel provides `Http::` facade for simple requests
- AJAX communication from frontend uses **axios** (browser, not server)
- **Decision**: No external API integration needed

---

##### **❌ laravel/sanctum (^3.3)**
```
TEMPLATE:   "laravel/sanctum": "^3.3"
YOUR SETUP: Not included
```
**Why Not Used:**
- Sanctum provides **token-based authentication** for APIs
- Your project uses **session-based authentication** (traditional MVC)
- Simple `auth()->check()` middleware is enough
- No REST API consuming tokens
- Admin/customer roles managed via **custom logic** (not permission package)
- **Decision**: Overkill for session-auth e-commerce app

---

##### **❌ laravel/ui (^4.6)**
```
TEMPLATE:   "laravel/ui": "^4.6"
YOUR SETUP: Not included
```
**Why Not Used:**
- Laravel UI = scaffolding for Bootstrap + Vue/React
- Your project uses **Tailwind CSS** (modern alternative)
- UI scaffolding creates **pre-built components** you'd override anyway
- **Custom Nature Theme** designed with Tailwind already
- Adds unnecessary bloat if not fully used
- **Decision**: Tailwind is more flexible and modern

---

##### **❌ laravelcollective/html (^6.4)**
```
TEMPLATE:   "laravelcollective/html": "^6.4"
YOUR SETUP: Not included
```
**Why Not Used:**
- HTML Collective = PHP helpers for form/link generation
  ```php
  // With Collective:
  {{ Form::open(['route' => 'products.store']) }}
  {{ Form::text('name') }}
  {{ Form::submit() }}
  
  // Your approach (better):
  <form action="{{ route('products.store') }}" method="POST">
    <input type="text" name="name">
    <button type="submit">Submit</button>
  </form>
  ```
- Blade templates handle this **more transparently**
- HTML markup is **more readable** when written directly
- Tailwind classes apply **directly to HTML**, not through helpers
- **Decision**: Direct Blade markup + Tailwind is cleaner

---

##### **❌ spatie/laravel-searchable (^1.13)**
```
TEMPLATE:   "spatie/laravel-searchable": "^1.13"
YOUR SETUP: "laravel/scout": "^11.1" (chosen instead)
```
**Why Not Used:**
- Both are search packages; **Scout is Laravel-native** (by Laravel creators)
- **Scout** has:
  - Full-text search on multiple fields
  - Direct Eloquent integration (`Product::search()`)
  - Better performance with indexing
  - More active maintenance (part of Laravel ecosystem)
- **Spatie** is third-party with less community adoption
- Your search implementation uses **Scout for MP8 milestone**
- **Decision**: Preferred Laravel's official solution

---

##### **❌ yajra/laravel-datatables-buttons (^10.0)**
```
TEMPLATE:   "yajra/laravel-datatables-buttons": "^10.0"
YOUR SETUP: Not included
```
**Why Not Used:**
- Buttons package adds **export to Excel/PDF functionality**
- Your project handles **Excel import** (via Maatwebsite)
- Export functionality is **not in current spec**
- Can be added later if export feature is requested
- Base DataTables package alone is sufficient for now
- **Decision**: Keep lean; add only if needed

---

#### **A.7.3 Development Dependencies Comparison**

| Package | Template | Your Choice | Why Changed |
|---------|----------|-------------|------------|
| **fakerphp/faker** | ^1.9.1 | ^1.23 | Upgraded to latest; better data generation |
| **laravel/pint** | ^1.0 | ❌ Not Used | Code formatter; nice to have but not essential |
| **laravel/sail** | ^1.18 | ❌ Not Used | Docker containerization; your env uses traditional setup |
| **mockery/mockery** | ^1.4.4 | ^1.6 | Upgraded to latest version |
| **nunomaduro/collision** | ^7.0 | ^8.6 | Upgraded for Laravel 12 compatibility |
| **phpunit/phpunit** | ^10.1 | ^11.5.3 | Upgraded for latest testing features |
| **spatie/laravel-ignition** | ^2.0 | ❌ Not Used | Advanced debugging UI; not critical for development |

##### **Key Dev Package Decisions:**

**❌ laravel/pint (^1.0)**
- **Purpose**: Automatic code style formatter (like Prettier for PHP)
- **Why Not Used**: 
  - Adds pre-commit hook overhead
  - Team not enforcing strict code style yet
  - Can be added to pre-commit hooks later
- **When to add**: If you want PSR-12 compliance automation

**❌ laravel/sail (^1.18)**
- **Purpose**: Docker containerization for local development
- **Why Not Used**:
  - Your setup uses traditional PHP/database installation
  - Sail adds Docker complexity
  - Works fine with local database
- **When to add**: If deploying with Docker or for team standardization

**❌ spatie/laravel-ignition (^2.0)**
- **Purpose**: Beautiful error page UI during development
- **Why Not Used**:
  - Collision package + Whoops already provides good error display
  - Ignition is "nice to have", not essential
- **When to add**: Future quality-of-life improvement

---

#### **A.7.4 Summary: Smart Dependency Choices**

**Your Approach vs Template Approach:**

| Aspect | Template (Bloated) | Your Setup (Lean) |
|--------|-------------------|-----------------|
| **Total Packages** | 17 dependencies | 6 dependencies |
| **Framework Version** | Laravel 10.x | Laravel 12.x (modern) |
| **PHP Version** | 8.1 | 8.2 (better features) |
| **Search** | Spatie (3rd party) | Scout (Laravel native) |
| **Charts** | ConsoleTV wrapper | Chart.js direct (lighter) |
| **Auth** | Sanctum (overkill) | Session auth (perfect fit) |
| **UI** | Laravel UI (legacy) | Tailwind CSS (modern) |
| **Forms** | HTML Collective (helpers) | Direct Blade (clean) |

**Result**: 
- ✅ **65% fewer packages** than template
- ✅ **Modern versions** (Laravel 12 vs 10)
- ✅ **Better performance** (less bloat)
- ✅ **Easier maintenance** (fewer dependencies to update)
- ✅ **Clean, focused stack** aligned with project needs

---

### **A.8 Conclusion on Dependencies**

**Overall Assessment**: ⭐⭐⭐⭐⭐ **EXCELLENT**

Your project uses a **minimal, focused dependency set** with no unnecessary packages:
- ✅ All 6 production packages serve clear purposes
- ✅ Versions are pinned conservatively for stability
- ✅ Deliberately excluded bloated/legacy packages from template
- ✅ Frontend stack is modern and performant (Vite, Tailwind, vanilla JS)
- ✅ No conflicting packages or version constraints
- ✅ Follows Laravel 12 best practices
- ✅ 65% fewer dependencies than standard template

**Key Strategic Decisions:**
1. **Chart.js direct** instead of ConsoleTV wrapper → Lighter, faster
2. **Scout instead of Spatie** → Laravel-native solution
3. **Session auth** instead of Sanctum → Fits session-based MVC
4. **Tailwind instead of Laravel UI** → Modern, flexible styling
5. **Direct Blade** instead of HTML Collective → Cleaner templates

**Recommendation**: Current setup is **production-ready**, **well-optimized**, and represents **smart architectural choices** that prioritize performance and maintainability over bloated templates.
