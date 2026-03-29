# MP2: User Management CRUD

## Overview
**Total Points: 20pts**
- Stage 1: User Registration with Photo Upload (5pts) ✅
- Stage 2: User Profile Update with Photo (5pts) ✅
- Stage 3: DataTable User Listing (5pts) ✅
- Stage 4: Admin User Management (5pts) ✅

---

## Stage 1: User Registration with Photo Upload (5pts)

### User Model

```php
<?php

namespace App\Models;

use Illuminate\Foundation\Auth\User as Authenticatable;
use Illuminate\Database\Eloquent\Factories\HasFactory;

class User extends Authenticatable
{
    use HasFactory;
    
    /**
     * Attributes that can be mass-assigned
     * These are the only attributes that can be set via User::create() or $user->update()
     * This is a security feature to prevent "attribute injection"
     */
    protected $fillable = [
        'first_name',          // User's first name
        'last_name',           // User's last name
        'email',               // Email address (unique)
        'password',            // Hashed password
        'role',                // 'customer' or 'admin'
        'phone',               // Contact phone number
        'address',             // Delivery address
        'photo',               // Profile photo path
        'status',              // 'active' or 'inactive'
        'email_verified_at',   // Timestamp when email was verified
    ];
    
    /**
     * Attributes that should be hidden when serializing (e.g., API responses)
     * Password and remember token are sensitive and should not be exposed
     */
    protected $hidden = [
        'password',
        'remember_token',
    ];
    
    /**
     * Attribute casting
     * Automatically converts attributes to specified types
     */
    protected function casts(): array
    {
        return [
            'email_verified_at' => 'datetime',  // Convert to Carbon instance
            'password' => 'hashed',             // Automatically hash on assignment
        ];
    }

    // Relationships
    public function cartItems()
    {
        return $this->hasMany(CartItem::class, 'user_id');
    }

    public function orders()
    {
        return $this->hasMany(Order::class, 'user_id');
    }

    public function reviews()
    {
        return $this->hasMany(Review::class, 'user_id');
    }
}
```

### UserController - createRegister() Method (Show Form)

```php
/**
 * Show the registration form page
 * 
 * @return \Illuminate\View\View
 */
public function createRegister()
{
    return view('auth.register');
}
```

### UserController - storeRegister() Method (Process Registration)

```php
/**
 * Handle user registration with optional photo upload
 * 
 * 1. Validate all inputs including strong password requirements
 * 2. Hash password using bcrypt
 * 3. Handle photo upload to storage
 * 4. Create user record in database
 * 5. Generate email verification token
 * 6. Send verification email
 * 
 * @param Request $request
 * @return \Illuminate\Http\RedirectResponse
 */
public function storeRegister(Request $request)
{
    // Validate all registration fields
    $validated = $request->validate([
        // First name: 2+ chars, letters/spaces/hyphens only
        'first_name' => 'required|string|min:2|max:255|regex:/^[a-zA-Z\s\-\']+$/',
        
        // Last name: 2+ chars, letters/spaces/hyphens only
        'last_name' => 'required|string|min:2|max:255|regex:/^[a-zA-Z\s\-\']+$/',
        
        // Email: must be valid, unique, and verify with DNS
        'email' => 'required|email:rfc,dns|unique:users|max:255',
        
        // Password: 8+ chars, must include:
        // - At least one lowercase letter (?=.*[a-z])
        // - At least one uppercase letter (?=.*[A-Z])
        // - At least one digit (?=.*\d)
        // - At least one special character (?=.*[@$!%*?&])
        'password' => 'required|string|min:8|max:255|regex:/^(?=.*[a-z])(?=.*[A-Z])(?=.*\d)(?=.*[@$!%*?&])[A-Za-z\d@$!%*?&]+$/|confirmed',
        
        // Password confirmation: must match password field exactly
        'password_confirmation' => 'required|string|same:password',
        
        // Optional profile photo: max 2MB, at least 100x100px
        'photo' => 'nullable|image|mimes:jpeg,png,jpg,gif|max:2048|dimensions:min_width=100,min_height=100',
    ], [
        // Custom error messages
        'first_name.regex' => 'First name can only contain letters, spaces, hyphens, and apostrophes.',
        'email.unique' => 'This email address is already registered.',
        'password.regex' => 'Password must contain at least one uppercase letter, one lowercase letter, one number, and one special character.',
        'password.confirmed' => 'Passwords do not match.',
        'photo.dimensions' => 'Photo must be at least 100x100 pixels.',
    ]);

    // Prepare user data
    $userData = [
        'first_name' => $validated['first_name'],
        'last_name' => $validated['last_name'],
        'email' => $validated['email'],
        'password' => bcrypt($validated['password']),  // Hash password with bcrypt
        'role' => 'customer',                           // Default role
        'status' => 'active',                           // Default status
        'email_verified_at' => null,                    // Not verified until email click
    ];

    // Handle optional photo upload
    if ($request->hasFile('photo')) {
        // Store in storage/app/public/users/profiles/
        // Returns path: users/profiles/uuid-filename.jpg
        $userData['photo'] = $request->file('photo')->store('users/profiles', 'public');
    }

    // Create user record
    $user = User::create($userData);

    // Generate email verification token
    $token = Str::random(64);  // 64-character random string
    EmailVerificationToken::create([
        'user_id' => $user->id,
        'token' => $token,
        'created_at' => now(),
    ]);

    // Send verification email
    Mail::send(new VerifyEmailMail($user, $token));

    // Redirect to login with success message
    return redirect()->route('login')
                    ->with('success', 'Registration successful! A verification link has been sent to your email. Please check your inbox.');
}
```

**Key Validation Rules:**

| Rule | Regex | Meaning |
|------|-------|---------|
| `email:rfc,dns` | - | Validates against RFC 5321 + checks DNS records |
| `regex:/^[a-zA-Z\s\-\']+$/` | Letter/space/hyphen/apostrophe | Allows John-Paul O'Brien |
| `regex:/^(?=.*[a-z])...` | Lookahead for lowercase | Must contain at least one a-z |
| `(?=.*[A-Z])` | Lookahead for uppercase | Must contain at least one A-Z |
| `(?=.*\d)` | Lookahead for digit | Must contain at least one 0-9 |
| `(?=.*[@$!%*?&])` | Lookahead for special char | Must contain one of @$!%*?& |

### Blade View - Registration Form

```blade
<form method="POST" action="{{ route('register') }}" enctype="multipart/form-data">
    @csrf
    
    <div class="row">
        <div class="col-md-6 mb-3">
            <label for="first_name" class="form-label">First Name</label>
            <input type="text" class="form-control @error('first_name') is-invalid @enderror" 
                   id="first_name" name="first_name" value="{{ old('first_name') }}" required>
            @error('first_name')
                <div class="invalid-feedback d-block">{{ $message }}</div>
            @enderror
        </div>
        
        <div class="col-md-6 mb-3">
            <label for="last_name" class="form-label">Last Name</label>
            <input type="text" class="form-control @error('last_name') is-invalid @enderror" 
                   id="last_name" name="last_name" value="{{ old('last_name') }}" required>
            @error('last_name')
                <div class="invalid-feedback d-block">{{ $message }}</div>
            @enderror
        </div>
    </div>

    <div class="mb-3">
        <label for="email" class="form-label">Email Address</label>
        <input type="email" class="form-control @error('email') is-invalid @enderror" 
               id="email" name="email" value="{{ old('email') }}" required>
        @error('email')
            <div class="invalid-feedback d-block">{{ $message }}</div>
        @enderror
    </div>

    <div class="mb-3">
        <label for="password" class="form-label">Password</label>
        <input type="password" class="form-control @error('password') is-invalid @enderror" 
               id="password" name="password" required>
        <small class="form-text text-muted">
            Must be 8+ characters with uppercase, lowercase, number, and special character
        </small>
        @error('password')
            <div class="invalid-feedback d-block">{{ $message }}</div>
        @enderror
    </div>

    <div class="mb-3">
        <label for="password_confirmation" class="form-label">Confirm Password</label>
        <input type="password" class="form-control @error('password_confirmation') is-invalid @enderror" 
               id="password_confirmation" name="password_confirmation" required>
        @error('password_confirmation')
            <div class="invalid-feedback d-block">{{ $message }}</div>
        @enderror
    </div>

    <div class="mb-3">
        <label for="photo" class="form-label">Profile Photo (Optional)</label>
        <input type="file" class="form-control @error('photo') is-invalid @enderror" 
               id="photo" name="photo" accept="image/*" onchange="previewPhoto(this)">
        <img id="photoPreview" src="" alt="Preview" style="max-width: 150px; margin-top: 10px; display: none;">
        @error('photo')
            <div class="invalid-feedback d-block">{{ $message }}</div>
        @enderror
    </div>

    <button type="submit" class="btn btn-primary">Register</button>
</form>

<script>
function previewPhoto(input) {
    const preview = document.getElementById('photoPreview');
    if (input.files && input.files[0]) {
        const reader = new FileReader();
        reader.onload = function(e) {
            preview.src = e.target.result;
            preview.style.display = 'block';
        }
        reader.readAsDataURL(input.files[0]);
    }
}
</script>
```

---

## Stage 2: User Profile Update with Photo (5pts)

### UserController - createProfile() (Show Form)

```php
/**
 * Show form for user to create/complete their profile
 * Called after registration or on first login
 */
public function createProfile()
{
    return view('profile.create');
}
```

### UserController - storeProfile() (Save Profile)

```php
/**
 * Store profile data for currently authenticated user
 * Handles phone, address, and optional photo upload
 */
public function storeProfile(Request $request)
{
    // Get current authenticated user
    $user = Auth::user();
    
    // Validate profile fields
    $validated = $request->validate([
        'phone' => 'required|string|max:50',
        'address' => 'required|string|max:1000',
        'photo' => 'nullable|image|mimes:jpeg,png,jpg,gif|max:2048',
    ]);

    // Handle photo upload
    if ($request->hasFile('photo')) {
        // Delete old photo if exists
        if ($user->photo) {
            Storage::disk('public')->delete($user->photo);
        }
        
        // Upload new photo
        $validated['photo'] = $request->file('photo')->store('users/profiles', 'public');
    }

    // Update user with validated data
    $user->update($validated);

    return redirect()->route('home')
                    ->with('success', 'Profile saved successfully!');
}
```

### UserController - editProfile() (Show Edit Form)

```php
/**
 * Show form to edit current user's profile
 */
public function editProfile()
{
    return view('profile.edit');
}
```

### UserController - updateProfile() (Update Profile)

```php
/**
 * Update profile for currently authenticated user
 * Similar to storeProfile but called for existing profile updates
 */
public function updateProfile(Request $request)
{
    $user = Auth::user();
    
    $validated = $request->validate([
        'phone' => 'required|string|max:50',
        'address' => 'required|string|max:1000',
        'photo' => 'nullable|image|mimes:jpeg,png,jpg,gif|max:2048',
    ]);

    // Handle photo update
    if ($request->hasFile('photo')) {
        if ($user->photo) {
            Storage::disk('public')->delete($user->photo);  // Delete old
        }
        $validated['photo'] = $request->file('photo')->store('users/profiles', 'public');
    }

    $user->update($validated);
    
    return redirect()->route('profile.index')
                    ->with('success', 'Profile updated successfully');
}
```

### Blade View - Edit Profile Form

```blade
<form method="POST" action="{{ route('profile.update') }}" enctype="multipart/form-data">
    @csrf
    @method('PUT')
    
    <div class="row">
        <div class="col-md-8">
            <div class="form-group mb-3">
                <label for="first_name" class="form-label">First Name</label>
                <input type="text" class="form-control" id="first_name" 
                       value="{{ Auth::user()->first_name }}" disabled>
            </div>
            
            <div class="form-group mb-3">
                <label for="email" class="form-label">Email</label>
                <input type="email" class="form-control" id="email" 
                       value="{{ Auth::user()->email }}" disabled>
            </div>
            
            <div class="form-group mb-3">
                <label for="phone" class="form-label">Phone</label>
                <input type="tel" class="form-control @error('phone') is-invalid @enderror" 
                       id="phone" name="phone" value="{{ old('phone', Auth::user()->phone) }}" required>
                @error('phone')
                    <div class="invalid-feedback d-block">{{ $message }}</div>
                @enderror
            </div>
            
            <div class="form-group mb-3">
                <label for="address" class="form-label">Address</label>
                <textarea class="form-control @error('address') is-invalid @enderror" 
                          id="address" name="address" rows="3" required>{{ old('address', Auth::user()->address) }}</textarea>
                @error('address')
                    <div class="invalid-feedback d-block">{{ $message }}</div>
                @enderror
            </div>
        </div>
        
        <div class="col-md-4">
            <div class="form-group mb-3">
                <label for="photo" class="form-label">Profile Photo</label>
                
                <!-- Current photo display -->
                @if(Auth::user()->photo)
                    <div class="mb-2">
                        <img src="{{ Storage::url(Auth::user()->photo) }}" 
                             alt="Current photo" class="img-thumbnail" style="max-width: 200px;">
                    </div>
                @endif
                
                <!-- Photo upload input -->
                <input type="file" class="form-control @error('photo') is-invalid @enderror" 
                       id="photo" name="photo" accept="image/*" onchange="previewPhoto(this)">
                
                <!-- Preview for new photo -->
                <img id="photoPreview" src="" alt="Preview" class="img-thumbnail mt-2" 
                     style="max-width: 200px; display: none;">
                
                @error('photo')
                    <div class="invalid-feedback d-block">{{ $message }}</div>
                @enderror
            </div>
        </div>
    </div>
    
    <button type="submit" class="btn btn-primary">Update Profile</button>
</form>

<script>
function previewPhoto(input) {
    const preview = document.getElementById('photoPreview');
    if (input.files && input.files[0]) {
        const reader = new FileReader();
        reader.onload = function(e) {
            preview.src = e.target.result;
            preview.style.display = 'block';
        }
        reader.readAsDataURL(input.files[0]);
    }
}
</script>
```

---

## Stage 3: DataTable User Listing (5pts)

### UserController - index() (Show DataTable View)

```php
/**
 * Display the user management page with DataTable
 */
public function index()
{
    return view('users.index');
}
```

### UserController - datatable() (API Endpoint)

```php
/**
 * Get users data for DataTables server-side processing
 * Called by AJAX from the DataTable on the index view
 * 
 * Security checks:
 * 1. User must be authenticated
 * 2. User must be admin
 */
public function datatable(Request $request)
{
    try {
        // Log access for audit trail
        \Log::info('UserController::datatable called - User: ' . (auth()->check() ? auth()->user()->id : 'NOT_AUTHENTICATED'));
        
        // Check authentication
        if (!auth()->check()) {
            return response()->json(['error' => 'Unauthorized'], 401);
        }
        
        // Check admin role
        if (auth()->user()->role !== 'admin') {
            return response()->json(['error' => 'Forbidden'], 403);
        }
        
        // Build query - order by newest first
        $query = User::query()->orderBy('created_at', 'desc');

        // Process with Yajra DataTables
        return DataTables::of($query)
            // Combine first_name and last_name into single "name" column
            ->addColumn('name', function ($user) {
                return $user->first_name . ' ' . $user->last_name;
            })
            
            // Display user photo with fallback
            ->addColumn('photo', function ($user) {
                return $user->photo 
                    ? '<img src="' . Storage::url($user->photo) . '" alt="' . $user->first_name . '" 
                             width="40" height="40" class="img-thumbnail rounded-circle">'
                    : '<span class="badge bg-secondary">No Photo</span>';
            })
            
            // Role column with dropdown (for admin to change role)
            ->addColumn('role', function ($user) {
                // Admin can change role of OTHER users (not themselves)
                if (auth()->check() && auth()->user()->role === 'admin' && auth()->user()->id !== $user->id) {
                    return '<form action="' . route('users.updateRole', $user) . '" method="POST" style="display:inline;">
                        ' . csrf_field() . method_field('PUT') . '
                        <select name="role" class="form-select form-select-sm" onchange="this.form.submit()" style="width: 120px;">
                            <option value="customer" ' . ($user->role === 'customer' ? 'selected' : '') . '>Customer</option>
                            <option value="admin" ' . ($user->role === 'admin' ? 'selected' : '') . '>Admin</option>
                        </select>
                    </form>';
                } else {
                    // Non-admin users see read-only badge
                    return '<span class="badge bg-' . ($user->role === 'admin' ? 'danger' : 'primary') . '">' 
                        . ucfirst($user->role) . '</span>';
                }
            })
            
            // Status column with dropdown (for admin to change status)
            ->addColumn('status', function ($user) {
                if (auth()->check() && auth()->user()->role === 'admin' && auth()->user()->id !== $user->id) {
                    return '<form action="' . route('users.updateStatus', $user) . '" method="POST" style="display:inline;">
                        ' . csrf_field() . method_field('PUT') . '
                        <select name="status" class="form-select form-select-sm" onchange="this.form.submit()" style="width: 120px;">
                            <option value="active" ' . ($user->status === 'active' ? 'selected' : '') . '>Active</option>
                            <option value="inactive" ' . ($user->status === 'inactive' ? 'selected' : '') . '>Inactive</option>
                        </select>
                    </form>';
                } else {
                    return '<span class="badge bg-' . ($user->status === 'active' ? 'success' : 'warning') . '">' 
                        . ucfirst($user->status) . '</span>';
                }
            })
            
            // Format created date
            ->addColumn('created', function ($user) {
                return $user->created_at->format('M d, Y');
            })
            
            // Action buttons
            ->addColumn('actions', function ($user) {
                return $this->renderUserActions($user);
            })
            
            // Custom search on name fields
            ->filterColumn('name', function ($query, $keyword) {
                $query->where('first_name', 'like', "%{$keyword}%")
                      ->orWhere('last_name', 'like', "%{$keyword}%")
                      ->orWhere('email', 'like', "%{$keyword}%")
                      ->orWhere('phone', 'like', "%{$keyword}%");
            })
            
            // Raw HTML columns (not escaped)
            ->rawColumns(['photo', 'role', 'status', 'actions'])
            ->make(true);
            
    } catch (\Exception $e) {
        \Log::error('DataTable error: ' . $e->getMessage());
        return response()->json(['error' => 'Server error: ' . $e->getMessage()], 500);
    }
}
```

### renderUserActions() Helper

```php
/**
 * Generate action buttons for each user row
 */
private function renderUserActions($user)
{
    $actions = '<div class="btn-group btn-group-sm">';
    
    // View profile button
    $actions .= '<a href="' . route('users.show', $user) . '" class="btn btn-info" title="View">
                    <i class="fas fa-eye"></i>
                </a>';
    
    // Admin-only actions
    if (auth()->check() && auth()->user()->role === 'admin') {
        // Edit button
        $actions .= '<a href="' . route('users.edit', $user) . '" class="btn btn-warning" title="Edit">
                        <i class="fas fa-edit"></i>
                    </a>';
        
        // Delete button (prevent self-delete)
        if (auth()->user()->id !== $user->id) {
            $actions .= '<button type="button" class="btn btn-danger" title="Delete" 
                            onclick="if(confirm(\'Are you sure?\')) { 
                                document.getElementById(\'delete-user-' . $user->id . '\').submit(); 
                            }">
                            <i class="fas fa-trash"></i>
                        </button>';
            
            // Hidden delete form
            $actions .= '<form id="delete-user-' . $user->id . '" 
                            action="' . route('users.destroy', $user) . '" method="POST" style="display:none;">
                            <input type="hidden" name="_method" value="DELETE">
                            <input type="hidden" name="_token" value="' . csrf_token() . '">
                        </form>';
        }
    }
    
    $actions .= '</div>';
    return $actions;
}
```

### Blade View - Users DataTable

```blade
<div class="table-responsive">
    <table id="usersTable" class="table table-striped table-hover">
        <thead class="table-dark">
            <tr>
                <th>Name</th>
                <th>Email</th>
                <th>Phone</th>
                <th>Photo</th>
                <th>Role</th>
                <th>Status</th>
                <th>Created</th>
                <th>Actions</th>
            </tr>
        </thead>
        <tbody></tbody>
    </table>
</div>

<script>
$(document).ready(function() {
    $('#usersTable').DataTable({
        serverSide: true,
        ajax: '{{ route("users.datatable") }}',
        columns: [
            { data: 'name' },
            { data: 'email' },
            { data: 'phone' },
            { data: 'photo' },
            { data: 'role' },
            { data: 'status' },
            { data: 'created' },
            { data: 'actions', orderable: false, searchable: false }
        ],
        pageLength: 10,
        lengthMenu: [5, 10, 25, 50, 100],
        order: [[6, 'desc']],  // Order by created date descending
        language: {
            search: "Search users (name, email, phone):",
        }
    });
});
</script>
```

---

## Stage 4: Admin User Management (5pts)

### UserController - updateRole() (Admin Change Role)

```php
/**
 * Update user role via AJAX form submission
 * Admin can change other users' roles but not their own
 * 
 * @param Request $request
 * @param User $user
 * @return \Illuminate\Http\RedirectResponse
 */
public function updateRole(Request $request, User $user)
{
    // Security: Check authentication, admin role, and prevent self-update
    if (!auth()->check() || auth()->user()->role !== 'admin' || auth()->user()->id === $user->id) {
        return redirect()->route('users.index')
                        ->with('error', 'Unauthorized action');
    }
    
    // Validate new role
    $validated = $request->validate([
        'role' => 'required|in:customer,admin',
    ]);
    
    // Update user role
    $user->update(['role' => $validated['role']]);
    
    return redirect()->route('users.index')
                    ->with('success', $user->first_name . '\'s role has been updated to ' . ucfirst($validated['role']));
}
```

### UserController - updateStatus() (Admin Change Status)

```php
/**
 * Update user status (active/inactive) via AJAX form
 * Admin can change other users' status but not their own
 * Inactive users cannot log in
 * 
 * @param Request $request
 * @param User $user
 * @return \Illuminate\Http\RedirectResponse
 */
public function updateStatus(Request $request, User $user)
{
    // Security checks
    if (!auth()->check() || auth()->user()->role !== 'admin' || auth()->user()->id === $user->id) {
        return redirect()->route('users.index')
                        ->with('error', 'Unauthorized action');
    }
    
    // Validate new status
    $validated = $request->validate([
        'status' => 'required|in:active,inactive',
    ]);
    
    // Update status
    $user->update(['status' => $validated['status']]);
    
    return redirect()->route('users.index')
                    ->with('success', $user->first_name . '\'s status has been changed to ' . ucfirst($validated['status']));
}
```

### Modification to Login - Prevent Inactive User Login

```php
/**
 * In UserController::storeLogin(), add check for user status
 */
public function storeLogin(Request $request)
{
    $credentials = $request->validate([
        'email' => 'required|email',
        'password' => 'required|string',
    ]);

    $user = User::where('email', $credentials['email'])->first();
    
    // Check email verification
    if ($user && is_null($user->email_verified_at)) {
        return back()->withErrors([
            'email' => 'Please verify your email before logging in.',
        ])->onlyInput('email');
    }
    
    // Check user status (ADDED)
    if ($user && $user->status !== 'active') {
        return back()->withErrors([
            'email' => 'Your account has been deactivated. Please contact support.',
        ])->onlyInput('email');
    }

    if (Auth::attempt($credentials)) {
        $request->session()->regenerate();

        if (Auth::user()->role === 'admin') {
            return redirect()->route('dashboard');
        }

        return redirect()->route('home');
    }

    return back()->withErrors([
        'email' => 'The provided credentials do not match our records.',
    ])->onlyInput('email');
}
```

---

## Database Schema

```sql
CREATE TABLE users (
    id BIGINT UNSIGNED PRIMARY KEY AUTO_INCREMENT,
    first_name VARCHAR(255) NOT NULL,
    last_name VARCHAR(255) NOT NULL,
    email VARCHAR(255) UNIQUE NOT NULL,
    password VARCHAR(255) NOT NULL,
    role VARCHAR(255) DEFAULT 'customer',  -- 'customer' or 'admin'
    phone VARCHAR(50) NULLABLE,
    address TEXT NULLABLE,
    photo VARCHAR(255) NULLABLE,
    status VARCHAR(255) DEFAULT 'active',  -- 'active' or 'inactive'
    email_verified_at TIMESTAMP NULL,      -- NULL if not verified
    remember_token VARCHAR(100) NULLABLE,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP
);
```

---

## Summary

**MP2 implements complete user management with:**
- ✅ Strong password requirements (uppercase, lowercase, number, special char)
- ✅ Optional profile photo upload with validation
- ✅ Email verification system
- ✅ DataTable listing with pagination & advanced search
- ✅ Admin can manage user roles (customer/admin)
- ✅ Admin can manage user status (active/inactive)  
- ✅ Inactive users cannot login
- ✅ Admins cannot modify their own role/status (security)
- ✅ Photo storage with fallback badges

**Total Implementation: 20 Points**
