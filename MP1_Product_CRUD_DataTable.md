# MP1: Product/Service CRUD with DataTable

## Overview
**Total Points: 53pts**
- Stage 1: DataTable CRUD (8pts) ✅
- Stage 2: Soft Deletes & Restore (10pts) ✅
- Stage 3: Single & Multiple Photo Upload (15pts) ✅
- Stage 4: Excel Import (20pts) ✅

---

## Stage 1: Product CRUD with DataTable (8pts)

### Key Files
- **Controller**: `app/Http/Controllers/ProductController.php`
- **Model**: `app/Models/Product.php`
- **View**: `resources/views/products/index.blade.php`
- **Route**: `routes/web.php` - `/products` & `/products/datatable`

### ProductController - index() Method

```php
/**
 * Display a listing of the resource.
 * This method loads the products index view with all categories
 */
public function index()
{
    // Fetch all categories for filter dropdowns
    $categories = Category::all();
    
    // Return the products listing view
    return view('products.index', ['categories' => $categories]);
}
```

**Explanation:**
- `Category::all()` - Retrieves all categories from the database to populate filter options
- `view()` - Returns a Blade template with the data array passed
- The actual DataTable rendering happens on the frontend via JavaScript

### ProductController - datatable() Method (API Endpoint)

```php
/**
 * Get products data for DataTables (AJAX API endpoint)
 * This method is called by jquery-datatables AJAX requests
 * Returns server-side processed data
 */
public function datatable(Request $request)
{
    try {
        // Log access for debugging
        \Log::info('ProductController::datatable called - User: ' . (auth()->check() ? auth()->user()->id : 'NOT_AUTHENTICATED'));
        
        // Authorization Check 1: User must be authenticated
        if (!auth()->check()) {
            \Log::warning('Unauthorized datatable access - user not authenticated');
            return response()->json(['error' => 'Unauthorized'], 401);
        }
        
        // Authorization Check 2: User must be admin
        if (auth()->user()->role !== 'admin') {
            \Log::warning('Unauthorized datatable access - user not admin');
            return response()->json(['error' => 'Forbidden'], 403);
        }
        
        // Build base query with all relationships
        $query = Product::select('products.*')
            ->selectRaw('product_id as id')  // DataTables requires 'id' column
            ->with('category', 'stock', 'images')  // Eager load relationships
            ->withTrashed()  // Include soft-deleted products (Show Deleted toggle)
            ->orderBy('products.product_id', 'desc');  // Newest first

        // Use Yajra DataTables to process the query
        return DataTables::of($query)
            ->setRowId('product_id')  // Set unique row ID for actions
            
            // Transform image column into HTML img tag
            ->editColumn('img_path', function ($product) {
                $mainImage = $product->img_path 
                    ? Storage::url($product->img_path)  // Get storage URL
                    : asset('images/no-image.png');     // Fallback to placeholder
                return '<img src="' . $mainImage . '" alt="' . $product->name . '" 
                        width="50" height="50" class="img-thumbnail">';
            })
            
            // Add category name column
            ->addColumn('category_name', function ($product) {
                return $product->category ? $product->category->name : 'Uncategorized';
            })
            
            // Add stock quantity column with color-coded badge
            ->addColumn('stock_quantity', function ($product) {
                $stock = $product->stock->sum('quantity');  // Sum all stock quantities
                return $stock > 0 
                    ? '<span class="badge bg-success">' . $stock . '</span>'     // Green for in stock
                    : '<span class="badge bg-danger">Out of Stock</span>';        // Red for out of stock
            })
            
            // Add photo count badge
            ->addColumn('photo_count', function ($product) {
                return '<span class="badge bg-info">' . $product->images->count() . '</span>';
            })
            
            // Add status badge (Active/Deleted)
            ->addColumn('status', function ($product) {
                return $product->deleted_at 
                    ? '<span class="badge bg-secondary">Deleted</span>'    // Gray for deleted
                    : '<span class="badge bg-success">Active</span>';      // Green for active
            })
            
            // Add action buttons column
            ->addColumn('actions', function ($product) {
                return $this->renderActions($product);
            })
            
            // Client-side search filter on name column
            ->filterColumn('name', function ($query, $keyword) {
                $query->where('name', 'like', "%{$keyword}%")
                      ->orWhere('description', 'like', "%{$keyword}%");
            })
            
            // Render raw HTML in columns (not escaped)
            ->rawColumns(['img_path', 'stock_quantity', 'photo_count', 'status', 'actions'])
            
            // Convert to DataTables format
            ->make(true);
            
    } catch (\Exception $e) {
        // Log error and return JSON error response
        \Log::error('DataTable error: ' . $e->getMessage(), ['exception' => $e]);
        return response()->json(['error' => 'Server error: ' . $e->getMessage()], 500);
    }
}
```

**Key Explanations:**

1. **Authentication & Authorization**: Double check - user must be authenticated AND be an admin
2. **Eager Loading**: `with('category', 'stock', 'images')` prevents N+1 query problem
3. **withTrashed()**: Includes soft-deleted products so they appear in DataTable
4. **DataTables::of()**: Yajra package wraps query for AJAX processing
5. **editColumn()**: Transforms existing column (img_path) into HTML
6. **addColumn()**: Creates new computed columns (category_name, stock_quantity, etc.)
7. **filterColumn()**: Custom filtering for name/description search
8. **rawColumns()**: Prevents HTML escaping for columns with HTML content

### renderActions() Helper Method

```php
/**
 * Render action buttons for each product row
 * Returns HTML with View, Edit, Restore (if deleted), Delete buttons
 */
private function renderActions($product)
{
    // Initialize actions button group
    $actions = '<div class="btn-group btn-group-sm" role="group" aria-label="Product actions">';
    
    // View button - always visible
    $actions .= '<a href="' . route('products.show', $product) . '" 
                    class="btn btn-info" title="View">
                    <i class="fas fa-eye"></i>
                </a>';
    
    // Admin-only actions
    if (auth()->check() && auth()->user()->role === 'admin') {
        // Edit button
        $actions .= '<a href="' . route('products.edit', $product) . '" 
                        class="btn btn-warning" title="Edit">
                        <i class="fas fa-edit"></i>
                    </a>';
        
        // Restore button (only for soft-deleted products)
        if ($product->deleted_at) {
            $actions .= '<a href="' . route('products.restore', $product->product_id) . '" 
                            class="btn btn-success" title="Restore">
                            <i class="fas fa-undo"></i>
                        </a>';
        }
        
        // Delete button with confirmation
        $actions .= '<button type="button" class="btn btn-danger" title="Delete" 
                        onclick="if(confirm(\'Are you sure?\')) { 
                            document.getElementById(\'delete-product-' . $product->product_id . '\').submit(); 
                        }">
                        <i class="fas fa-trash"></i>
                    </button>';
    }
    
    $actions .= '</div>';

    // Hidden form for DELETE request (only for admins)
    if (auth()->check() && auth()->user()->role === 'admin') {
        $actions .= '<form id="delete-product-' . $product->product_id . '" 
                        action="' . route('products.destroy', $product) . '" 
                        method="POST" style="display:none;">
                        <input type="hidden" name="_method" value="DELETE">
                        <input type="hidden" name="_token" value="' . csrf_token() . '">
                    </form>';
    }

    return $actions;
}
```

**Explanation:**
- `route()` - Generates URL for RESTful routes
- `csrf_token()` - Security token required for form submissions
- Hidden form uses `_method` hidden field because HTML forms only support GET/POST natively, but Laravel converts this to DELETE/PUT

### Product Model

```php
<?php

namespace App\Models;

use Illuminate\Database\Eloquent\Model;
use Illuminate\Database\Eloquent\SoftDeletes;  // For soft deletes
use Laravel\Scout\Searchable;  // For full-text search
use Illuminate\Database\Eloquent\Relations\BelongsTo;
use Illuminate\Database\Eloquent\Relations\HasMany;

class Product extends Model
{
    use SoftDeletes, Searchable;  // Include soft delete & search traits
    
    // Override default table name (Laravel expects 'products' by default - so this is redundant but explicit)
    protected $table = 'products';
    
    // Override default primary key (Laravel expects 'id' by default, we use 'product_id')
    protected $primaryKey = 'product_id';
    
    // Enable timestamps (created_at, updated_at automatically managed)
    public $timestamps = true;
    
    // Mass assignable attributes
    protected $fillable = [
        'name',
        'description',
        'cost_price',
        'sell_price',
        'category_id',
        'img_path',
    ];
    
    // Cast attributes to specific types
    protected $casts = [
        'cost_price' => 'decimal:2',   // Always 2 decimal places
        'sell_price' => 'decimal:2',
        'category_id' => 'int',
    ];

    /**
     * Relationship: Product belongs to one Category
     */
    public function category(): BelongsTo
    {
        return $this->belongsTo(Category::class, 'category_id', 'category_id');
    }

    /**
     * Relationship: Product has many ProductImages (gallery)
     */
    public function images(): HasMany
    {
        return $this->hasMany(ProductImage::class, 'product_id', 'product_id');
    }

    /**
     * Relationship: Product has many Reviews
     */
    public function reviews(): HasMany
    {
        return $this->hasMany(Review::class, 'product_id', 'product_id');
    }

    /**
     * Relationship: Product has many CartItems
     */
    public function cartItems(): HasMany
    {
        return $this->hasMany(CartItem::class, 'product_id', 'product_id');
    }

    /**
     * Relationship: Product has many OrderItems (when ordered)
     */
    public function orderItems(): HasMany
    {
        return $this->hasMany(OrderItem::class, 'product_id', 'product_id');
    }

    /**
     * Relationship: Product has many Stock records
     */
    public function stock(): HasMany
    {
        return $this->hasMany(Stock::class, 'product_id', 'product_id');
    }

    /**
     * Query Scope: Search products by keyword
     * Searches in name and description fields using LIKE
     * 
     * Usage: Product::search($searchTerm)->get()
     */
    public function scopeSearch(?string $term)
    {
        $term = trim((string) $term);  // Remove whitespace

        // If search term is empty, return all
        if ($term === '') {
            return $this;
        }

        // Search in name OR description (case-insensitive)
        return $this->where('name', 'LIKE', "%{$term}%")
                    ->orWhere('description', 'LIKE', "%{$term}%");
    }
}
```

**Key Points:**
- **SoftDeletes trait**: Marks deleted records with `deleted_at` timestamp instead of removing them
- **Searchable trait**: Enables Laravel Scout full-text search integration
- **$fillable**: Security measure - only these attributes can be mass-assigned
- **$casts**: Automatic type conversion when retrieving/storing data
- **Relationships**: Define connections to other models

---

## Stage 2: Soft Deletes & Restore (10pts)

### What is Soft Delete?
Instead of permanently removing a record from the database, we mark it with a `deleted_at` timestamp. This allows us to:
1. Recover accidentally deleted data
2. Keep historical records
3. Maintain referential integrity

### Database Migration

```php
<?php

use Illuminate\Database\Migrations\Migration;
use Illuminate\Database\Schema\Blueprint;
use Illuminate\Support\Facades\Schema;

return new class extends Migration
{
    /**
     * Run the migrations.
     * Adds deleted_at column to products table
     */
    public function up(): void
    {
        Schema::table('products', function (Blueprint $table) {
            // Nullable timestamp for soft delete
            // If NULL = not deleted
            // If has timestamp = deleted at that time
            $table->softDeletes();  // Shorthand for: $table->timestamp('deleted_at')->nullable();
        });
    }

    /**
     * Reverse the migrations.
     */
    public function down(): void
    {
        Schema::table('products', function (Blueprint $table) {
            $table->dropSoftDeletes();
        });
    }
};
```

### Product Model - Soft Delete Implementation

```php
use Illuminate\Database\Eloquent\SoftDeletes;

class Product extends Model
{
    use SoftDeletes;  // Enable soft delete functionality
    
    // Automatically exclude soft-deleted records from queries
    // Product::all() will NOT include deleted products
    // To include them: Product::withTrashed()->get()
}
```

### ProductController - destroy() Method (Soft Delete)

```php
/**
 * Soft delete a product (mark as deleted, don't remove from DB)
 * 
 * @param Product $product
 * @return \Illuminate\Http\RedirectResponse
 */
public function destroy(Product $product)
{
    // Check admin authorization
    if (auth()->user()->role !== 'admin') {
        return redirect()->back()->with('error', 'Unauthorized');
    }

    // Soft delete: sets deleted_at timestamp
    $product->delete();

    return redirect()->route('products.index')
                    ->with('success', 'Product deleted successfully');
}
```

**How it works:**
1. `$product->delete()` - Calls Eloquent's delete method
2. Since `SoftDeletes` trait is used, it sets `deleted_at = now()`
3. Product is NOT physically removed from database
4. Future queries automatically exclude it (except those using `withTrashed()`)

### ProductController - restore() Method

```php
/**
 * Restore a soft-deleted product
 * 
 * @param int $id - Product ID
 * @return \Illuminate\Http\RedirectResponse
 */
public function restore($id)
{
    // Find including trashed records
    $product = Product::withTrashed()->findOrFail($id);
    
    // Check admin authorization
    if (auth()->user()->role !== 'admin') {
        return redirect()->back()->with('error', 'Unauthorized');
    }

    // Restore: sets deleted_at to NULL
    $product->restore();

    return redirect()->route('products.index')
                    ->with('success', 'Product restored successfully');
}
```

**How it works:**
1. `Product::withTrashed()` - Include soft-deleted records in query
2. `findOrFail($id)` - Find by ID or throw 404 exception
3. `$product->restore()` - Sets `deleted_at = NULL` and saves
4. Product is now visible in normal queries again

### Routes

```php
// In routes/web.php
Route::middleware(['auth', 'admin'])->group(function () {
    // Standard CRUD routes
    Route::resource('products', ProductController::class);
    
    // Additional routes
    Route::get('/products/{product}/restore', [ProductController::class, 'restore'])
        ->name('products.restore');
    Route::post('/products/import', [ProductController::class, 'import'])
        ->name('products.import');
    Route::get('/products/import/form', [ProductController::class, 'importForm'])
        ->name('products.importForm');
    Route::delete('/products-image/{imageId}', [ProductController::class, 'deleteImage'])
        ->name('products.deleteImage');
});

// REST Resource routes automatically create:
// GET    /products              → index()
// GET    /products/create       → create()
// POST   /products              → store()
// GET    /products/{product}    → show()
// GET    /products/{product}/edit → edit()
// PUT    /products/{product}    → update()
// DELETE /products/{product}    → destroy()
```

---

## Stage 3: Single & Multiple Photo Upload (15pts)

### Features
- **Main Image**: Single product photo
- **Gallery**: Multiple additional photos
- **Preview**: Images preview before upload
- **AJAX Delete**: Remove photos without page reload
- **Validation**: File type, size, and dimension checks

### ProductImage Model

```php
<?php

namespace App\Models;

use Illuminate\Database\Eloquent\Model;
use Illuminate\Database\Eloquent\Relations\BelongsTo;

class ProductImage extends Model
{
    protected $table = 'product_images';
    protected $primaryKey = 'image_id';  // Custom primary key
    public $timestamps = true;
    
    protected $fillable = [
        'product_id',
        'img_path',
    ];

    /**
     * Each ProductImage belongs to one Product
     */
    public function product(): BelongsTo
    {
        return $this->belongsTo(Product::class, 'product_id', 'product_id');
    }
}
```

### ProductController - store() Method (Photo Upload)

```php
/**
 * Store a newly created product with photo upload
 */
public function store(Request $request)
{
    // Validate all inputs
    $validated = $request->validate([
        // Product name validation
        'name' => 'required|string|min:3|max:255|regex:/^[a-zA-Z0-9\s\-&\/.,()]+$/',
        
        // Description validation
        'description' => 'required|string|min:10|max:5000',
        
        // Price validation
        'cost_price' => 'required|numeric|min:0|max:999999.99',
        'sell_price' => 'required|numeric|min:0|max:999999.99|gte:cost_price',
        
        // Category optional
        'category_id' => 'nullable|exists:categories,category_id',
        
        // Stock quantity
        'stocks' => 'nullable|integer|min:0',
        
        // Main product image
        'img_path' => 'nullable|image|mimes:jpeg,png,jpg,gif|max:2048|dimensions:min_width=100,min_height=100',
        
        // Gallery images array
        'images.*' => 'nullable|image|mimes:jpeg,png,jpg,gif|max:2048|dimensions:min_width=100,min_height=100',
    ], [
        // Custom error messages
        'name.required' => 'Product name is required.',
        'name.regex' => 'Product name contains invalid characters.',
        'sell_price.gte' => 'Selling price must be >= cost price.',
        'img_path.image' => 'Main image must be a valid image file.',
        'img_path.max' => 'Main image must be under 2MB.',
        'images.*.image' => 'Gallery images must be valid image files.',
    ]);

    // Create product record
    $product = Product::create($validated);

    // Handle main image upload
    if ($request->hasFile('img_path')) {
        $file = $request->file('img_path');
        // Store in storage/app/public/products/
        // Put method returns the path relative to storage/app/public
        $path = $file->store('products', 'public');
        
        // Update product with main image path
        $product->update(['img_path' => $path]);
    }

    // Handle multiple gallery images
    if ($request->hasFile('images')) {
        foreach ($request->file('images') as $image) {
            // Store each image
            $path = $image->store('products', 'public');
            
            // Create ProductImage record
            ProductImage::create([
                'product_id' => $product->product_id,
                'img_path' => $path,
            ]);
        }
    }

    // Create stock record
    if ($validated['stocks'] ?? 0) {
        Stock::create([
            'product_id' => $product->product_id,
            'quantity' => $validated['stocks'],
        ]);
    }

    return redirect()->route('products.index')
                    ->with('success', 'Product created successfully with ' . ($request->file('images') ? count($request->file('images')) : 0) . ' gallery images.');
}
```

**Validation Rules Explained:**

| Rule | Meaning |
|------|---------|
| `required` | Field must have a value |
| `string` | Must be text |
| `min:3` / `max:255` | Length constraints |
| `numeric` | Must be a number |
| `gte:cost_price` | Must be >= cost_price (greater than or equal) |
| `image` | Must be valid image MIME type |
| `mimes:jpeg,png,jpg,gif` | Allowed image formats |
| `max:2048` | Maximum file size 2048 KB (2MB) |
| `dimensions:min_width=100,min_height=100` | Image must be at least 100x100px |
| `nullable` | Field is optional |
| `.*` | Array items - each element must match rule |

### ProductController - update() Method (Photo Management)

```php
/**
 * Update product including new gallery photos
 */
public function update(Request $request, Product $product)
{
    // Validate inputs
    $validated = $request->validate([
        'name' => 'required|string|min:3|max:255',
        'description' => 'required|string|min:10|max:5000',
        'cost_price' => 'required|numeric|min:0|max:999999.99',
        'sell_price' => 'required|numeric|min:0|max:999999.99|gte:cost_price',
        'category_id' => 'nullable|exists:categories,category_id',
        'img_path' => 'nullable|image|mimes:jpeg,png,jpg,gif|max:2048|dimensions:min_width=100,min_height=100',
        'images.*' => 'nullable|image|mimes:jpeg,png,jpg,gif|max:2048|dimensions:min_width=100,min_height=100',
    ]);

    // Update main product fields
    $product->update($validated);

    // Replace main image if new one uploaded
    if ($request->hasFile('img_path')) {
        // Delete old image
        if ($product->img_path) {
            Storage::disk('public')->delete($product->img_path);
        }
        
        // Store new image
        $path = $request->file('img_path')->store('products', 'public');
        $product->update(['img_path' => $path]);
    }

    // Add new gallery images
    if ($request->hasFile('images')) {
        foreach ($request->file('images') as $image) {
            $path = $image->store('products', 'public');
            ProductImage::create([
                'product_id' => $product->product_id,
                'img_path' => $path,
            ]);
        }
    }

    return redirect()->route('products.index')
                    ->with('success', 'Product updated successfully.');
}
```

### ProductController - deleteImage() Method (AJAX Photo Delete)

```php
/**
 * Delete a single gallery image via AJAX
 * 
 * @param int $imageId - ProductImage ID
 * @return \Illuminate\Http\JsonResponse
 */
public function deleteImage($imageId)
{
    // Find image or return 404
    $image = ProductImage::findOrFail($imageId);
    
    // Check authorization
    if (auth()->user()->role !== 'admin') {
        return response()->json(['error' => 'Unauthorized'], 403);
    }

    // Delete file from storage
    if ($image->img_path) {
        Storage::disk('public')->delete($image->img_path);
    }

    // Delete database record
    $image->delete();

    // Return JSON response for AJAX
    return response()->json([
        'success' => true,
        'message' => 'Image deleted successfully'
    ]);
}
```

### File Storage Structure

```
storage/
  app/
    public/
      products/
        2026/02/22/
          uuid-filename.jpg
          uuid-filename2.jpg
          ...
```

Laravel automatically organized files by date directories for better performance.

### Blade View - Create Form with Preview

```blade
<form method="POST" action="{{ route('products.store') }}" enctype="multipart/form-data">
    @csrf
    
    <!-- Product Info -->
    <div class="mb-3">
        <label for="name" class="form-label">Product Name</label>
        <input type="text" class="form-control @error('name') is-invalid @enderror" 
               id="name" name="name" value="{{ old('name') }}" required>
        @error('name')
            <div class="invalid-feedback">{{ $message }}</div>
        @enderror
    </div>

    <!-- Main Image Upload -->
    <div class="mb-3">
        <label for="img_path" class="form-label">Main Product Image</label>
        <input type="file" class="form-control @error('img_path') is-invalid @enderror" 
               id="img_path" name="img_path" accept="image/*" onchange="previewMainImage(this)">
        <img id="mainImagePreview" src="" alt="Preview" style="max-width: 200px; margin-top: 10px; display: none;">
        @error('img_path')
            <div class="invalid-feedback d-block">{{ $message }}</div>
        @enderror
    </div>

    <!-- Gallery Images Upload -->
    <div class="mb-3">
        <label for="images" class="form-label">Gallery Images (Multiple)</label>
        <input type="file" class="form-control @error('images') is-invalid @enderror" 
               id="images" name="images[]" multiple accept="image/*" onchange="previewGallery(this)">
        <div id="galleryPreview" class="mt-3"></div>
        @error('images')
            <div class="invalid-feedback d-block">{{ $message }}</div>
        @enderror
    </div>

    <button type="submit" class="btn btn-primary">Create Product</button>
</form>

<script>
// Preview main image
function previewMainImage(input) {
    const preview = document.getElementById('mainImagePreview');
    if (input.files && input.files[0]) {
        const reader = new FileReader();
        reader.onload = function(e) {
            preview.src = e.target.result;
            preview.style.display = 'block';
        }
        reader.readAsDataURL(input.files[0]);
    }
}

// Preview gallery images
function previewGallery(input) {
    const gallery = document.getElementById('galleryPreview');
    gallery.innerHTML = '';  // Clear previous
    
    if (input.files) {
        for (let i = 0; i < input.files.length; i++) {
            const reader = new FileReader();
            reader.onload = function(e) {
                const img = document.createElement('img');
                img.src = e.target.result;
                img.style.maxWidth = '100px';
                img.style.margin = '5px';
                gallery.appendChild(img);
            }
            reader.readAsDataURL(input.files[i]);
        }
    }
}
</script>
```

---

## Stage 4: Excel Import (20pts)

### ProductsImport Class

```php
<?php

namespace App\Imports;

use App\Models\Product;
use App\Models\Category;
use Maatwebsite\Excel\Concerns\ToModel;
use Maatwebsite\Excel\Concerns\WithHeadingRow;
use Maatwebsite\Excel\Concerns\WithValidation;

class ProductsImport implements ToModel, WithHeadingRow, WithValidation
{
    /**
     * Map each row to a Product model
     * WithHeadingRow automatically skips first row and uses as column names
     */
    public function model(array $row)
    {
        // Get or create category if name provided
        $category = null;
        if (!empty($row['category_name'])) {
            $category = Category::firstOrCreate(
                ['name' => $row['category_name']],
                ['description' => 'Imported from Excel']
            );
        }

        // Create product record
        return new Product([
            'name'        => $row['name'],
            'description' => $row['description'] ?? 'No description provided',
            'cost_price'  => (float) $row['cost_price'],
            'sell_price'  => (float) $row['sell_price'],
            'category_id' => $category ? $category->category_id : null,
        ]);
    }

    /**
     * Define validation rules for imported data
     */
    public function rules(): array
    {
        return [
            'name' => 'required|string|max:255',
            'description' => 'nullable|string',
            'cost_price' => 'required|numeric|min:0',
            'sell_price' => 'required|numeric|min:0',
            'category_name' => 'nullable|string|max:255',
        ];
    }
}
```

### ProductController - importForm() Method

```php
/**
 * Show Excel import form
 */
public function importForm()
{
    return view('products.import');
}
```

### ProductController - import() Method

```php
/**
 * Handle Excel file import
 */
public function import(Request $request)
{
    // Validate file upload
    $validated = $request->validate([
        'file' => 'required|file|mimes:xlsx,xls,csv|max:5120',  // 5MB max
    ], [
        'file.required' => 'Please select a file to import.',
        'file.mimes' => 'File must be Excel (.xlsx, .xls) or CSV format.',
        'file.max' => 'File size must be under 5MB.',
    ]);

    try {
        // Import file using Maatwebsite\Excel
        Excel::import(new ProductsImport(), $validated['file']);
        
        return redirect()->route('products.index')
                        ->with('success', 'Products imported successfully!');
    } catch (\Maatwebsite\Excel\Validators\ValidationException $e) {
        // Handle validation errors
        $failures = $e->failures();
        $errors = [];
        
        foreach ($failures as $failure) {
            $errors[] = "Row {$failure->row()}: " . implode(', ', $failure->errors());
        }
        
        return back()->with('errors', $errors);
    } catch (\Exception $e) {
        // Handle other errors
        return back()->with('error', 'Import failed: ' . $e->getMessage());
    }
}
```

### Excel File Format

```
Required Columns (headers in first row):
- name (required)
- description (optional)
- cost_price (required, decimal)
- sell_price (required, decimal)
- category_name (optional)

Example:
| name           | description              | cost_price | sell_price | category_name |
|----------------|-------------------------|------------|-----------|---------------|
| Tent 2-Person  | Lightweight camping tent | 45.50      | 79.99     | Camping       |
| Sleeping Bag   | Warm weather bag        | 30.00      | 59.99     | Camping       |
| Water Bottle   | Stainless steel 1L      | 5.00       | 12.99     | Accessories   |
```

### Blade View - Import Form

```blade
<form method="POST" action="{{ route('products.import') }}" enctype="multipart/form-data">
    @csrf
    
    <div class="card">
        <div class="card-header">
            <h4>Import Products from Excel</h4>
        </div>
        
        <div class="card-body">
            <div class="mb-3">
                <label for="file" class="form-label">Select Excel File</label>
                <input type="file" class="form-control @error('file') is-invalid @enderror" 
                       id="file" name="file" accept=".xlsx,.xls,.csv" required>
                @error('file')
                    <div class="invalid-feedback d-block">{{ $message }}</div>
                @enderror
            </div>
            
            <h5>Required Columns:</h5>
            <ul>
                <li><strong>name</strong> - Product name (required)</li>
                <li><strong>description</strong> - Description (optional)</li>
                <li><strong>cost_price</strong> - Cost price as decimal (required)</li>
                <li><strong>sell_price</strong> - Selling price as decimal (required)</li>
                <li><strong>category_name</strong> - Category name (optional)</li>
            </ul>
            
            <a href="{{ route('products.template') }}" class="btn btn-secondary">
                Download Template
            </a>
        </div>
        
        <div class="card-footer">
            <button type="submit" class="btn btn-primary">Import Products</button>
            <a href="{{ route('products.index') }}" class="btn btn-secondary">Cancel</a>
        </div>
    </div>
</form>
```

---

## Database Schema

```sql
CREATE TABLE products (
    product_id BIGINT UNSIGNED PRIMARY KEY AUTO_INCREMENT,
    name VARCHAR(255) NOT NULL,
    description TEXT NOT NULL,
    cost_price DECIMAL(10,2) NOT NULL,
    sell_price DECIMAL(10,2) NOT NULL,
    category_id BIGINT UNSIGNED NULLABLE,
    img_path VARCHAR(255) NULLABLE,
    deleted_at TIMESTAMP NULL,  -- For soft deletes
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    
    FOREIGN KEY (category_id) REFERENCES categories(category_id) ON DELETE SET NULL
);

CREATE TABLE product_images (
    image_id BIGINT UNSIGNED PRIMARY KEY AUTO_INCREMENT,
    product_id BIGINT UNSIGNED NOT NULL,
    img_path VARCHAR(255) NOT NULL,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    
    FOREIGN KEY (product_id) REFERENCES products(product_id) ON DELETE CASCADE
);
```

---

## Summary

**MP1 implements a complete product management system with:**
- ✅ DataTable UI for easy product viewing with pagination & search
- ✅ Soft delete functionality for data recovery
- ✅ Single main image + multiple gallery images
- ✅ Excel import for bulk product uploads
- ✅ Comprehensive validation
- ✅ Admin-only access control
- ✅ Responsive design with preview functionality

**Total Implementation: 53 Points**
