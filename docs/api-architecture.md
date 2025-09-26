# API Architecture & Endpoints for Enhanced Warehouse Module

## Overview
This document outlines the complete REST API architecture for the enhanced warehouse module, providing comprehensive endpoints for product management, inventory operations, multi-warehouse support, IMPA/SKU/Barcode systems, and advanced tracking features.

## API Architecture Principles

### 1. RESTful Design
- Resource-based URLs
- HTTP verbs for actions (GET, POST, PUT, DELETE)
- Consistent response format
- Proper HTTP status codes

### 2. Multi-Tenant Support
- Company-scoped resources
- Branch-level permissions
- Warehouse-specific operations
- User-based access control

### 3. Real-time Capabilities
- WebSocket integration for live updates
- Event broadcasting for inventory changes
- Push notifications for critical alerts
- Background job processing

## Core API Endpoints

### 1. Warehouse Management

#### Warehouse Operations
```php
<?php

namespace App\Modules\Warehouses\Presentation\Controllers\Api;

use App\Http\Controllers\Controller;
use App\Modules\Warehouses\Application\Services\WarehouseService;
use Illuminate\Http\Request;
use Illuminate\Http\JsonResponse;

class WarehouseController extends Controller
{
    protected WarehouseService $warehouseService;

    public function __construct(WarehouseService $warehouseService)
    {
        $this->warehouseService = $warehouseService;
    }

    /**
     * GET /api/warehouses
     * List all warehouses for the authenticated company
     */
    public function index(Request $request): JsonResponse
    {
        $warehouses = $this->warehouseService->getWarehousesByCompany(
            auth()->user()->company_id,
            $request->only(['type', 'status', 'branch_id', 'search'])
        );

        return response()->json([
            'success' => true,
            'data' => $warehouses,
            'meta' => [
                'total' => $warehouses->total(),
                'per_page' => $warehouses->perPage(),
                'current_page' => $warehouses->currentPage(),
            ]
        ]);
    }

    /**
     * POST /api/warehouses
     * Create a new warehouse
     */
    public function store(StoreWarehouseRequest $request): JsonResponse
    {
        $warehouse = $this->warehouseService->createWarehouse(
            auth()->user()->company_id,
            $request->validated()
        );

        return response()->json([
            'success' => true,
            'message' => 'Warehouse created successfully',
            'data' => $warehouse
        ], 201);
    }

    /**
     * GET /api/warehouses/{warehouse}
     * Get warehouse details with zones and locations
     */
    public function show(int $warehouseId): JsonResponse
    {
        $warehouse = $this->warehouseService->getWarehouseWithDetails($warehouseId);

        return response()->json([
            'success' => true,
            'data' => $warehouse
        ]);
    }

    /**
     * PUT /api/warehouses/{warehouse}
     * Update warehouse information
     */
    public function update(UpdateWarehouseRequest $request, int $warehouseId): JsonResponse
    {
        $warehouse = $this->warehouseService->updateWarehouse($warehouseId, $request->validated());

        return response()->json([
            'success' => true,
            'message' => 'Warehouse updated successfully',
            'data' => $warehouse
        ]);
    }

    /**
     * DELETE /api/warehouses/{warehouse}
     * Soft delete a warehouse
     */
    public function destroy(int $warehouseId): JsonResponse
    {
        $this->warehouseService->deleteWarehouse($warehouseId);

        return response()->json([
            'success' => true,
            'message' => 'Warehouse deleted successfully'
        ]);
    }

    /**
     * GET /api/warehouses/{warehouse}/zones
     * Get warehouse zones
     */
    public function getZones(int $warehouseId): JsonResponse
    {
        $zones = $this->warehouseService->getWarehouseZones($warehouseId);

        return response()->json([
            'success' => true,
            'data' => $zones
        ]);
    }

    /**
     * GET /api/warehouses/{warehouse}/locations
     * Get warehouse locations
     */
    public function getLocations(Request $request, int $warehouseId): JsonResponse
    {
        $locations = $this->warehouseService->getWarehouseLocations(
            $warehouseId,
            $request->only(['zone_id', 'type', 'available'])
        );

        return response()->json([
            'success' => true,
            'data' => $locations
        ]);
    }

    /**
     * GET /api/warehouses/{warehouse}/capacity
     * Get warehouse capacity information
     */
    public function getCapacity(int $warehouseId): JsonResponse
    {
        $capacity = $this->warehouseService->getWarehouseCapacity($warehouseId);

        return response()->json([
            'success' => true,
            'data' => $capacity
        ]);
    }
}
```

### 2. Product Management

#### Product Operations
```php
<?php

namespace App\Modules\Warehouses\Presentation\Controllers\Api;

use App\Http\Controllers\Controller;
use App\Modules\Warehouses\Application\Services\ProductService;
use Illuminate\Http\Request;
use Illuminate\Http\JsonResponse;

class ProductController extends Controller
{
    protected ProductService $productService;

    public function __construct(ProductService $productService)
    {
        $this->productService = $productService;
    }

    /**
     * GET /api/products
     * List products with advanced filtering and search
     */
    public function index(Request $request): JsonResponse
    {
        $products = $this->productService->getProducts(
            auth()->user()->company_id,
            $request->only([
                'category_id', 'type', 'status', 'brand', 'search',
                'has_expiry', 'is_serialized', 'is_lot_tracked',
                'sort_by', 'sort_direction'
            ])
        );

        return response()->json([
            'success' => true,
            'data' => $products,
            'meta' => [
                'total' => $products->total(),
                'per_page' => $products->perPage(),
                'current_page' => $products->currentPage(),
            ]
        ]);
    }

    /**
     * POST /api/products
     * Create a new product
     */
    public function store(StoreProductRequest $request): JsonResponse
    {
        $product = $this->productService->createProduct(
            auth()->user()->company_id,
            $request->validated()
        );

        return response()->json([
            'success' => true,
            'message' => 'Product created successfully',
            'data' => $product
        ], 201);
    }

    /**
     * GET /api/products/{product}
     * Get product details with variants and identifiers
     */
    public function show(int $productId): JsonResponse
    {
        $product = $this->productService->getProductWithDetails($productId);

        return response()->json([
            'success' => true,
            'data' => $product
        ]);
    }

    /**
     * PUT /api/products/{product}
     * Update product information
     */
    public function update(UpdateProductRequest $request, int $productId): JsonResponse
    {
        $product = $this->productService->updateProduct($productId, $request->validated());

        return response()->json([
            'success' => true,
            'message' => 'Product updated successfully',
            'data' => $product
        ]);
    }

    /**
     * DELETE /api/products/{product}
     * Soft delete a product
     */
    public function destroy(int $productId): JsonResponse
    {
        $this->productService->deleteProduct($productId);

        return response()->json([
            'success' => true,
            'message' => 'Product deleted successfully'
        ]);
    }

    /**
     * GET /api/products/search
     * Advanced product search with multiple identifiers
     */
    public function search(Request $request): JsonResponse
    {
        $products = $this->productService->searchProducts(
            auth()->user()->company_id,
            $request->get('query'),
            $request->only(['identifier_type', 'category', 'type'])
        );

        return response()->json([
            'success' => true,
            'data' => $products
        ]);
    }

    /**
     * GET /api/products/{product}/identifiers
     * Get product identifiers (IMPA, SKU, Barcodes)
     */
    public function getIdentifiers(int $productId): JsonResponse
    {
        $identifiers = $this->productService->getProductIdentifiers($productId);

        return response()->json([
            'success' => true,
            'data' => $identifiers
        ]);
    }

    /**
     * POST /api/products/{product}/identifiers
     * Add new identifier to product
     */
    public function addIdentifier(AddIdentifierRequest $request, int $productId): JsonResponse
    {
        $identifier = $this->productService->addProductIdentifier(
            $productId,
            $request->validated()
        );

        return response()->json([
            'success' => true,
            'message' => 'Identifier added successfully',
            'data' => $identifier
        ], 201);
    }

    /**
     * GET /api/products/{product}/variants
     * Get product variants (sizes, colors, etc.)
     */
    public function getVariants(int $productId): JsonResponse
    {
        $variants = $this->productService->getProductVariants($productId);

        return response()->json([
            'success' => true,
            'data' => $variants
        ]);
    }

    /**
     * POST /api/products/{product}/variants
     * Create product variant
     */
    public function createVariant(CreateVariantRequest $request, int $productId): JsonResponse
    {
        $variant = $this->productService->createProductVariant(
            $productId,
            $request->validated()
        );

        return response()->json([
            'success' => true,
            'message' => 'Product variant created successfully',
            'data' => $variant
        ], 201);
    }
}
```

### 3. Inventory Management

#### Inventory Operations
```php
<?php

namespace App\Modules\Warehouses\Presentation\Controllers\Api;

use App\Http\Controllers\Controller;
use App\Modules\Warehouses\Application\Services\InventoryService;
use Illuminate\Http\Request;
use Illuminate\Http\JsonResponse;

class InventoryController extends Controller
{
    protected InventoryService $inventoryService;

    public function __construct(InventoryService $inventoryService)
    {
        $this->inventoryService = $inventoryService;
    }

    /**
     * GET /api/inventory
     * Get inventory levels across warehouses
     */
    public function index(Request $request): JsonResponse
    {
        $inventory = $this->inventoryService->getInventoryLevels(
            auth()->user()->company_id,
            $request->only(['warehouse_id', 'product_id', 'location_id', 'status'])
        );

        return response()->json([
            'success' => true,
            'data' => $inventory
        ]);
    }

    /**
     * GET /api/inventory/summary
     * Get inventory summary statistics
     */
    public function getSummary(Request $request): JsonResponse
    {
        $summary = $this->inventoryService->getInventorySummary(
            auth()->user()->company_id,
            $request->get('warehouse_id')
        );

        return response()->json([
            'success' => true,
            'data' => $summary
        ]);
    }

    /**
     * GET /api/inventory/{product}/levels
     * Get stock levels for specific product across all warehouses
     */
    public function getProductLevels(int $productId): JsonResponse
    {
        $levels = $this->inventoryService->getProductStockLevels($productId);

        return response()->json([
            'success' => true,
            'data' => $levels
        ]);
    }

    /**
     * POST /api/inventory/receipt
     * Record goods receipt
     */
    public function receipt(ReceiptRequest $request): JsonResponse
    {
        $receipt = $this->inventoryService->processReceipt(
            auth()->user()->company_id,
            $request->validated()
        );

        return response()->json([
            'success' => true,
            'message' => 'Receipt processed successfully',
            'data' => $receipt
        ], 201);
    }

    /**
     * POST /api/inventory/issue
     * Record stock issue
     */
    public function issue(IssueRequest $request): JsonResponse
    {
        $issue = $this->inventoryService->processIssue(
            auth()->user()->company_id,
            $request->validated()
        );

        return response()->json([
            'success' => true,
            'message' => 'Issue processed successfully',
            'data' => $issue
        ], 201);
    }

    /**
     * POST /api/inventory/transfer
     * Process inter-warehouse transfer
     */
    public function transfer(TransferRequest $request): JsonResponse
    {
        $transfer = $this->inventoryService->processTransfer(
            auth()->user()->company_id,
            $request->validated()
        );

        return response()->json([
            'success' => true,
            'message' => 'Transfer processed successfully',
            'data' => $transfer
        ], 201);
    }

    /**
     * POST /api/inventory/adjustment
     * Record stock adjustment
     */
    public function adjustment(AdjustmentRequest $request): JsonResponse
    {
        $adjustment = $this->inventoryService->processAdjustment(
            auth()->user()->company_id,
            $request->validated()
        );

        return response()->json([
            'success' => true,
            'message' => 'Adjustment processed successfully',
            'data' => $adjustment
        ], 201);
    }

    /**
     * GET /api/inventory/movements
     * Get stock movement history
     */
    public function getMovements(Request $request): JsonResponse
    {
        $movements = $this->inventoryService->getStockMovements(
            auth()->user()->company_id,
            $request->only([
                'warehouse_id', 'product_id', 'movement_type',
                'date_from', 'date_to', 'reference_type', 'reference_id'
            ])
        );

        return response()->json([
            'success' => true,
            'data' => $movements
        ]);
    }

    /**
     * GET /api/inventory/alerts
     * Get inventory alerts (low stock, expiry warnings, etc.)
     */
    public function getAlerts(Request $request): JsonResponse
    {
        $alerts = $this->inventoryService->getInventoryAlerts(
            auth()->user()->company_id,
            $request->only(['warehouse_id', 'alert_type', 'severity', 'status'])
        );

        return response()->json([
            'success' => true,
            'data' => $alerts
        ]);
    }

    /**
     * POST /api/inventory/alerts/{alert}/acknowledge
     * Acknowledge an inventory alert
     */
    public function acknowledgeAlert(int $alertId): JsonResponse
    {
        $this->inventoryService->acknowledgeAlert($alertId, auth()->id());

        return response()->json([
            'success' => true,
            'message' => 'Alert acknowledged successfully'
        ]);
    }
}
```

### 4. Advanced Tracking Features

#### Expiry Date Management
```php
<?php

namespace App\Modules\Warehouses\Presentation\Controllers\Api;

use App\Http\Controllers\Controller;
use App\Modules\Warehouses\Application\Services\ExpiryService;
use Illuminate\Http\Request;
use Illuminate\Http\JsonResponse;

class ExpiryController extends Controller
{
    protected ExpiryService $expiryService;

    public function __construct(ExpiryService $expiryService)
    {
        $this->expiryService = $expiryService;
    }

    /**
     * GET /api/inventory/expiry/alerts
     * Get expiry alerts and warnings
     */
    public function getExpiryAlerts(Request $request): JsonResponse
    {
        $alerts = $this->expiryService->getExpiryAlerts(
            auth()->user()->company_id,
            $request->only(['warehouse_id', 'days_ahead', 'severity'])
        );

        return response()->json([
            'success' => true,
            'data' => $alerts
        ]);
    }

    /**
     * GET /api/inventory/expiry/products
     * Get products with expiry dates
     */
    public function getExpiringProducts(Request $request): JsonResponse
    {
        $products = $this->expiryService->getExpiringProducts(
            auth()->user()->company_id,
            $request->get('warehouse_id'),
            $request->get('days_ahead', 30)
        );

        return response()->json([
            'success' => true,
            'data' => $products
        ]);
    }

    /**
     * POST /api/inventory/expiry/batch-update
     * Batch update expiry dates
     */
    public function batchUpdateExpiry(BatchUpdateExpiryRequest $request): JsonResponse
    {
        $result = $this->expiryService->batchUpdateExpiryDates(
            $request->validated()
        );

        return response()->json([
            'success' => true,
            'message' => 'Expiry dates updated successfully',
            'data' => $result
        ]);
    }

    /**
     * GET /api/inventory/expiry/reports
     * Generate expiry reports
     */
    public function getExpiryReport(Request $request): JsonResponse
    {
        $report = $this->expiryService->generateExpiryReport(
            auth()->user()->company_id,
            $request->only(['warehouse_id', 'category_id', 'date_range'])
        );

        return response()->json([
            'success' => true,
            'data' => $report
        ]);
    }
}
```

#### Serial Number Tracking
```php
<?php

namespace App\Modules\Warehouses\Presentation\Controllers\Api;

use App\Http\Controllers\Controller;
use App\Modules\Warehouses\Application\Services\SerialNumberService;
use Illuminate\Http\Request;
use Illuminate\Http\JsonResponse;

class SerialNumberController extends Controller
{
    protected SerialNumberService $serialNumberService;

    public function __construct(SerialNumberService $serialNumberService)
    {
        $this->serialNumberService = $serialNumberService;
    }

    /**
     * GET /api/inventory/serials
     * Get serial numbers with tracking information
     */
    public function index(Request $request): JsonResponse
    {
        $serials = $this->serialNumberService->getSerialNumbers(
            auth()->user()->company_id,
            $request->only(['product_id', 'warehouse_id', 'status', 'search'])
        );

        return response()->json([
            'success' => true,
            'data' => $serials
        ]);
    }

    /**
     * POST /api/inventory/serials
     * Generate or add serial numbers
     */
    public function store(StoreSerialRequest $request): JsonResponse
    {
        $serials = $this->serialNumberService->createSerialNumbers(
            $request->validated()
        );

        return response()->json([
            'success' => true,
            'message' => 'Serial numbers created successfully',
            'data' => $serials
        ], 201);
    }

    /**
     * GET /api/inventory/serials/{serial}/tracking
     * Get serial number tracking history
     */
    public function getTracking(string $serialNumber): JsonResponse
    {
        $tracking = $this->serialNumberService->getSerialTracking($serialNumber);

        return response()->json([
            'success' => true,
            'data' => $tracking
        ]);
    }

    /**
     * POST /api/inventory/serials/{serial}/move
     * Move serial number to new location
     */
    public function moveSerial(MoveSerialRequest $request, int $serialId): JsonResponse
    {
        $movement = $this->serialNumberService->moveSerial(
            $serialId,
            $request->validated()
        );

        return response()->json([
            'success' => true,
            'message' => 'Serial number moved successfully',
            'data' => $movement
        ]);
    }

    /**
     * GET /api/inventory/serials/{serial}/warranty
     * Get warranty information for serial number
     */
    public function getWarranty(int $serialId): JsonResponse
    {
        $warranty = $this->serialNumberService->getWarrantyInfo($serialId);

        return response()->json([
            'success' => true,
            'data' => $warranty
        ]);
    }
}
```

### 5. Batch/Lot Management

#### Batch Operations
```php
<?php

namespace App\Modules\Warehouses\Presentation\Controllers\Api;

use App\Http\Controllers\Controller;
use App\Modules\Warehouses\Application\Services\BatchLotService;
use Illuminate\Http\Request;
use Illuminate\Http\JsonResponse;

class BatchLotController extends Controller
{
    protected BatchLotService $batchLotService;

    public function __construct(BatchLotService $batchLotService)
    {
        $this->batchLotService = $batchLotService;
    }

    /**
     * GET /api/inventory/batches
     * Get batch/lot information
     */
    public function index(Request $request): JsonResponse
    {
        $batches = $this->batchLotService->getBatches(
            auth()->user()->company_id,
            $request->only(['product_id', 'warehouse_id', 'status', 'expiry_range'])
        );

        return response()->json([
            'success' => true,
            'data' => $batches
        ]);
    }

    /**
     * POST /api/inventory/batches
     * Create new batch/lot
     */
    public function store(StoreBatchRequest $request): JsonResponse
    {
        $batch = $this->batchLotService->createBatch($request->validated());

        return response()->json([
            'success' => true,
            'message' => 'Batch created successfully',
            'data' => $batch
        ], 201);
    }

    /**
     * GET /api/inventory/batches/{batch}/tracking
     * Get batch movement tracking
     */
    public function getTracking(int $batchId): JsonResponse
    {
        $tracking = $this->batchLotService->getBatchTracking($batchId);

        return response()->json([
            'success' => true,
            'data' => $tracking
        ]);
    }

    /**
     * POST /api/inventory/batches/{batch}/quality-check
     * Record quality check for batch
     */
    public function qualityCheck(QualityCheckRequest $request, int $batchId): JsonResponse
    {
        $result = $this->batchLotService->recordQualityCheck(
            $batchId,
            $request->validated()
        );

        return response()->json([
            'success' => true,
            'message' => 'Quality check recorded successfully',
            'data' => $result
        ]);
    }

    /**
     * GET /api/inventory/batches/fifo-queue
     * Get FIFO queue for product batches
     */
    public function getFifoQueue(Request $request): JsonResponse
    {
        $queue = $this->batchLotService->getFifoQueue(
            $request->get('product_id'),
            $request->get('warehouse_id')
        );

        return response()->json([
            'success' => true,
            'data' => $queue
        ]);
    }
}
```

## API Response Format

### Standard Success Response
```json
{
    "success": true,
    "message": "Operation completed successfully",
    "data": {
        // Resource data
    },
    "meta": {
        "total": 150,
        "per_page": 20,
        "current_page": 1,
        "last_page": 8
    }
}
```

### Standard Error Response
```json
{
    "success": false,
    "message": "Operation failed",
    "errors": {
        "field_name": ["Error message"]
    },
    "error_code": "VALIDATION_ERROR"
}
```

## Real-time Features

### WebSocket Integration
```php
// Broadcasting inventory changes
broadcast(new InventoryUpdated($inventoryItem))->toOthers();

// Stock alerts
broadcast(new StockAlertCreated($alert))->to("company.{$companyId}");

// Expiry warnings
broadcast(new ExpiryWarning($products))->to("warehouse.{$warehouseId}");
```

### Event Listeners
```php
// Real-time stock level updates
Event::listen(StockMovementCreated::class, UpdateInventoryLevels::class);

// Automatic alert generation
Event::listen(InventoryUpdated::class, CheckStockAlerts::class);

// Expiry date monitoring
Event::listen(BatchCreated::class, ScheduleExpiryChecks::class);
```

This comprehensive API architecture provides:

1. **Complete CRUD operations** for all warehouse entities
2. **Advanced search and filtering** capabilities
3. **Multi-warehouse support** with company/branch isolation
4. **Real-time updates** through WebSocket integration
5. **Comprehensive tracking** for serials, batches, and expiry dates
6. **Professional error handling** and response formatting
7. **Performance optimization** through efficient queries
8. **Scalable architecture** supporting future enhancements

The API follows REST principles and Laravel best practices, ensuring maintainability and extensibility.