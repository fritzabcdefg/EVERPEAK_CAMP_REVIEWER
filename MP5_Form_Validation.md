# MP5: Form Validation

## Overview
**Total Points: 15pts**
- Form validation for add/edit product/service (8pts)
- User registration validation (7pts)

---

## Validation Concepts

### Validation Types in Laravel

1. **Request validation** (most common) - Validate incoming HTTP requests
2. **Model validation** - Validate data before saving to database  
3. **Form attributes** - HTML5 validation attributes (client-side)
4. **Custom validation rules** - Complex business logic

### Syntax

```php
// Basic validation in controller
$validated = $request->validate([
    'field_name' => 'rule1|rule2|rule3',
], [
    'field_name.rule1' => 'Custom error message',
]);
```

---

## Product Validation

### ProductController - store() Validation

```php
public function store(Request $request)
{
    $validated = $request->validate([
        // Product name: 3-255 chars, alphanumeric + special chars
        'name' => 'required|string|min:3|max:255|regex:/^[a-zA-Z0-9\s\-&\/.,()]+$/',
        
        // Description: 10-5000 chars
        'description' => 'required|string|min:10|max:5000',
        
        // Cost price: numeric, non-negative, up to 999999.99
        'cost_price' => 'required|numeric|min:0|max:999999.99',
        
        // Sell price: numeric, >= cost_price (higher than cost)
        'sell_price' => 'required|numeric|min:0|max:999999.99|gte:cost_price',
        
        // Category: must exist in categories table
        'category_id' => 'nullable|exists:categories,category_id',
        
        // Stock quantity: optional integer
        'stocks' => 'nullable|integer|min:0',
        
        // Main image: optional, must be image file
        'img_path' => 'nullable|image|mimes:jpeg,png,jpg,gif|max:2048|dimensions:min_width=100,min_height=100',
        
        // Gallery images: array of images
        'images.*' => 'nullable|image|mimes:jpeg,png,jpg,gif|max:2048|dimensions:min_width=100,min_height=100',
    ], [
        // Custom error messages for better UX
        
        'name.required' => 'Product name is required.',
        'name.min' => 'Product name must be at least 3 characters.',
        'name.max' => 'Product name cannot exceed 255 characters.',
        'name.regex' => 'Product name contains invalid characters. Only letters, numbers, hyphens, ampersands, slashes, periods, commas, and parentheses are allowed.',
        
        'description.required' => 'Description is required.',
        'description.min' => 'Description must be at least 10 characters.',
        'description.max' => 'Description cannot exceed 5000 characters.',
        
        'cost_price.required' => 'Cost price is required.',
        'cost_price.numeric' => 'Cost price must be a valid number.',
        'cost_price.min' => 'Cost price cannot be negative.',
        'cost_price.max' => 'Cost price is too high.',
        
        'sell_price.required' => 'Selling price is required.',
        'sell_price.numeric' => 'Selling price must be a valid number.',
        'sell_price.min' => 'Selling price cannot be negative.',
        'sell_price.max' => 'Selling price is too high.',
        'sell_price.gte' => 'Selling price must be greater than or equal to cost price.',
        
        'category_id.exists' => 'The selected category does not exist.',
        
        'img_path.image' => 'Main image must be a valid image file.',
        'img_path.mimes' => 'Main image must be JPG, PNG, or GIF format.',
        'img_path.max' => 'Main image cannot exceed 2MB.',
        'img_path.dimensions' => 'Main image must be at least 100x100 pixels.',
        
        'images.*.image' => 'Gallery images must be valid image files.',
        'images.*.mimes' => 'Gallery images must be JPG, PNG, or GIF format.',
        'images.*.max' => 'Gallery images cannot exceed 2MB.',
        'images.*.dimensions' => 'Gallery images must be at least 100x100 pixels.',
    ]);

    // After validation passes, $validated array contains only validated data
    // Use $validated for creating model
    $product = Product::create($validated);
    
    // ... handle file uploads
}
```

### ProductController - update() Validation

```php
public function update(Request $request, Product $product)
{
    // Similar to store() but:
    // - Some fields are optional for partial updates
    // - Email fields might need unique exception
    
    $validated = $request->validate([
        'name' => 'sometimes|string|min:3|max:255|regex:/^[a-zA-Z0-9\s\-&\/.,()]+$/',
        'description' => 'sometimes|string|min:10|max:5000',
        'cost_price' => 'sometimes|numeric|min:0|max:999999.99',
        'sell_price' => 'sometimes|numeric|min:0|max:999999.99|gte:cost_price',
        'category_id' => 'nullable|exists:categories,category_id',
        'img_path' => 'sometimes|image|mimes:jpeg,png,jpg,gif|max:2048|dimensions:min_width=100,min_height=100',
        'images.*' => 'sometimes|image|mimes:jpeg,png,jpg,gif|max:2048|dimensions:min_width=100,min_height=100',
    ], [
        'sell_price.gte' => 'Selling price must be >= cost price.',
    ]);

    $product->update($validated);
}
```

---

## User Registration Validation

### UserController - storeRegister() Validation

```php
public function storeRegister(Request $request)
{
    $validated = $request->validate([
        // First name: 2+ chars, letters/spaces/hyphens/apostrophes only
        'first_name' => 'required|string|min:2|max:255|regex:/^[a-zA-Z\s\-\']+$/',
        
        // Last name: 2+ chars, letters/spaces/hyphens/apostrophes only  
        'last_name' => 'required|string|min:2|max:255|regex:/^[a-zA-Z\s\-\']+$/',
        
        // Email: valid format, unique in users table, verify DNS
        'email' => 'required|email:rfc,dns|unique:users|max:255',
        
        // Password: 8+ chars with uppercase, lowercase, number, special char
        // (?=.*[a-z]) - lookahead for lowercase
        // (?=.*[A-Z]) - lookahead for uppercase
        // (?=.*\d) - lookahead for digit
        // (?=.*[@$!%*?&]) - lookahead for special character
        'password' => 'required|string|min:8|max:255|regex:/^(?=.*[a-z])(?=.*[A-Z])(?=.*\d)(?=.*[@$!%*?&])[A-Za-z\d@$!%*?&]+$/|confirmed',
        
        // Confirmation: must match password field
        'password_confirmation' => 'required|string|same:password',
        
        // Profile photo: optional
        'photo' => 'nullable|image|mimes:jpeg,png,jpg,gif|max:2048|dimensions:min_width=100,min_height=100',
    ], [
        // FIRST NAME ERRORS
        'first_name.required' => 'First name is required.',
        'first_name.min' => 'First name must be at least 2 characters long.',
        'first_name.max' => 'First name cannot exceed 255 characters.',
        'first_name.regex' => 'First name can only contain letters, spaces, hyphens, and apostrophes.',
        
        // LAST NAME ERRORS
        'last_name.required' => 'Last name is required.',
        'last_name.min' => 'Last name must be at least 2 characters long.',
        'last_name.max' => 'Last name cannot exceed 255 characters.',
        'last_name.regex' => 'Last name can only contain letters, spaces, hyphens, and apostrophes.',
        
        // EMAIL ERRORS
        'email.required' => 'Email address is required.',
        'email.email' => 'Please enter a valid email address.',
        'email.unique' => 'This email address is already registered.',
        'email.max' => 'Email address cannot exceed 255 characters.',
        
        // PASSWORD ERRORS
        'password.required' => 'Password is required.',
        'password.min' => 'Password must be at least 8 characters long.',
        'password.max' => 'Password cannot exceed 255 characters.',
        'password.regex' => 'Password must contain at least one uppercase letter, one lowercase letter, one number, and one special character (@$!%*?&).',
        'password.confirmed' => 'Passwords do not match.',
        
        // PASSWORD CONFIRMATION ERRORS
        'password_confirmation.required' => 'Password confirmation is required.',
        'password_confirmation.same' => 'Passwords do not match.',
        
        // PHOTO ERRORS
        'photo.image' => 'Photo must be a valid image file.',
        'photo.mimes' => 'Photo must be JPG, PNG or GIF format.',
        'photo.max' => 'Photo cannot exceed 2MB.',
        'photo.dimensions' => 'Photo must be at least 100x100 pixels.',
    ]);

    // Store user with validated data
    User::create($validated);
}
```

---

## Validation Rules Reference

### Rules Used in This Project

| Rule | Meaning | Example |
|------|---------|---------|
| `required` | Field must have a value | `'name' => 'required'` |
| `string` | Value must be text | `'name' => 'string'` |
| `numeric` | Value must be a number | `'price' => 'numeric'` |
| `integer` | Value must be whole number | `'quantity' => 'integer'` |
| `min:3` | String min length OR number minimum | `'name' => 'min:3'` |
| `max:255` | String max length OR number maximum | `'name' => 'max:255'` |
| `email` | Must be valid email format | `'email' => 'email'` |
| `email:rfc,dns` | Email + DNS verification | `'email' => 'email:rfc,dns'` |
| `unique:table` | Value must be unique in table | `'email' => 'unique:users'` |
| `exists:table,column` | Value must exist in table | `'category_id' => 'exists:categories,category_id'` |
| `image` | File must be image MIME type | `'photo' => 'image'` |
| `mimes:jpeg,png` | File must be one of types | `'photo' => 'mimes:jpeg,png'` |
| `max:2048` | File max size in KB | `'photo' => 'max:2048'` |
| `dimensions:min_width=100` | Image must be min dimensions | `'photo' => 'dimensions:min_width=100'` |
| `confirmed` | Value must match {field}_confirmation | `'password' => 'confirmed'` |
| `same:field` | Value must equal another field | `'confirm' => 'same:password'` |
| `gte:field` | Value must be >= another field | `'sell' => 'gte:cost'` |
| `regex:/pattern/` | Regular expression pattern | `'name' => 'regex:/^[a-z]+$/'` |
| `nullable` | Field optional (can be null) | `'photo' => 'nullable'` |
| `sometimes` | Only validate if field present | `'name' => 'sometimes'` |

---

## Blade Template - Displaying Errors

### Flash Error Messages

```blade
<!-- resources/views/layouts/flash-messages.blade.php -->

@if ($errors->any())
    <div class="alert alert-danger alert-dismissible fade show" role="alert">
        <strong>Validation Error!</strong>
        @foreach ($errors->all() as $message)
            <div>{{ $message }}</div>
        @endforeach
        <button type="button" class="btn-close" data-bs-dismiss="alert"></button>
    </div>
@endif

@if (session('success'))
    <div class="alert alert-success alert-dismissible fade show" role="alert">
        {{ session('success') }}
        <button type="button" class="btn-close" data-bs-dismiss="alert"></button>
    </div>
@endif
```

### Individual Field Errors

```blade
<!-- In forms -->
<div class="mb-3">
    <label for="email" class="form-label">Email</label>
    <input type="email" 
           class="form-control @error('email') is-invalid @enderror" 
           id="email" name="email"
           value="{{ old('email') }}">  {{-- Repopulate form on error --}}
    
    <!-- Show error if exists -->
    @error('email')
        <div class="invalid-feedback d-block">
            {{ $message }}  {{-- Display the error message --}}
        </div>
    @enderror
</div>
```

### HTML5 Validation Attributes

```blade
<!-- Client-side validation (browser-level) -->
<form method="POST">
    @csrf
    
    <input type="text" name="name" required minlength="3" maxlength="255" pattern="^[a-zA-Z0-9\s\-&\/.,()]+$">
    
    <input type="number" name="price" required min="0" max="999999.99" step="0.01">
    
    <input type="email" name="email" required>
    
    <input type="password" name="password" required minlength="8">
    
    <input type="file" name="photo" accept="image/*">
    
    <button type="submit">Submit</button>
</form>
```

---

## Summary

**MP5 implements comprehensive validation with:**
- ✅ Product field validation (name, description, prices, images)
- ✅ User registration validation (names, email, strong password)
- ✅ Custom error messages for better UX
- ✅ HTML5 client-side validation
- ✅ Server-side validation with Laravel
- ✅ Image dimension/type/size checks
- ✅ Email DNS verification
- ✅ Strong password requirements
- ✅ Form repopulation on validation errors

**Total Implementation: 15 Points**
