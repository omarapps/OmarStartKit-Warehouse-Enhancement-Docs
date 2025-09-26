# Database Schema Design for Enhanced Warehouse Module

## Overview
This document outlines the complete database schema for the enhanced warehouse module, supporting multi-warehouse management, advanced product tracking, IMPA/SKU/Barcode systems, expiry dates, and serial numbers.

## Core Tables Structure

### 1. Enhanced Warehouse Tables

#### warehouses (Enhanced)
```sql
ALTER TABLE warehouses ADD COLUMN IF NOT EXISTS warehouse_type VARCHAR(50) DEFAULT 'general';
ALTER TABLE warehouses ADD COLUMN IF NOT EXISTS climate_controlled BOOLEAN DEFAULT FALSE;
ALTER TABLE warehouses ADD COLUMN IF NOT EXISTS hazmat_certified BOOLEAN DEFAULT FALSE;
ALTER TABLE warehouses ADD COLUMN IF NOT EXISTS max_capacity_cubic_meters DECIMAL(10,2);
ALTER TABLE warehouses ADD COLUMN IF NOT EXISTS max_weight_capacity_kg DECIMAL(10,2);
ALTER TABLE warehouses ADD COLUMN IF NOT EXISTS security_clearance_level INTEGER DEFAULT 1;
ALTER TABLE warehouses ADD COLUMN IF NOT EXISTS operating_license VARCHAR(100);
ALTER TABLE warehouses ADD COLUMN IF NOT EXISTS certification_expiry DATE;
ALTER TABLE warehouses ADD COLUMN IF NOT EXISTS emergency_contact JSON;

-- Index for performance
CREATE INDEX idx_warehouses_type ON warehouses(warehouse_type);
CREATE INDEX idx_warehouses_branch ON warehouses(branch_id);
CREATE INDEX idx_warehouses_status ON warehouses(status, is_active);
```

#### warehouse_zones (New)
```sql
CREATE TABLE warehouse_zones (
    id BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    warehouse_id BIGINT UNSIGNED NOT NULL,
    code VARCHAR(20) NOT NULL,
    name VARCHAR(100) NOT NULL,
    description TEXT,
    zone_type ENUM('receiving', 'storage', 'picking', 'shipping', 'quarantine', 'returns') NOT NULL,
    temperature_min DECIMAL(5,2),
    temperature_max DECIMAL(5,2),
    humidity_min DECIMAL(5,2),
    humidity_max DECIMAL(5,2),
    is_climate_controlled BOOLEAN DEFAULT FALSE,
    is_hazmat_zone BOOLEAN DEFAULT FALSE,
    max_capacity_cubic_meters DECIMAL(10,2),
    current_utilization_percentage DECIMAL(5,2) DEFAULT 0,
    is_active BOOLEAN DEFAULT TRUE,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    
    FOREIGN KEY (warehouse_id) REFERENCES warehouses(id) ON DELETE CASCADE,
    UNIQUE KEY uk_warehouse_zone_code (warehouse_id, code),
    INDEX idx_warehouse_zones_type (zone_type),
    INDEX idx_warehouse_zones_active (is_active)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci;
```

#### warehouse_locations (New)
```sql
CREATE TABLE warehouse_locations (
    id BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    warehouse_id BIGINT UNSIGNED NOT NULL,
    zone_id BIGINT UNSIGNED,
    aisle VARCHAR(10),
    rack VARCHAR(10),
    shelf VARCHAR(10),
    bin VARCHAR(10),
    full_location_code VARCHAR(50) NOT NULL,
    barcode VARCHAR(100),
    location_type ENUM('bulk', 'pick', 'reserve', 'floor', 'overhead') DEFAULT 'bulk',
    capacity_cubic_meters DECIMAL(8,2),
    max_weight_kg DECIMAL(8,2),
    restrictions JSON, -- {"temperature_sensitive": true, "hazmat_only": false}
    is_pickable BOOLEAN DEFAULT TRUE,
    is_active BOOLEAN DEFAULT TRUE,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    
    FOREIGN KEY (warehouse_id) REFERENCES warehouses(id) ON DELETE CASCADE,
    FOREIGN KEY (zone_id) REFERENCES warehouse_zones(id) ON DELETE SET NULL,
    UNIQUE KEY uk_warehouse_location_code (warehouse_id, full_location_code),
    UNIQUE KEY uk_warehouse_location_barcode (barcode),
    INDEX idx_warehouse_locations_zone (zone_id),
    INDEX idx_warehouse_locations_type (location_type),
    INDEX idx_warehouse_locations_active (is_active)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci;
```

### 2. Product Management Tables

#### product_categories (New)
```sql
CREATE TABLE product_categories (
    id BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    parent_id BIGINT UNSIGNED NULL,
    code VARCHAR(20) NOT NULL UNIQUE,
    name VARCHAR(100) NOT NULL,
    description TEXT,
    requires_expiry BOOLEAN DEFAULT FALSE,
    requires_serial BOOLEAN DEFAULT FALSE,
    requires_lot_batch BOOLEAN DEFAULT FALSE,
    default_shelf_life_days INTEGER,
    storage_requirements JSON,
    handling_instructions TEXT,
    is_hazmat BOOLEAN DEFAULT FALSE,
    is_temperature_sensitive BOOLEAN DEFAULT FALSE,
    is_active BOOLEAN DEFAULT TRUE,
    sort_order INTEGER DEFAULT 0,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    
    FOREIGN KEY (parent_id) REFERENCES product_categories(id) ON DELETE SET NULL,
    INDEX idx_product_categories_parent (parent_id),
    INDEX idx_product_categories_active (is_active),
    INDEX idx_product_categories_code (code)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci;
```

#### products (New)
```sql
CREATE TABLE products (
    id BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    company_id BIGINT UNSIGNED NOT NULL,
    category_id BIGINT UNSIGNED,
    name VARCHAR(200) NOT NULL,
    description TEXT,
    brand VARCHAR(100),
    model VARCHAR(100),
    manufacturer VARCHAR(100),
    country_of_origin VARCHAR(2),
    
    -- Physical Properties
    length_cm DECIMAL(8,2),
    width_cm DECIMAL(8,2),
    height_cm DECIMAL(8,2),
    weight_kg DECIMAL(8,3),
    volume_cubic_cm DECIMAL(12,2),
    
    -- Product Classification
    product_type ENUM('finished_good', 'raw_material', 'work_in_progress', 'spare_part', 'consumable', 'equipment') NOT NULL,
    unit_of_measure VARCHAR(20) NOT NULL DEFAULT 'pcs',
    base_cost DECIMAL(10,4),
    standard_price DECIMAL(10,2),
    currency_code VARCHAR(3) DEFAULT 'USD',
    
    -- Inventory Settings
    is_serialized BOOLEAN DEFAULT FALSE,
    is_lot_tracked BOOLEAN DEFAULT FALSE,
    has_expiry BOOLEAN DEFAULT FALSE,
    shelf_life_days INTEGER,
    reorder_point INTEGER DEFAULT 0,
    min_stock_level INTEGER DEFAULT 0,
    max_stock_level INTEGER,
    lead_time_days INTEGER DEFAULT 0,
    
    -- Status
    status ENUM('active', 'inactive', 'discontinued', 'pending') DEFAULT 'active',
    is_active BOOLEAN DEFAULT TRUE,
    created_by BIGINT UNSIGNED,
    updated_by BIGINT UNSIGNED,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    deleted_at TIMESTAMP NULL,
    
    FOREIGN KEY (company_id) REFERENCES companies(id) ON DELETE CASCADE,
    FOREIGN KEY (category_id) REFERENCES product_categories(id) ON DELETE SET NULL,
    FOREIGN KEY (created_by) REFERENCES users(id) ON DELETE SET NULL,
    FOREIGN KEY (updated_by) REFERENCES users(id) ON DELETE SET NULL,
    
    INDEX idx_products_company (company_id),
    INDEX idx_products_category (category_id),
    INDEX idx_products_type (product_type),
    INDEX idx_products_status (status, is_active),
    INDEX idx_products_name (name),
    INDEX idx_products_brand (brand),
    FULLTEXT idx_products_search (name, description, brand, model, manufacturer)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci;
```

#### product_identifiers (New)
```sql
CREATE TABLE product_identifiers (
    id BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    product_id BIGINT UNSIGNED NOT NULL,
    identifier_type ENUM('impa', 'sku', 'upc', 'ean', 'gtin', 'manufacturer_part', 'internal') NOT NULL,
    identifier_value VARCHAR(100) NOT NULL,
    is_primary BOOLEAN DEFAULT FALSE,
    verified_at TIMESTAMP NULL,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    
    FOREIGN KEY (product_id) REFERENCES products(id) ON DELETE CASCADE,
    UNIQUE KEY uk_product_identifier (identifier_type, identifier_value),
    INDEX idx_product_identifiers_product (product_id),
    INDEX idx_product_identifiers_type (identifier_type),
    INDEX idx_product_identifiers_value (identifier_value),
    INDEX idx_product_identifiers_primary (is_primary)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci;
```

#### product_variants (New)
```sql
CREATE TABLE product_variants (
    id BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    product_id BIGINT UNSIGNED NOT NULL,
    variant_name VARCHAR(100),
    
    -- Size Variants
    size_type ENUM('numeric', 'text', 'dimension') DEFAULT 'text',
    size_value VARCHAR(50),
    size_unit VARCHAR(20),
    
    -- Color Variants
    color_name VARCHAR(50),
    color_code VARCHAR(7), -- Hex color
    color_family VARCHAR(30),
    
    -- Style/Material Variants
    material VARCHAR(100),
    finish VARCHAR(100),
    style VARCHAR(100),
    
    -- Pricing
    price_adjustment DECIMAL(10,2) DEFAULT 0,
    cost_adjustment DECIMAL(10,2) DEFAULT 0,
    
    -- Stock
    additional_sku VARCHAR(100),
    additional_barcode VARCHAR(100),
    
    is_active BOOLEAN DEFAULT TRUE,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    
    FOREIGN KEY (product_id) REFERENCES products(id) ON DELETE CASCADE,
    UNIQUE KEY uk_product_variant_sku (additional_sku),
    UNIQUE KEY uk_product_variant_barcode (additional_barcode),
    INDEX idx_product_variants_product (product_id),
    INDEX idx_product_variants_size (size_value),
    INDEX idx_product_variants_color (color_name),
    INDEX idx_product_variants_active (is_active)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci;
```

### 3. Inventory Management Tables

#### inventory_items (New)
```sql
CREATE TABLE inventory_items (
    id BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    product_id BIGINT UNSIGNED NOT NULL,
    product_variant_id BIGINT UNSIGNED NULL,
    warehouse_id BIGINT UNSIGNED NOT NULL,
    location_id BIGINT UNSIGNED NULL,
    
    -- Stock Levels
    quantity_on_hand DECIMAL(12,3) DEFAULT 0,
    quantity_allocated DECIMAL(12,3) DEFAULT 0,
    quantity_available DECIMAL(12,3) DEFAULT 0,
    quantity_in_transit DECIMAL(12,3) DEFAULT 0,
    
    -- Costing
    average_cost DECIMAL(10,4) DEFAULT 0,
    last_cost DECIMAL(10,4) DEFAULT 0,
    total_value DECIMAL(15,2) DEFAULT 0,
    
    -- Dates
    last_received_at TIMESTAMP NULL,
    last_issued_at TIMESTAMP NULL,
    last_counted_at TIMESTAMP NULL,
    next_count_due_at TIMESTAMP NULL,
    
    -- Status
    status ENUM('active', 'hold', 'quarantine', 'obsolete') DEFAULT 'active',
    notes TEXT,
    
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    
    FOREIGN KEY (product_id) REFERENCES products(id) ON DELETE CASCADE,
    FOREIGN KEY (product_variant_id) REFERENCES product_variants(id) ON DELETE CASCADE,
    FOREIGN KEY (warehouse_id) REFERENCES warehouses(id) ON DELETE CASCADE,
    FOREIGN KEY (location_id) REFERENCES warehouse_locations(id) ON DELETE SET NULL,
    
    UNIQUE KEY uk_inventory_item (product_id, product_variant_id, warehouse_id, location_id),
    INDEX idx_inventory_items_warehouse (warehouse_id),
    INDEX idx_inventory_items_location (location_id),
    INDEX idx_inventory_items_status (status),
    INDEX idx_inventory_items_quantity (quantity_on_hand)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci;
```

#### stock_movements (New)
```sql
CREATE TABLE stock_movements (
    id BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    movement_number VARCHAR(50) NOT NULL UNIQUE,
    product_id BIGINT UNSIGNED NOT NULL,
    product_variant_id BIGINT UNSIGNED NULL,
    warehouse_id BIGINT UNSIGNED NOT NULL,
    location_id BIGINT UNSIGNED NULL,
    
    -- Movement Details
    movement_type ENUM('receipt', 'issue', 'transfer_in', 'transfer_out', 'adjustment', 'return', 'scrap', 'cycle_count') NOT NULL,
    reference_type VARCHAR(50), -- 'purchase_order', 'sales_order', 'transfer_order', etc.
    reference_id BIGINT UNSIGNED,
    reference_number VARCHAR(100),
    
    -- Quantities
    quantity_before DECIMAL(12,3),
    quantity_moved DECIMAL(12,3) NOT NULL,
    quantity_after DECIMAL(12,3),
    unit_cost DECIMAL(10,4),
    total_cost DECIMAL(15,2),
    
    -- Batch/Lot/Serial Tracking
    lot_number VARCHAR(100),
    serial_number VARCHAR(100),
    expiry_date DATE,
    manufacturing_date DATE,
    
    -- Movement Details
    reason_code VARCHAR(50),
    notes TEXT,
    movement_date DATETIME NOT NULL,
    processed_by BIGINT UNSIGNED,
    approved_by BIGINT UNSIGNED,
    
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    
    FOREIGN KEY (product_id) REFERENCES products(id) ON DELETE CASCADE,
    FOREIGN KEY (product_variant_id) REFERENCES product_variants(id) ON DELETE CASCADE,
    FOREIGN KEY (warehouse_id) REFERENCES warehouses(id) ON DELETE CASCADE,
    FOREIGN KEY (location_id) REFERENCES warehouse_locations(id) ON DELETE SET NULL,
    FOREIGN KEY (processed_by) REFERENCES users(id) ON DELETE SET NULL,
    FOREIGN KEY (approved_by) REFERENCES users(id) ON DELETE SET NULL,
    
    INDEX idx_stock_movements_product (product_id),
    INDEX idx_stock_movements_warehouse (warehouse_id),
    INDEX idx_stock_movements_type (movement_type),
    INDEX idx_stock_movements_date (movement_date),
    INDEX idx_stock_movements_reference (reference_type, reference_id),
    INDEX idx_stock_movements_lot (lot_number),
    INDEX idx_stock_movements_serial (serial_number),
    INDEX idx_stock_movements_expiry (expiry_date)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci;
```

#### batch_lots (New)
```sql
CREATE TABLE batch_lots (
    id BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    product_id BIGINT UNSIGNED NOT NULL,
    lot_number VARCHAR(100) NOT NULL,
    batch_number VARCHAR(100),
    
    -- Dates
    manufacturing_date DATE,
    expiry_date DATE,
    received_date DATE,
    
    -- Supplier Information
    supplier_id BIGINT UNSIGNED NULL,
    supplier_lot_number VARCHAR(100),
    certificate_of_analysis TEXT,
    
    -- Quality Information
    quality_status ENUM('approved', 'pending', 'rejected', 'quarantine') DEFAULT 'pending',
    quality_notes TEXT,
    tested_by BIGINT UNSIGNED NULL,
    tested_at TIMESTAMP NULL,
    
    -- Quantities
    initial_quantity DECIMAL(12,3) NOT NULL,
    current_quantity DECIMAL(12,3) NOT NULL,
    allocated_quantity DECIMAL(12,3) DEFAULT 0,
    
    -- Status
    status ENUM('active', 'consumed', 'expired', 'recalled') DEFAULT 'active',
    notes TEXT,
    
    created_by BIGINT UNSIGNED,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    
    FOREIGN KEY (product_id) REFERENCES products(id) ON DELETE CASCADE,
    FOREIGN KEY (tested_by) REFERENCES users(id) ON DELETE SET NULL,
    FOREIGN KEY (created_by) REFERENCES users(id) ON DELETE SET NULL,
    
    UNIQUE KEY uk_product_lot (product_id, lot_number),
    INDEX idx_batch_lots_product (product_id),
    INDEX idx_batch_lots_expiry (expiry_date),
    INDEX idx_batch_lots_status (status),
    INDEX idx_batch_lots_quality (quality_status)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci;
```

#### serial_numbers (New)
```sql
CREATE TABLE serial_numbers (
    id BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    product_id BIGINT UNSIGNED NOT NULL,
    product_variant_id BIGINT UNSIGNED NULL,
    serial_number VARCHAR(100) NOT NULL UNIQUE,
    
    -- Location Tracking
    current_warehouse_id BIGINT UNSIGNED,
    current_location_id BIGINT UNSIGNED,
    
    -- Status Tracking
    status ENUM('in_stock', 'allocated', 'shipped', 'sold', 'returned', 'scrapped', 'warranty') DEFAULT 'in_stock',
    
    -- Manufacturing Information
    manufacturing_date DATE,
    manufacturing_batch VARCHAR(100),
    manufacturer_serial VARCHAR(100),
    
    -- Warranty Information
    warranty_start_date DATE,
    warranty_end_date DATE,
    warranty_terms TEXT,
    
    -- Movement History (Current Location)
    last_movement_id BIGINT UNSIGNED,
    last_movement_date DATETIME,
    
    -- Customer/Order Information
    sold_to_customer VARCHAR(200),
    sold_date DATE,
    sales_order_number VARCHAR(100),
    
    notes TEXT,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    
    FOREIGN KEY (product_id) REFERENCES products(id) ON DELETE CASCADE,
    FOREIGN KEY (product_variant_id) REFERENCES product_variants(id) ON DELETE CASCADE,
    FOREIGN KEY (current_warehouse_id) REFERENCES warehouses(id) ON DELETE SET NULL,
    FOREIGN KEY (current_location_id) REFERENCES warehouse_locations(id) ON DELETE SET NULL,
    
    INDEX idx_serial_numbers_product (product_id),
    INDEX idx_serial_numbers_status (status),
    INDEX idx_serial_numbers_warehouse (current_warehouse_id),
    INDEX idx_serial_numbers_warranty (warranty_end_date),
    INDEX idx_serial_numbers_manufacturing (manufacturing_date)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci;
```

### 4. Operational Tables

#### stock_alerts (New)
```sql
CREATE TABLE stock_alerts (
    id BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    product_id BIGINT UNSIGNED NOT NULL,
    warehouse_id BIGINT UNSIGNED NOT NULL,
    alert_type ENUM('low_stock', 'expiry_warning', 'overstock', 'no_movement', 'quality_hold') NOT NULL,
    severity ENUM('low', 'medium', 'high', 'critical') DEFAULT 'medium',
    
    current_quantity DECIMAL(12,3),
    threshold_quantity DECIMAL(12,3),
    alert_message TEXT,
    
    status ENUM('active', 'acknowledged', 'resolved') DEFAULT 'active',
    acknowledged_by BIGINT UNSIGNED NULL,
    acknowledged_at TIMESTAMP NULL,
    resolved_at TIMESTAMP NULL,
    
    alert_date DATETIME NOT NULL,
    expires_at DATETIME NULL,
    
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    
    FOREIGN KEY (product_id) REFERENCES products(id) ON DELETE CASCADE,
    FOREIGN KEY (warehouse_id) REFERENCES warehouses(id) ON DELETE CASCADE,
    FOREIGN KEY (acknowledged_by) REFERENCES users(id) ON DELETE SET NULL,
    
    INDEX idx_stock_alerts_product (product_id),
    INDEX idx_stock_alerts_warehouse (warehouse_id),
    INDEX idx_stock_alerts_type (alert_type),
    INDEX idx_stock_alerts_status (status),
    INDEX idx_stock_alerts_severity (severity),
    INDEX idx_stock_alerts_date (alert_date)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci;
```

#### inventory_transactions (New)
```sql
CREATE TABLE inventory_transactions (
    id BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    transaction_number VARCHAR(50) NOT NULL UNIQUE,
    transaction_type ENUM('receipt', 'issue', 'transfer', 'adjustment', 'count') NOT NULL,
    
    from_warehouse_id BIGINT UNSIGNED NULL,
    to_warehouse_id BIGINT UNSIGNED NULL,
    
    reference_document VARCHAR(100),
    reference_number VARCHAR(100),
    
    total_items INTEGER DEFAULT 0,
    total_quantity DECIMAL(15,3) DEFAULT 0,
    total_value DECIMAL(15,2) DEFAULT 0,
    
    status ENUM('draft', 'processing', 'completed', 'cancelled') DEFAULT 'draft',
    processed_by BIGINT UNSIGNED,
    approved_by BIGINT UNSIGNED,
    
    transaction_date DATETIME NOT NULL,
    notes TEXT,
    
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    
    FOREIGN KEY (from_warehouse_id) REFERENCES warehouses(id) ON DELETE SET NULL,
    FOREIGN KEY (to_warehouse_id) REFERENCES warehouses(id) ON DELETE SET NULL,
    FOREIGN KEY (processed_by) REFERENCES users(id) ON DELETE SET NULL,
    FOREIGN KEY (approved_by) REFERENCES users(id) ON DELETE SET NULL,
    
    INDEX idx_inventory_transactions_type (transaction_type),
    INDEX idx_inventory_transactions_status (status),
    INDEX idx_inventory_transactions_date (transaction_date),
    INDEX idx_inventory_transactions_from_warehouse (from_warehouse_id),
    INDEX idx_inventory_transactions_to_warehouse (to_warehouse_id)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci;
```

## Key Features of This Schema

### 1. Multi-Warehouse Support
- Warehouses linked to company branches
- Multiple warehouse types supported
- Zone and location hierarchy

### 2. Product Management
- Comprehensive product information
- Multi-standard identification (IMPA, SKU, UPC, etc.)
- Product variants for size, color, style
- Category hierarchy with specific requirements

### 3. Advanced Tracking
- **Expiry Date Management**: Full expiry tracking with alerts
- **Serial Number Tracking**: Individual item tracking with warranty
- **Batch/Lot Management**: Manufacturing and quality control
- **Multi-location Inventory**: Precise location tracking

### 4. Performance Optimization
- Strategic indexing for fast queries
- Partitioning considerations for large datasets
- Full-text search capabilities
- Efficient movement tracking

### 5. Data Integrity
- Foreign key constraints
- Unique constraints for identifiers
- Check constraints for business rules
- Audit trail capabilities

This schema provides the foundation for a world-class inventory management system supporting all requirements including IMPA codes, multi-warehouse operations, expiry tracking, and serial number management.