# MP6: Product Filtering

## Overview
**Total Points: 25pts**
- Stage 1: Price Filtering (10pts) ✅
- Stage 2: Multi-Criteria Filtering (15pts) ✅

---

## Filtering Features

### Stage 1: Price Filtering (10pts)

#### Implementation in ShopController

```php
/**
 * Show shop with price filtering
 * Supports minimum and maximum price filters
 */
public function show(Request $request)
{
    $search = trim((string) $request->get('search', ''));
    $selectedCategories = $request->input('category', []);
    
    // Get price filter parameters from URL query string
    // Examples: ?min_price=10&max_price=100
    $minPrice = $request->get('min_price', '');  // Default empty = no minimum
    $maxPrice = $request->get('max_price', '');  // Default empty = no maximum

    // Perform search with filters
    $products = $this->searchWithLike($search, $selectedCategories, $minPrice, $maxPrice, $request);

    // Get categories for filter dropdown
    $categories = Category::orderBy('name')->get();

    // Get min/max prices from database for filter UI display
    $priceStats = Product::selectRaw('MIN(sell_price) as min_price, MAX(sell_price) as max_price')->first();

    return view('shop.show', [
        'products' => $products,
        'categories' => $categories,
        'search' => $search,
        'selectedCategories' => $selectedCategories,
        'minPrice' => $minPrice,
        'maxPrice' => $maxPrice,
        'priceStats' => $priceStats,
    ]);
}

/**
 * Apply price filters to query
 */
private function applyCommonFilters($query, $selectedCategories, $minPrice, $maxPrice, Request $request)
{
    // Filter by category if selected
    if (!empty($selectedCategories)) {
        $query->whereIn('category_id', $selectedCategories);
    }

    // Filter by minimum price: sell_price >= minPrice
    if ($minPrice !== '') {
        $query->where('sell_price', '>=', (float) $minPrice);
    }

    // Filter by maximum price: sell_price <= maxPrice
    if ($maxPrice !== '') {
        $query->where('sell_price', '<=', (float) $maxPrice);
    }

    // Handle sorting
    $sortBy = $request->get('sort_by', 'name');
    $sortOrder = $request->get('sort_order', 'asc');

    // Only allow specific columns for security
    if (in_array($sortBy, ['name', 'sell_price', 'created_at'])) {
        $query->orderBy($sortBy, in_array($sortOrder, ['asc', 'desc']) ? $sortOrder : 'asc');
    } else {
        $query->orderBy('name', 'asc');
    }

    // Paginate with query parameters preserved in links
    return $query->paginate(12)->appends($request->query());
}
```

### Stage 2: Multi-Criteria Filtering (15pts)

```php
/**
 * Handle multiple filter criteria
 * - Product name/description search
 * - Multiple categories
 * - Price range (min & max)
 * - Sorting by name, price, or date
 */
private function searchWithLike($search, $selectedCategories, $minPrice, $maxPrice, Request $request)
{
    $query = Product::with('category', 'stock');

    // FILTER 1: Search by product name, description, or category name
    if ($search !== '') {
        $query->where(function ($q) use ($search) {
            // Search in product name OR description OR category name (LIKE - case insensitive)
            $q->where('name', 'like', "%{$search}%")
              ->orWhere('description', 'like', "%{$search}%")
              ->orWhereHas('category', function ($categoryQuery) use ($search) {
                  $categoryQuery->where('name', 'like', "%{$search}%");
              });
        });
    }

    // Exclude soft-deleted products
    $query->whereNull('deleted_at');

    // Apply common filters (category, price, sorting)
    return $this->applyCommonFilters($query, $selectedCategories, $minPrice, $maxPrice, $request);
}

/**
 * Apply all filters together
 */
private function applyCommonFilters($query, $selectedCategories, $minPrice, $maxPrice, Request $request)
{
    // FILTER 2: Category filtering (multiple select)
    // If user selects multiple categories, show products from ANY selected category (OR logic)
    if (!empty($selectedCategories)) {
        $query->whereIn('category_id', $selectedCategories);
    }

    // FILTER 3: Minimum price filtering
    if ($minPrice !== '') {
        $minPrice = (float) $minPrice;  // Cast to float for safety
        $query->where('sell_price', '>=', $minPrice);  // >= operator
    }

    // FILTER 4: Maximum price filtering
    if ($maxPrice !== '') {
        $maxPrice = (float) $maxPrice;
        $query->where('sell_price', '<=', $maxPrice);  // <= operator
    }

    // FILTER 5: Sorting
    $sortBy = $request->get('sort_by', 'name');    // Default: sort by name
    $sortOrder = $request->get('sort_order', 'asc');  // Default: ascending order

    // Only match valid columns (prevent SQL injection)
    if (in_array($sortBy, ['name', 'sell_price', 'created_at'])) {
        // Validate sort order (only ASC or DESC allowed)
        $validOrder = in_array($sortOrder, ['asc', 'desc']) ? $sortOrder : 'asc';
        $query->orderBy($sortBy, $validOrder);
    } else {
        // Default fallback: sort by name ascending
        $query->orderBy('name', 'asc');
    }

    // Paginate and preserve all filter query parameters in pagination links
    return $query->paginate(12)->appends($request->query());
}
```

---

## Blade View - Shop Filter UI

```blade
<!-- resources/views/shop/show.blade.php -->

<div class="container-fluid mt-4">
    <div class="row">
        <!-- FILTERS SIDEBAR -->
        <div class="col-md-3">
            <div class="card">
                <div class="card-header">
                    <h5>Filters</h5>
                </div>
                <div class="card-body">
                    <form method="GET" action="{{ route('shop.show') }}" id="filterForm">
                        
                        <!-- SEARCH FILTER -->
                        <div class="mb-3">
                            <label for="search" class="form-label">Search</label>
                            <input type="text" class="form-control" id="search" name="search" 
                                   value="{{ $search }}" placeholder="Product name...">
                        </div>
                        
                        <!-- CATEGORY FILTER (Multiple Select) -->
                        <div class="mb-3">
                            <label for="category" class="form-label">Categories</label>
                            <div class="category-list">
                                @foreach($categories as $category)
                                    <div class="form-check">
                                        <input class="form-check-input" type="checkbox" 
                                               name="category[]" 
                                               value="{{ $category->category_id }}"
                                               id="category_{{ $category->category_id }}"
                                               {{ in_array($category->category_id, $selectedCategories) ? 'checked' : '' }}
                                               onchange="document.getElementById('filterForm').submit();">
                                        <label class="form-check-label" for="category_{{ $category->category_id }}">
                                            {{ $category->name }}
                                        </label>
                                    </div>
                                @endforeach
                            </div>
                        </div>
                        
                        <!-- PRICE FILTER -->
                        <div class="mb-3">
                            <label class="form-label">Price Range</label>
                            <div class="price-range d-flex gap-2">
                                <!-- Minimum Price Input -->
                                <input type="number" name="min_price" class="form-control form-control-sm" 
                                       placeholder="Min" value="{{ $minPrice }}" 
                                       min="0" step="0.01">
                                
                                <!-- Maximum Price Input -->
                                <input type="number" name="max_price" class="form-control form-control-sm" 
                                       placeholder="Max" value="{{ $maxPrice }}" 
                                       min="0" step="0.01">
                            </div>
                            <small class="text-muted">Available: ${{ number_format($priceStats->min_price, 2) }} - ${{ number_format($priceStats->max_price, 2) }}</small>
                        </div>
                        
                        <!-- SORTING -->
                        <div class="mb-3">
                            <label for="sort_by" class="form-label">Sort By</label>
                            <select name="sort_by" id="sort_by" class="form-select form-select-sm" 
                                    onchange="document.getElementById('filterForm').submit();">
                                <option value="name" {{ $sortBy === 'name' ? 'selected' : '' }}>Name</option>
                                <option value="sell_price" {{ $sortBy === 'sell_price' ? 'selected' : '' }}>Price</option>
                                <option value="created_at" {{ $sortBy === 'created_at' ? 'selected' : '' }}>Newest</option>
                            </select>
                        </div>
                        
                        <!-- ACTION BUTTONS -->
                        <div class="d-grid gap-2">
                            <button type="submit" class="btn btn-primary btn-sm">Apply Filters</button>
                            <a href="{{ route('shop.show') }}" class="btn btn-secondary btn-sm">Clear All</a>
                        </div>
                    </form>
                </div>
            </div>
        </div>
        
        <!-- PRODUCTS GRID -->
        <div class="col-md-9">
            <div class="row">
                @forelse($products as $product)
                    <div class="col-md-4 mb-4">
                        <div class="card h-100">
                            <!-- Product Image -->
                            <img src="{{ $product->img_path ? Storage::url($product->img_path) : asset('images/no-image.png') }}" 
                                 class="card-img-top" alt="{{ $product->name }}" 
                                 style="height: 200px; object-fit: cover;">
                            
                            <div class="card-body">
                                <!-- Product Category Badge -->
                                @if($product->category)
                                    <span class="badge bg-info mb-2">{{ $product->category->name }}</span>
                                @endif
                                
                                <!-- Product Name -->
                                <h5 class="card-title">{{ $product->name }}</h5>
                                
                                <!-- Description (truncated) -->
                                <p class="card-text text-muted small">
                                    {{ \Illuminate\Support\Str::limit($product->description, 80) }}
                                </p>
                                
                                <!-- Stock Status -->
                                <p class="mb-2">
                                    @php
                                        $stock = $product->stock->sum('quantity');
                                    @endphp
                                    @if($stock > 0)
                                        <span class="badge bg-success">In Stock ({{ $stock }})</span>
                                    @else
                                        <span class="badge bg-danger">Out of Stock</span>
                                    @endif
                                </p>
                                
                                <!-- Price -->
                                <h4 class="text-primary">${{ number_format($product->sell_price, 2) }}</h4>
                            </div>
                            
                            <!-- Action Buttons -->
                            <div class="card-footer bg-light">
                                <a href="{{ route('products.show', $product->product_id) }}" class="btn btn-sm btn-primary">
                                    View Details
                                </a>
                                @if($stock > 0)
                                    <button class="btn btn-sm btn-success" onclick="addToCart({{ $product->product_id }})">
                                        Add to Cart
                                    </button>
                                @endif
                            </div>
                        </div>
                    </div>
                @empty
                    <div class="col-12">
                        <div class="alert alert-info" role="alert">
                            <h4>No products found</h4>
                            <p>Try adjusting your filters or search terms.</p>
                        </div>
                    </div>
                @endforelse
            </div>
            
            <!-- PAGINATION with filters preserved -->
            <div class="d-flex justify-content-center">
                {{ $products->links('pagination::bootstrap-5') }}
            </div>
        </div>
    </div>
</div>

<script>
function addToCart(productId) {
    // Add to cart functionality
    alert('Added to cart: ' + productId);
}
</script>
```

---

## Query Examples

```
# Price Filtering Only
GET /shop?min_price=10&max_price=100

# Category Filtering
GET /shop?category[]=1&category[]=3

# Search + Price Filter
GET /shop?search=tent&min_price=50&max_price=200

# Multiple Filters + Sort
GET /shop?search=camping&category[]=1&min_price=25&max_price=150&sort_by=sell_price&sort_order=asc

# All Parameters
GET /shop?search=tent&category[]=1&category[]=2&min_price=10&max_price=500&sort_by=created_at&sort_order=desc
```

---

## Database Query

```sql
-- Example Query with all filters applied
SELECT products.*, categories.name as category_name
FROM products
LEFT JOIN categories ON products.category_id = categories.category_id
WHERE 
    (products.name LIKE '%tent%' 
     OR products.description LIKE '%tent%'
     OR categories.name LIKE '%tent%')
    AND (category_id IN (1, 3))
    AND (sell_price >= 25)
    AND (sell_price <= 150)
    AND (deleted_at IS NULL)
ORDER BY products.sell_price ASC
LIMIT 12;
```

---

## Summary

**MP6 implements comprehensive filtering with:**
- ✅ Minimum price filter (>= operator)
- ✅ Maximum price filter (<= operator)
- ✅ Multiple category selection (OR logic)
- ✅ Product name & description search
- ✅ Category name search
- ✅ Sorting by name, price, or date (ASC/DESC)
- ✅ Combined filter support
- ✅ Pagination with filter preservation
- ✅ Pricing statistics display
- ✅ Stock level indicators

**Total Implementation: 25 Points**
