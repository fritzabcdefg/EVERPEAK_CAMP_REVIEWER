# MP8: Product Search (3 Methods)

## Overview
**Total Points: 30pts**
- Method 1: LIKE Query Search (8pts) ✅
- Method 2: Model Scope Search (10pts) ✅
- Method 3: Laravel Scout Full-Text Search (12pts) ✅

---

## Method 1: LIKE Query Search (8pts)

### Direct SQL LIKE Query

```php
/**
 * Search using LIKE query (8pts)
 * Direct SQL LIKE operator - simplest and most reliable
 * Searches product name and description
 * 
 * SQL: WHERE name LIKE '%search%' OR description LIKE '%search%'
 */
private function performLikeSearch(Request $request, string $search, int $perPage, &$products): void
{
    $products = Product::with('category', 'stock')
        // Exclude soft-deleted products
        ->whereNull('deleted_at')
        // Search in name or description using LIKE operator
        ->where(function ($query) use ($search) {
            // LIKE '%search%' = contains the search term (case-insensitive)
            $query->where('name', 'LIKE', "%{$search}%")
                  ->orWhere('description', 'LIKE', "%{$search}%");
        })
        // Paginate results (12 per page)
        ->paginate($perPage)
        // Preserve search query string in pagination links
        ->withQueryString();
}
```

### How LIKE Works

```
Pattern          Matches
'%tent%'         Contains "tent": "camping tent", "tent poles", "tent-shaped"
'tent%'          Starts with "tent": "tent", "tents", "tent-poles"
'%tent'          Ends with "tent": "camping tent", "a tent"  
'_tent'          Single char + "tent": "atent", "btent", "ctent"
```

### Performance Characteristics

| Aspect | LIKE |
|--------|------|
| Speed | ~OK for <10k records |
| Index Support | Limited (needs index on column) |
| Search Quality | Basic pattern matching |
| Database Load | Moderate |

---

## Method 2: Model Scope Search (10pts)

### Eloquent Scope

```php
/**
 * Search using Model scope (10pts)
 * Cleaner code using Eloquent scope method
 * Encapsulates search logic
 * More testable and maintainable
 */
private function performModelSearch(Request $request, string $search, int $perPage, &$products): void
{
    $products = Product::with('category', 'stock')
        ->whereNull('deleted_at')
        // Use the custom search() scope defined in Product model
        ->search($search)
        ->paginate($perPage)
        ->withQueryString();
}
```

### Product Model - Search Scope Definition

```php
<?php

namespace App\Models;

use Illuminate\Database\Eloquent\Builder;

class Product extends Model
{
    /**
     * Query scope: Search products by keyword
     * Custom reusable scope for searching
     * 
     * Usage: Product::search('tent')->get()
     */
    public function scopeSearch(Builder $query, ?string $term): Builder
    {
        // Trim whitespace and cast to string
        $term = trim((string) $term);

        // If search term is empty, return all
        if ($term === '') {
            return $query;
        }

        // Return query filtered by name or description LIKE
        return $query->where(function (Builder $builder) use ($term) {
            $builder->where('name', 'LIKE', "%{$term}%")
                    ->orWhere('description', 'LIKE', "%{$term}%");
        });
    }
}
```

### Why Use Scopes?

1. **Reusability**: Define once, use in multiple places
2. **Testability**: Easy to unit test
3. **Maintainability**: DRY - Don't Repeat Yourself
4. **Readability**: Self-documenting code

---

## Method 3: Laravel Scout Search (12pts)

### What is Laravel Scout?

Scout is Laravel's simple, driver-based full-text search solution.

**Drivers:**
- `database` - Uses database full-text search (built-in)
- `algolia` - Cloud-based search (premium service)
- `meilisearch` - Open-source search engine

### Configuration

```php
// config/scout.php
return [
    'driver' => env('SCOUT_DRIVER', 'database'),  // Default to database
    'prefix' => env('SCOUT_PREFIX', ''),
    
    'database' => [
        'search_input' => 'like',  // Can also use 'full text'
    ],
];

// .env
SCOUT_DRIVER=database
```

### Scout Implementation

```php
/**
 * Search using Laravel Scout (15pts)
 * Full-text search with automatic indexing
 * Better search quality and performance
 */
private function performScoutSearch(Request $request, string $search, int $perPage, &$products): void
{
    // Scout::search() initializes search
    // ->query() allows eager loading & conditions
    // ->paginate() gets results with pagination
    $products = Product::search($search)
        // Apply eager loading and soft-delete filter
        ->query(fn ($query) => 
            $query->with('category', 'stock')
                  ->whereNull('deleted_at')
        )
        // Paginate with query string preservation
        ->paginate($perPage)
        ->withQueryString();
}
```

### Product Model - Searchable Integration

```php
<?php

namespace App\Models;

use Laravel\Scout\Searchable;

class Product extends Model
{
    // Enable full-text indexing
    use Searchable;

    /**
     * Define what fields are searchable
     * Scout indexes these fields for fast searching
     */
    public function toSearchableArray(): array
    {
        return [
            'product_id' => $this->product_id,
            'name' => $this->name,
            'description' => $this->description,
        ];
    }

    /**
     * Get the index name for Scout
     */
    public function searchableAs(): string
    {
        return 'products_index';
    }
}
```

### Scout Maintenance Commands

```bash
# Index all products (create searchable index)
php artisan scout:import App\\Models\\Product

# Remove all products from index
php artisan scout:flush App\\Models\\Product

# Reindex (clear and reimport)
php artisan scout:import App\\Models\\Product --fresh

# Rebuild all indexes
php artisan scout:all
```

### Scout vs Database Search

| Aspect | LIKE Search | Model Scope | Scout |
|--------|-------------|-------------|-------|
| **Speed** | Slow (>10k records) | Slow (>10k records) | Fast (any size) |
| **Setup** | None | Define scope | Configure + Index |
| **Relevance** | Basic | Basic | Advanced ranking |
| **Scalability** | Limited | Limited | Unlimited |
| **Cost** | Free | Free | Free (database) or paid (Algolia) |
| **Use Case** | Small catalogs | Any size | Large catalogs |

---

## HomeController - Main Search Method

```php
<?php

namespace App\Http\Controllers;

use App\Models\Product;
use App\Models\Category;
use Illuminate\Http\Request;

class HomeController extends Controller
{
    /**
     * Display homepage with selectable search method
     * 
     * Query Parameters:
     * ?search=keyword              - Search term
     * ?search_type=scout|like|model - Which search method to use
     */
    public function index(Request $request)
    {
        $search = trim((string) $request->query('search', ''));
        $searchType = $request->query('search_type', 'scout');  // Default: Scout
        $perPage = 12;

        if ($search === '') {
            // No search - show all featured products
            $products = Product::with('category', 'stock')
                ->whereNull('deleted_at')
                ->latest('product_id')
                ->paginate($perPage)
                ->withQueryString();
            $searchMethod = null;
        } else {
            // Route to correct search method based on query parameter
            match ($searchType) {
                'like' => $this->performLikeSearch($request, $search, $perPage, $products),
                'model' => $this->performModelSearch($request, $search, $perPage, $products),
                'scout' => $this->performScoutSearch($request, $search, $perPage, $products),
                default => $this->performScoutSearch($request, $search, $perPage, $products),
            };
            
            $searchMethod = $searchType;
        }

        $categories = Category::orderBy('name')->get();

        return view('home', [
            'products' => $products,
            'search' => $search,
            'searchMethod' => $searchMethod,
            'categories' => $categories,
        ]);
    }

    // ... search methods as shown above
}
```

---

## Blade View - Search with Method Selection

```blade
<!-- resources/views/home.blade.php -->

<div class="container mt-4">
    <h1>Search Products</h1>
    
    <!-- Search Form with Method Selection -->
    <form method="GET" action="{{ route('home') }}" class="mb-4">
        <div class="row">
            <div class="col-md-8">
                <input type="text" name="search" class="form-control form-control-lg" 
                       placeholder="Search products..." value="{{ $search }}">
            </div>
            
            <!-- Search Method Selector -->
            <div class="col-md-3">
                <select name="search_type" class="form-select form-select-lg">
                    <option value="scout" {{ $searchMethod === 'scout' ? 'selected' : '' }}>
                        Scout (Recommended)
                    </option>
                    <option value="model" {{ $searchMethod === 'model' ? 'selected' : '' }}>
                        Model Scope
                    </option>
                    <option value="like" {{ $searchMethod === 'like' ? 'selected' : '' }}>
                        LIKE Query
                    </option>
                </select>
            </div>
            
            <div class="col-md-1">
                <button type="submit" class="btn btn-primary btn-lg w-100">Search</button>
            </div>
        </div>
        
        @if($searchMethod)
            <small class="text-muted">
                Using {{ ucfirst($searchMethod) }} search method
                {{ $search ? "for \"$search\"" : '' }}
            </small>
        @endif
    </form>
    
    <!-- Search Results -->
    <div class="row">
        @forelse($products as $product)
            <div class="col-md-3 mb-4">
                <div class="card">
                    <img src="{{ Storage::url($product->img_path) ?? asset('images/no-image.png') }}" 
                         class="card-img-top" alt="{{ $product->name }}">
                    <div class="card-body">
                        <h5>{{ $product->name }}</h5>
                        <p class="text-muted small">
                            {{ Str::limit($product->description, 80) }}
                        </p>
                        <h4 class="text-primary">${{ $product->sell_price }}</h4>
                    </div>
                </div>
            </div>
        @empty
            <div class="col-12">
                <div class="alert alert-info">
                    No products found for "{{ $search }}"
                </div>
            </div>
        @endforelse
    </div>
    
    <!-- Pagination -->
    {{ $products->links() }}
</div>
```

---

## Test URLs

```
# Default (Scout search)
GET /home?search=tent

# LIKE search
GET /home?search=tent&search_type=like

# Model scope search
GET /home?search=tent&search_type=model

# Scout search explicit
GET /home?search=tent&search_type=scout

# No search (show all)
GET /home

# With pagination
GET /home?search=tent&page=2
```

---

## Summary

**MP8 implements three search methods:**

**Method 1: LIKE Query (8pts)**
- ✅ Direct SQL LIKE operator
- ✅ Simple, no setup required
- ✅ Good for small datasets

**Method 2: Model Scope (10pts)**
- ✅ Eloquent scope encapsulation
- ✅ Reusable across application
- ✅ Cleaner code structure
- ✅ Easy to maintain

**Method 3: Laravel Scout (12pts)**
- ✅ Full-text search engine integration
- ✅ Best performance for large catalogs
- ✅ Advanced search ranking
- ✅ Automatic indexing
- ✅ Database or Algolia driver support

**Total Implementation: 30 Points**
