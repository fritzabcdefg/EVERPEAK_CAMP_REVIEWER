# MP3: Authentication System

## Overview
**Total Points: Built-in to registration/login system**
- Email verification after registration
- Only verified users can login
- Admin routes protected with IsAdmin middleware
- Unauthenticated and unauthorized users blocked

---

## Email Verification System

### EmailVerificationToken Model

```php
<?php

namespace App\Models;

use Illuminate\Database\Eloquent\Model;
use Illuminate\Database\Eloquent\Relations\BelongsTo;

class EmailVerificationToken extends Model
{
    protected $table = 'email_verification_tokens';
    protected $fillable = ['user_id', 'token'];
    public $timestamps = false;  // Only track created_at
    const CREATED_AT = 'created_at';
    const UPDATED_AT = null;

    /**
     * Each token belongs to one user
     */
    public function user(): BelongsTo
    {
        return $this->belongsTo(User::class, 'user_id');
    }

    /**
     * Check if token is still valid (created within last 24 hours)
     */
    public function isValid(): bool
    {
        // Token expires after 24 hours
        return $this->created_at->addHours(24)->isFuture();
    }
}
```

### VerifyEmailMail Class

```php
<?php

namespace App\Mail;

use App\Models\User;
use Illuminate\Bus\Queueable;
use Illuminate\Mail\Mailable;
use Illuminate\Mail\Mailables\Envelope;
use Illuminate\Mail\Mailables\Content;

class VerifyEmailMail extends Mailable
{
    use Queueable;

    public function __construct(public User $user, public string $token)
    {
    }

    /**
     * Get the message envelope.
     */
    public function envelope(): Envelope
    {
        return new Envelope(
            subject: 'Verify Your Email Address - EverPeak Camp',
        );
    }

    /**
     * Get the message content definition.
     */
    public function content(): Content
    {
        // Generate verification link
        $verificationLink = route('email.verify', ['token' => $this->token]);

        return new Content(
            view: 'emails.verify-email',
            with: [
                'user' => $this->user,
                'verificationLink' => $verificationLink,
                'expiresAt' => now()->addHours(24)->format('M d, Y H:i'),
            ],
        );
    }
}
```

### Blade Email Template

```blade
<!-- resources/views/emails/verify-email.blade.php -->
<h1>Welcome to EverPeak Camp!</h1>

<p>Hi {{ $user->first_name }},</p>

<p>Thank you for registering! Please verify your email address by clicking the button below:</p>

<a href="{{ $verificationLink }}" class="btn btn-primary" style="padding: 10px 20px; background-color: #007bff; color: white; text-decoration: none; border-radius: 5px;">
    Verify Email Address
</a>

<p style="color: #666; font-size: 12px;">
    Or copy and paste this link: <br>
    {{ $verificationLink }}
</p>

<p style="color: red; font-weight: bold;">
    This link expires at: {{ $expiresAt }}
</p>

<p>If you did not create this account, please ignore this email.</p>

<p>Best regards,<br>EverPeak Camp Team</p>
```

### UserController - verifyEmail() Method

```php
/**
 * Verify user email with token sent via email
 * 
 * @param string $token - Verification token from email link
 * @return \Illuminate\Http\RedirectResponse
 */
public function verifyEmail($token)
{
    // Find token in database
    $verificationToken = EmailVerificationToken::where('token', $token)->first();

    // Token not found
    if (!$verificationToken) {
        return redirect()->route('login')
                        ->withErrors(['email' => 'Invalid verification link.']);
    }

    // Check if token is expired (older than 24 hours)
    if (!$verificationToken->isValid()) {
        // Delete expired token
        $verificationToken->delete();
        return redirect()->route('login')
                        ->withErrors(['email' => 'Verification link has expired. Please register again or request a new verification link.']);
    }

    // Token is valid - mark user as verified
    $user = $verificationToken->user;
    $user->update(['email_verified_at' => now()]);

    // Delete token after using it
    $verificationToken->delete();

    // Refresh user from database
    $user = $user->fresh();
    
    // Auto-login verified user
    Auth::login($user);

    return redirect()->route('profile.create')
                    ->with('success', 'Email verified successfully! Please complete your profile.');
}
```

### UserController - resendVerification() Method

```php
/**
 * Resend verification email to unverified user
 * 
 * @param Request $request - Must contain 'email' field
 * @return \Illuminate\Http\RedirectResponse
 */
public function resendVerification(Request $request)
{
    $validated = $request->validate([
        'email' => 'required|email|exists:users',
    ]);

    $user = User::where('email', $validated['email'])->first();

    // Only allow resend for unverified users
    if ($user->email_verified_at) {
        return back()->with('info', 'This email is already verified.');
    }

    // Delete old tokens before creating new one
    EmailVerificationToken::where('user_id', $user->id)->delete();

    // Generate new verification token
    $token = Str::random(64);
    EmailVerificationToken::create([
        'user_id' => $user->id,
        'token' => $token,
        'created_at' => now(),
    ]);

    // Send verification email
    Mail::send(new VerifyEmailMail($user, $token));

    return back()->with('success', 'A new verification link has been sent to your email.');
}
```

---

## IsAdmin Middleware

### Purpose
Protects admin routes by checking:
1. User is authenticated
2. User's email is verified
3. User has 'admin' role

```php
<?php

namespace App\Http\Middleware;

use Illuminate\Support\Facades\Auth;

class IsAdmin
{
    /**
     * Handle an incoming request.
     * 
     * Three-layer security check:
     * 1. Authentication: User must be logged in
     * 2. Email Verification: Logged-in user must have verified email
     * 3. Authorization: User must have admin role
     */
    public function handle(Request $request, Closure $next)
    {
        // Layer 1: Check authentication
        if (!Auth::check()) {
            if ($request->expectsJson()) {
                return response()->json(['error' => 'Unauthenticated'], 401);
            }
            return redirect()->route('login')
                            ->with('error', 'Please log in to continue.');
        }

        // Layer 2: Check email verification
        // If email_verified_at is NULL, user hasn't verified email yet
        if (is_null(Auth::user()->email_verified_at)) {
            // Force logout for security
            Auth::logout();
            
            if ($request->expectsJson()) {
                return response()->json(['error' => 'Email not verified'], 403);
            }
            return redirect()->route('login')
                            ->with('error', 'Your email must be verified to access admin features.');
        }

        // Layer 3: Check admin role
        if (Auth::user()->role !== 'admin') {
            if ($request->expectsJson()) {
                return response()->json(['error' => 'Forbidden'], 403);
            }
            return redirect()->route('home')
                            ->with('error', 'You do not have permission to access this page.');
        }

        // All checks passed - proceed to route
        return $next($request);
    }
}
```

### Application in Routes

```php
// In routes/web.php

// Admin routes protected by IsAdmin middleware
Route::middleware(['auth', 'admin'])->group(function () {
    // Products
    Route::resource('products', ProductController::class);
    
    // Users
    Route::resource('users', UserController::class);
    
    // Reviews
    Route::resource('reviews', ReviewController::class);
    
    // Dashboard
    Route::get('/dashboard', [DashboardController::class, 'index'])->name('dashboard');
});

// Public/Auth routes (no admin check)
Route::get('/register', [UserController::class, 'createRegister'])->name('register');
Route::post('/register', [UserController::class, 'storeRegister'])->name('register.store');

Route::get('/login', [UserController::class, 'createLogin'])->name('login');
Route::post('/login', [UserController::class, 'storeLogin'])->name('login.store');

Route::post('/logout', [UserController::class, 'logout'])->name('logout');

// Email verification
Route::get('/email/verify/{token}', [UserController::class, 'verifyEmail'])->name('email.verify');
Route::post('/email/resend', [UserController::class, 'resendVerification'])->name('email.resend');
```

### Database Schema

```sql
CREATE TABLE email_verification_tokens (
    id BIGINT UNSIGNED PRIMARY KEY AUTO_INCREMENT,
    user_id BIGINT UNSIGNED NOT NULL,
    token VARCHAR(255) UNIQUE NOT NULL,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    
    FOREIGN KEY (user_id) REFERENCES users(id) ON DELETE CASCADE
);
```

---

## Authentication Flow Diagram

```
1. USER REGISTERS
   ├─ Fills: first_name, last_name, email, password, optional photo
   ├─ Password validated: min 8 chars, uppercase, lowercase, number, special char
   ├─ Photo uploaded: optional, max 2MB, min 100x100px
   ├─ User created with email_verified_at = NULL
   ├─ Verification token created & stored in database
   └─ Verification email sent to user

2. USER CLICKS EMAIL LINK
   ├─ Token extracted from URL: /email/verify/{token}
   ├─ Token validated: exists and not expired (24 hours)
   ├─ User email_verified_at set to NOW()
   ├─ Token deleted from database
   ├─ User auto-logged in
   └─ Redirect to profile completion

3. USER LOGS IN
   ├─ Email & password sent to login form
   ├─ User lookup by email
   ├─ Check: email_verified_at IS NOT NULL (if NULL → error)
   ├─ Check: status = 'active' (if inactive → error)
   ├─ Password comparison (bcrypt)
   ├─ Session created
   └─ Redirect to dashboard (if admin) or home (if customer)

4. ADMIN ACCESS
   ├─ Any admin route triggers IsAdmin middleware
   ├─ Check 1: Auth::check() → must be logged in
   ├─ Check 2: email_verified_at → must be verified
   ├─ Check 3: role === 'admin' → must be admin
   ├─ If any fails → redirect to login or home
   └─ If all pass → proceed to controller method
```

---

## Summary

**MP3 implements complete authentication with:**
- ✅ Email verification required after registration
- ✅ 24-hour expiring verification tokens
- ✅ Resend verification email functionality
- ✅ Only verified users can login
- ✅ Only verified+admin users can access admin routes
- ✅ Three-layer security middleware (auth, email, role)
- ✅ Inactive users blocked from login
- ✅ CSRF protection on all forms
- ✅ Password hashing with bcrypt
