# MP7: Dashboard Charts & Analytics

## Overview
**Total Points: Built-in analytics system**
- Yearly Sales Chart
- Daily Sales Chart with Date Range
- Product Sales Pie Chart
- Customer Growth Analytics

---

## DashboardController

```php
<?php

namespace App\Http\Controllers;

use App\Models\User;
use App\Models\Order;
use App\Models\Product;
use App\Models\OrderItem;
use Illuminate\Http\Request;
use Illuminate\Support\Facades\DB;

class DashboardController extends Controller
{
    /**
     * Display dashboard with multiple charts
     * Aggregates data from orders and generates chart data
     */
    public function index(Request $request)
    {
        // Admin only - middleware handles this
        
        // Get date ranges from query parameters
        $dailyStartDate = $request->get('daily_start_date', now()->subDays(30)->toDateString());
        $dailyEndDate = $request->get('daily_end_date', now()->toDateString());
        $yearlyStartDate = $request->get('yearly_start_date', '2024-01-01');
        $yearlyEndDate = $request->get('yearly_end_date', now()->toDateString());

        // STATISTICS
        $stats = [
            'total_users' => User::count(),
            'total_products' => Product::count(),
            'total_categories' => \App\Models\Category::count(),
            'total_orders' => Order::count(),
            'total_revenue' => Order::where('status', '!=', 'cancelled')
                                    ->sum(DB::raw('total_amount + COALESCE(shipping_fee, 0)')),
            'pending_orders' => Order::where('status', 'pending')->count(),
            'completed_orders' => Order::where('status', 'completed')->count(),
        ];

        // CHART 1: Yearly Sales Revenue
        $yearlySalesData = $this->getYearlySalesData($yearlyStartDate, $yearlyEndDate);

        // CHART 2: Daily Sales with Date Range
        $dailySalesData = $this->getDailySalesData($dailyStartDate, $dailyEndDate);

        // CHART 3: Product Sales Distribution (Pie Chart)
        $productSalesData = $this->getProductSalesData();

        // CHART 4: Customer Growth
        $customerGrowthData = $this->getCustomerGrowthData();

        return view('dashboard.index', [
            'stats' => $stats,
            'yearlySalesData' => $yearlySalesData,
            'dailySalesData' => $dailySalesData,
            'productSalesData' => $productSalesData,
            'customerGrowthData' => $customerGrowthData,
            'dailyStartDate' => $dailyStartDate,
            'dailyEndDate' => $dailyEndDate,
            'yearlyStartDate' => $yearlyStartDate,
            'yearlyEndDate' => $yearlyEndDate,
        ]);
    }

    /**
     * Get yearly sales revenue by year
     * Groups orders by year and sums total revenue (including shipping)
     */
    private function getYearlySalesData($startYear, $endYear)
    {
        // Query: Group orders by year, sum revenue
        $data = Order::where('status', '!=', 'cancelled')
                    ->whereBetween('order_date', [$startYear . '-01-01', $endYear . '-12-31'])
                    ->selectRaw('YEAR(order_date) as year')
                    ->selectRaw('SUM(total_amount + COALESCE(shipping_fee, 0)) as revenue')
                    ->groupBy('year')
                    ->orderBy('year')
                    ->get();

        // Build chart data
        $labels = [];
        $revenues = [];
        
        foreach ($data as $row) {
            $labels[] = (string) $row->year;
            $revenues[] = round($row->revenue, 2);
        }

        return [
            'labels' => $labels,
            'data' => [
                [
                    'label' => 'Yearly Revenue',
                    'data' => $revenues,
                    'backgroundColor' => 'rgba(75, 192, 192, 0.2)',
                    'borderColor' => 'rgba(75, 192, 192, 1)',
                    'borderWidth' => 1,
                ]
            ]
        ];
    }

    /**
     * Get daily sales for custom date range
     * Shows revenue per day
     */
    private function getDailySalesData($startDate, $endDate)
    {
        // Query: Get daily revenue
        $data = Order::where('status', '!=', 'cancelled')
                    ->whereBetween('order_date', [$startDate, $endDate])
                    ->selectRaw('DATE(order_date) as date')
                    ->selectRaw('SUM(total_amount + COALESCE(shipping_fee, 0)) as revenue')
                    ->groupBy('date')
                    ->orderBy('date')
                    ->get();

        $labels = [];
        $revenues = [];
        
        foreach ($data as $row) {
            $labels[] = \Carbon\Carbon::parse($row->date)->format('M d');
            $revenues[] = round($row->revenue, 2);
        }

        return [
            'labels' => $labels,
            'data' => [
                [
                    'label' => 'Daily Revenue',
                    'data' => $revenues,
                    'backgroundColor' => 'rgba(54, 162, 235, 0.2)',
                    'borderColor' => 'rgba(54, 162, 235, 1)',
                    'borderWidth' => 2,
                    'fill' => true,
                ]
            ]
        ];
    }

    /**
     * Get top products by sales volume
     * Shows percentage of total sales contributed by each product
     */
    private function getProductSalesData()
    {
        // Query: Get top 10 products by sales volume
        $data = OrderItem::selectRaw('product_id')
                        ->selectRaw('SUM(quantity) as total_qty')
                        ->selectRaw('SUM(quantity * unit_price) as total_sales')
                        ->groupBy('product_id')
                        ->orderByDesc('total_qty')
                        ->limit(10)
                        ->with('product')
                        ->get();

        // Calculate total sales and percentages
        $totalSales = $data->sum('total_sales');
        $labels = [];
        $salesData = [];
        $colors = [
            '#FF6384', '#36A2EB', '#FFCE56', '#4BC0C0', '#9966FF',
            '#FF9F40', '#FF6384', '#C9CBCF', '#4BC0C0', '#FF6384',
        ];

        foreach ($data as $index => $item) {
            $percentage = ($item->total_sales / $totalSales) * 100;
            $labels[] = $item->product?->name ?? 'Unknown';
            $salesData[] = round($percentage, 2);
        }

        return [
            'labels' => $labels,
            'data' => [
                [
                    'data' => $salesData,
                    'backgroundColor' => array_slice($colors, 0, count($labels)),
                    'borderColor' => array_slice($colors, 0, count($labels)),
                    'borderWidth' => 1,
                ]
            ]
        ];
    }

    /**
     * Get customer growth over time
     * Shows number of new users registered per month
     */
    private function getCustomerGrowthData()
    {
        $data = User::selectRaw('DATE_TRUNC(created_at, MONTH) as month')
                   ->selectRaw('COUNT(*) as count')
                   ->groupBy('month')
                   ->orderBy('month')
                   ->get();

        $labels = [];
        $counts = [];

        foreach ($data as $row) {
            $labels[] = \Carbon\Carbon::parse($row->month)->format('M Y');
            $counts[] = $row->count;
        }

        return [
            'labels' => $labels,
            'data' => [
                [
                    'label' => 'New Customers',
                    'data' => $counts,
                    'backgroundColor' => 'rgba(153, 102, 255, 0.2)',
                    'borderColor' => 'rgba(153, 102, 255, 1)',
                    'borderWidth' => 1,
                ]
            ]
        ];
    }
}
```

---

## Blade Dashboard View

```blade
<!-- resources/views/dashboard/index.blade.php -->

<div class="container-fluid mt-5">
    <h1>Dashboard</h1>
    
    <!-- STATISTICS CARDS -->
    <div class="row mb-4">
        <div class="col-md-3">
            <div class="card bg-primary text-white">
                <div class="card-body">
                    <h5>Total Revenue</h5>
                    <h2>${{ number_format($stats['total_revenue'], 2) }}</h2>
                </div>
            </div>
        </div>
        <div class="col-md-3">
            <div class="card bg-success text-white">
                <div class="card-body">
                    <h5>Completed Orders</h5>
                    <h2>{{ $stats['completed_orders'] }}</h2>
                </div>
            </div>
        </div>
        <div class="col-md-3">
            <div class="card bg-warning text-white">
                <div class="card-body">
                    <h5>Pending Orders</h5>
                    <h2>{{ $stats['pending_orders'] }}</h2>
                </div>
            </div>
        </div>
        <div class="col-md-3">
            <div class="card bg-info text-white">
                <div class="card-body">
                    <h5>Total Customers</h5>
                    <h2>{{ $stats['total_users'] }}</h2>
                </div>
            </div>
        </div>
    </div>

    <!-- CHART 1: Yearly Sales -->
    <div class="row mb-4">
        <div class="col-md-6">
            <div class="card">
                <div class="card-header">
                    <h5>Yearly Sales Revenue</h5>
                </div>
                <div class="card-body">
                    <canvas id="yearlySalesChart"></canvas>
                </div>
            </div>
        </div>
        
        <!-- CHART 2: Daily Sales -->
        <div class="col-md-6">
            <div class="card">
                <div class="card-header">
                    <h5>Daily Sales ({{ $dailyStartDate }} to {{ $dailyEndDate }})</h5>
                </div>
                <div class="card-body">
                    <canvas id="dailySalesChart"></canvas>
                </div>
            </div>
        </div>
    </div>

    <!-- CHART 3: Product Sales Distribution -->
    <div class="row mb-4">
        <div class="col-md-6">
            <div class="card">
                <div class="card-header">
                    <h5>Product Sales Distribution</h5>
                </div>
                <div class="card-body">
                    <canvas id="productSalesChart"></canvas>
                </div>
            </div>
        </div>
        
        <!-- CHART 4: Customer Growth -->
        <div class="col-md-6">
            <div class="card">
                <div class="card-header">
                    <h5>Customer Growth</h5>
                </div>
                <div class="card-body">
                    <canvas id="customerGrowthChart"></canvas>
                </div>
            </div>
        </div>
    </div>
</div>

<!-- Chart.js Library -->
<script src="https://cdn.jsdelivr.net/npm/chart.js@2.9.3/dist/Chart.min.js"></script>

<script>
    // Chart 1: Yearly Sales (Bar Chart)
    const yearlySalesCtx = document.getElementById('yearlySalesChart').getContext('2d');
    new Chart(yearlySalesCtx, {
        type: 'bar',
        data: {
            labels: {!! json_encode($yearlySalesData['labels']) !!},
            datasets: {!! json_encode($yearlySalesData['data']) !!}
        },
        options: {
            responsive: true,
            scales: {
                yAxes: [{
                    ticks: {
                        beginAtZero: true,
                        callback: function(value) {
                            return '$' + value.toLocaleString();
                        }
                    }
                }]
            }
        }
    });

    // Chart 2: Daily Sales (Line Chart)
    const dailySalesCtx = document.getElementById('dailySalesChart').getContext('2d');
    new Chart(dailySalesCtx, {
        type: 'line',
        data: {
            labels: {!! json_encode($dailySalesData['labels']) !!},
            datasets: {!! json_encode($dailySalesData['data']) !!}
        },
        options: {
            responsive: true,
            scales: {
                yAxes: [{
                    ticks: {
                        beginAtZero: true,
                        callback: function(value) {
                            return '$' + value.toLocaleString();
                        }
                    }
                }]
            }
        }
    });

    // Chart 3: Product Sales (Pie Chart)
    const productSalesCtx = document.getElementById('productSalesChart').getContext('2d');
    new Chart(productSalesCtx, {
        type: 'pie',
        data: {
            labels: {!! json_encode($productSalesData['labels']) !!},
            datasets: {!! json_encode($productSalesData['data']) !!}
        },
        options: {
            responsive: true,
            legend: {
                position: 'right',
            }
        }
    });

    // Chart 4: Customer Growth (Line Chart)
    const customerGrowthCtx = document.getElementById('customerGrowthChart').getContext('2d');
    new Chart(customerGrowthCtx, {
        type: 'line',
        data: {
            labels: {!! json_encode($customerGrowthData['labels']) !!},
            datasets: {!! json_encode($customerGrowthData['data']) !!}
        },
        options: {
            responsive: true,
            scales: {
                yAxes: [{
                    ticks: {
                        beginAtZero: true,
                    }
                }]
            }
        }
    });
</script>
```

---

## Summary

**MP7 implements comprehensive analytics with:**
- ✅ Yearly sales revenue bar chart
- ✅ Daily sales line chart with custom date range
- ✅ Product sales distribution pie chart
- ✅ Customer growth tracking
- ✅ Summary statistics cards
- ✅ Chart.js v2.9.3 integration
- ✅ Responsive design
- ✅ Real-time data aggregation
- ✅ Customizable date ranges
