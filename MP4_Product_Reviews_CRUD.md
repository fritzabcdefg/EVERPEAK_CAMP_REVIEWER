# MP4: Product Review CRUD

## Overview
**Total Points: Built-in to system**
- Stage 1: Create & View Reviews (8pts)
- Stage 2: Update & Delete Reviews (10pts)  
- Stage 3: DataTable Review Management (5pts)
- Stage 4: Admin Delete Capability (5pts)

---

## Review Model

```php
<?php

namespace App\Models;

use Illuminate\Database\Eloquent\Model;
use Illuminate\Database\Eloquent\Relations\BelongsTo;

class Review extends Model
{
    protected $table = 'reviews';
    protected $primaryKey = 'review_id';  // Custom primary key
    public $timestamps = true;

    protected $fillable = [
        'product_id',    // Which product is being reviewed
        'user_id',       // Who wrote the review
        'rating',        // 1-5 stars
        'comment',       // Optional text review
    ];

    protected $casts = [
        'rating' => 'integer',
    ];

    /**
     * Review belongs to one Product
     */
    public function product(): BelongsTo
    {
        return $this->belongsTo(Product::class, 'product_id', 'product_id');
    }

    /**
     * Review belongs to one User
     */
    public function user(): BelongsTo
    {
        return $this->belongsTo(User::class, 'user_id');
    }
}
```

---

## ReviewController Methods

### Stage 1-2: Create & Update Reviews

#### store() - Create Review

```php
/**
 * Store a new review
 * Only users who purchased the product (completed order) can review
 * 
 * @param Request $request
 * @return \Illuminate\Http\JsonResponse
 */
public function store(Request $request)
{
    // Authenticate user
    if (!Auth::check()) {
        return response()->json(['error' => 'Please login to leave a review'], 401);
    }

    $user_id = Auth::id();

    // Validate review input
    $validated = $request->validate([
        'product_id' => 'required|integer|exists:products,product_id',
        'rating' => 'required|integer|min:1|max:5',
        'comment' => 'nullable|string|max:1000',
    ]);

    // Check if user has purchased this product (completed order)
    if (!$this->hasPurchasedProduct($user_id, $validated['product_id'])) {
        return response()->json([
            'error' => 'You can only review products you have purchased and received.'
        ], 403);
    }

    // Check if user already reviewed this product
    $existingReview = Review::where('product_id', $validated['product_id'])
                             ->where('user_id', $user_id)
                             ->first();

    if ($existingReview) {
        return response()->json([
            'error' => 'You have already reviewed this product. You can update your review instead.'
        ], 422);
    }

    // Create review
    $review = Review::create([
        'product_id' => $validated['product_id'],
        'user_id' => $user_id,
        'rating' => $validated['rating'],
        'comment' => $validated['comment'] ?? '',
    ]);

    return response()->json([
        'success' => true,
        'message' => 'Review posted successfully!',
        'review' => $review,
    ]);
}

/**
 * Check if user has purchased and completed order for product
 * Only users who received the product can review it
 */
protected function hasPurchasedProduct($userId, $productId)
{
    return Order::whereHas('orderItems', function ($query) use ($productId) {
        $query->where('product_id', $productId);
    })
    ->where('user_id', $userId)
    ->where('status', 'completed')  // Only completed orders
    ->exists();
}
```

#### update() - Update Review

```php
/**
 * Update an existing review
 * User can only update their own reviews
 * Admin can update any review
 * 
 * @param Request $request
 * @param Review $review
 * @return \Illuminate\Http\JsonResponse
 */
public function update(Request $request, Review $review)
{
    // Check authorization: only author or admin can update
    if (Auth::id() !== $review->user_id && Auth::user()->role !== 'admin') {
        return response()->json(['error' => 'Unauthorized'], 403);
    }

    // Validate updated data
    $validated = $request->validate([
        'rating' => 'required|integer|min:1|max:5',
        'comment' => 'nullable|string|max:1000',
    ]);

    // Update review
    $review->update($validated);

    return response()->json([
        'success' => true,
        'message' => 'Review updated successfully!',
        'review' => $review,
    ]);
}
```

---

### Stage 3: DataTable Admin Review Management

#### index() - Show DataTable View

```php
/**
 * Show admin review management page
 * @return \Illuminate\View\View
 */
public function index(Request $request)
{
    // Admin only
    if (!Auth::check() || Auth::user()->role !== 'admin') {
        return redirect()->route('home')
                        ->with('error', 'Unauthorized access.');
    }

    return view('reviews.index');
}
```

#### datatable() - API Endpoint

```php
/**
 * Get reviews data for DataTables server-side processing
 * 
 * @param Request $request
 * @return \Illuminate\Http\JsonResponse
 */
public function datatable(Request $request)
{
    try {
        // Admin authorization check
        if (!Auth::check() || Auth::user()->role !== 'admin') {
            return response()->json(['error' => 'Unauthorized'], 401);
        }

        // Base query with eager loading
        $query = Review::select('reviews.*')
                      ->selectRaw('review_id as id')
                      ->with(['user', 'product'])
                      ->orderBy('reviews.created_at', 'desc');

        return DataTables::of($query)
            ->setRowId('review_id')
            
            // Display review ID
            ->addColumn('review_id', function ($review) {
                return '<strong>#' . $review->review_id . '</strong>';
            })
            
            // Display product name
            ->addColumn('product', function ($review) {
                return $review->product?->name ?? 'N/A';
            })
            
            // Display customer name
            ->addColumn('customer', function ($review) {
                return $review->user 
                    ? $review->user->first_name . ' ' . $review->user->last_name 
                    : 'N/A';
            })
            
            // Display rating with stars
            ->addColumn('rating', function ($review) {
                $stars = str_repeat('⭐', $review->rating) . ' ' . $review->rating . '/5';
                return '<span class="badge bg-warning text-dark">' . $stars . '</span>';
            })
            
            // Display comment (truncated)
            ->addColumn('comment', function ($review) {
                return '<small>' . \Illuminate\Support\Str::limit($review->comment ?? 'No comment', 50) . '</small>';
            })
            
            // Format review date
            ->addColumn('date', function ($review) {
                return $review->created_at->format('M d, Y');
            })
            
            // Admin delete action
            ->addColumn('actions', function ($review) {
                $actions = '<form action="' . route('reviews.destroy', $review) . '" method="POST" style="display:inline;">';
                $actions .= '<input type="hidden" name="_method" value="DELETE">';
                $actions .= '<input type="hidden" name="_token" value="' . csrf_token() . '">';
                $actions .= '<button type="submit" class="btn btn-sm btn-danger" title="Delete" onclick="return confirm(\'Are you sure?\')">';
                $actions .= '<i class="fas fa-trash"></i> Delete</button>';
                $actions .= '</form>';
                return $actions;
            })
            
            // Custom search on product name
            ->filterColumn('product', function ($query, $keyword) {
                $query->whereHas('product', function($pq) use ($keyword) {
                    $pq->where('name', 'like', "%{$keyword}%");
                });
            })
            
            // Custom search on customer name/email and comment
            ->filterColumn('customer', function ($query, $keyword) {
                $query->whereHas('user', function($uq) use ($keyword) {
                    $uq->where('first_name', 'like', "%{$keyword}%")
                       ->orWhere('last_name', 'like', "%{$keyword}%")
                       ->orWhere('email', 'like', "%{$keyword}%");
                })
                ->orWhere('comment', 'like', "%{$keyword}%");
            })
            
            // Raw HTML columns
            ->rawColumns(['review_id', 'rating', 'comment', 'actions'])
            ->make(true);
            
    } catch (\Exception $e) {
        \Log::error('Review DataTable error: ' . $e->getMessage());
        return response()->json(['error' => 'Server error'], 500);
    }
}
```

---

### Stage 4: Admin Delete

#### destroy() - Delete Review

```php
/**
 * Delete a review
 * Users can only delete their own reviews
 * Admins can delete any review
 * 
 * @param Review $review
 * @return \Illuminate\Http\RedirectResponse
 */
public function destroy(Review $review)
{
    // Authorization: author or admin only
    if (Auth::id() !== $review->user_id && Auth::user()->role !== 'admin') {
        return redirect()->back()
                        ->with('error', 'Unauthorized to delete this review');
    }

    // Get product for redirect back
    $product = $review->product;
    
    // Delete review
    $review->delete();

    // Redirect based on context
    if (Auth::user()->role === 'admin') {
        return redirect()->route('reviews.index')
                        ->with('success', 'Review deleted successfully');
    } else {
        return redirect()->route('products.show', $product)
                        ->with('success', 'Your review has been deleted');
    }
}
```

---

## Blade View - Product Review Form (Modal)

```blade
<!-- Modal for review form -->
<div class="modal fade" id="reviewModal" tabindex="-1">
    <div class="modal-dialog">
        <div class="modal-content">
            <div class="modal-header">
                <h5 class="modal-title">Leave a Review</h5>
                <button type="button" class="btn-close" data-bs-dismiss="modal"></button>
            </div>
            
            <form id="reviewForm" method="POST">
                @csrf
                <input type="hidden" name="product_id" value="{{ $product->product_id }}">
                
                <div class="modal-body">
                    <!-- Rating (1-5 stars) -->
                    <div class="mb-3">
                        <label class="form-label">Rating</label>
                        <div class="rating-stars">
                            @for ($i = 1; $i <= 5; $i++)
                                <input type="radio" name="rating" value="{{ $i }}" id="star{{ $i }}">
                                <label for="star{{ $i }}">★</label>
                            @endfor
                        </div>
                        <span id="ratingDisplay" class="text-muted">Click to rate</span>
                    </div>
                    
                    <!-- Comment -->
                    <div class="mb-3">
                        <label for="comment" class="form-label">Your Review (Optional)</label>
                        <textarea class="form-control" id="comment" name="comment" 
                                  rows="4" placeholder="Share your experience with this product..." 
                                  maxlength="1000"></textarea>
                        <small class="form-text text-muted">Max 1000 characters</small>
                    </div>
                    
                    <!-- Purchase requirement message -->
                    @if(!$canReview)
                        <div class="alert alert-warning">
                            You must purchase and receive this product before you can leave a review.
                        </div>
                    @endif
                </div>
                
                <div class="modal-footer">
                    <button type="button" class="btn btn-secondary" data-bs-dismiss="modal">Cancel</button>
                    <button type="submit" class="btn btn-primary" @if(!$canReview) disabled @endif>
                        Post Review
                    </button>
                </div>
            </form>
        </div>
    </div>
</div>

<script>
// Handle star rating selection
document.querySelectorAll('input[name="rating"]').forEach(star => {
    star.addEventListener('change', function() {
        document.getElementById('ratingDisplay').textContent = this.value + '/5 stars';
    });
});

// Handle form submission
document.getElementById('reviewForm').addEventListener('submit', function(e) {
    e.preventDefault();
    
    const formData = new FormData(this);
    
    fetch('{{ route("reviews.store") }}', {
        method: 'POST',
        headers: {
            'X-CSRF-TOKEN': '{{ csrf_token() }}'
        },
        body: formData
    })
    .then(response => response.json())
    .then(data => {
        if (data.success) {
            alert('Review posted successfully!');
            location.reload();
        } else {
            alert(data.error);
        }
    })
    .catch(error => console.error('Error:', error));
});
</script>

<style>
.rating-stars input[type="radio"] {
    display: none;
}

.rating-stars label {
    font-size: 2rem;
    cursor: pointer;
    color: #ddd;
    margin: 0 5px;
}

.rating-stars input[type="radio"]:checked ~ label,
.rating-stars label:hover {
    color: #ffc107;
}
</style>
```

---

## Database Schema

```sql
CREATE TABLE reviews (
    review_id BIGINT UNSIGNED PRIMARY KEY AUTO_INCREMENT,
    product_id BIGINT UNSIGNED NOT NULL,
    user_id BIGINT UNSIGNED NOT NULL,
    rating INTEGER NOT NULL CHECK (rating >= 1 AND rating <= 5),
    comment TEXT NULLABLE,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    
    FOREIGN KEY (product_id) REFERENCES products(product_id) ON DELETE CASCADE,
    FOREIGN KEY (user_id) REFERENCES users(id) ON DELETE CASCADE,
    UNIQUE KEY (product_id, user_id)  -- One review per product per user
);
```

---

## Summary

**MP4 implements complete review system with:**
- ✅ Only purchase-verified users can review
- ✅ One review per product per user
- ✅ 1-5 star rating system
- ✅ Optional comment text (max 1000 chars)
- ✅ Users can edit their reviews
- ✅ Users can delete their reviews
- ✅ Admin can delete any review
- ✅ DataTable for admin review management
- ✅ Search by product name, customer name/email
