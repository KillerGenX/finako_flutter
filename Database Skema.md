-- FINAKO COMPREHENSIVE DATABASE SCHEMA
-- Platform: Supabase (PostgreSQL)
-- Version: 1.0
-- =============================================================================

-- Enable necessary extensions
CREATE EXTENSION IF NOT EXISTS "uuid-ossp";
CREATE EXTENSION IF NOT EXISTS "pgcrypto";

-- =============================================================================
-- 1. AUTHENTICATION & USER MANAGEMENT
-- =============================================================================

-- Users table (extends Supabase auth.users)
CREATE TABLE public.users (
    id UUID REFERENCES auth.users(id) PRIMARY KEY,
    email VARCHAR(255) UNIQUE NOT NULL,
    full_name VARCHAR(255) NOT NULL,
    phone VARCHAR(20),
    avatar_url TEXT,
    preferred_language VARCHAR(10) DEFAULT 'id',
    timezone VARCHAR(50) DEFAULT 'Asia/Jakarta',
    is_active BOOLEAN DEFAULT true,
    last_login_at TIMESTAMPTZ,
    created_at TIMESTAMPTZ DEFAULT NOW(),
    updated_at TIMESTAMPTZ DEFAULT NOW()
);

-- =============================================================================
-- 2. SUBSCRIPTION & BILLING
-- =============================================================================

-- Subscription plans
CREATE TABLE public.subscription_plans (
    id UUID DEFAULT uuid_generate_v4() PRIMARY KEY,
    name VARCHAR(100) NOT NULL,
    description TEXT,
    price DECIMAL(12,2) NOT NULL,
    currency VARCHAR(3) DEFAULT 'IDR',
    billing_period VARCHAR(20) NOT NULL, -- monthly, yearly, lifetime
    max_businesses INTEGER,
    max_users_per_business INTEGER,
    max_products_per_business INTEGER,
    has_ai_features BOOLEAN DEFAULT false,
    has_multi_warehouse BOOLEAN DEFAULT false,
    has_advanced_reports BOOLEAN DEFAULT false,
    features JSONB, -- flexible features storage
    is_active BOOLEAN DEFAULT true,
    created_at TIMESTAMPTZ DEFAULT NOW(),
    updated_at TIMESTAMPTZ DEFAULT NOW()
);

-- User subscriptions
CREATE TABLE public.user_subscriptions (
    id UUID DEFAULT uuid_generate_v4() PRIMARY KEY,
    user_id UUID REFERENCES users(id) ON DELETE CASCADE,
    plan_id UUID REFERENCES subscription_plans(id),
    status VARCHAR(20) NOT NULL, -- active, cancelled, expired, pending
    starts_at TIMESTAMPTZ NOT NULL,
    ends_at TIMESTAMPTZ,
    payment_method VARCHAR(50),
    payment_reference VARCHAR(255),
    auto_renew BOOLEAN DEFAULT true,
    created_at TIMESTAMPTZ DEFAULT NOW(),
    updated_at TIMESTAMPTZ DEFAULT NOW()
);

-- Payment transactions
CREATE TABLE public.payment_transactions (
    id UUID DEFAULT uuid_generate_v4() PRIMARY KEY,
    user_id UUID REFERENCES users(id),
    subscription_id UUID REFERENCES user_subscriptions(id),
    amount DECIMAL(12,2) NOT NULL,
    currency VARCHAR(3) DEFAULT 'IDR',
    payment_method VARCHAR(50) NOT NULL,
    payment_provider VARCHAR(50), -- midtrans, stripe
    provider_transaction_id VARCHAR(255),
    status VARCHAR(20) NOT NULL, -- pending, success, failed, cancelled
    payment_data JSONB, -- store provider response
    created_at TIMESTAMPTZ DEFAULT NOW(),
    updated_at TIMESTAMPTZ DEFAULT NOW()
);

-- =============================================================================
-- 3. BUSINESS MANAGEMENT
-- =============================================================================

-- Businesses
CREATE TABLE public.businesses (
    id UUID DEFAULT uuid_generate_v4() PRIMARY KEY,
    owner_id UUID REFERENCES users(id) ON DELETE CASCADE,
    subscription_id UUID REFERENCES user_subscriptions(id),
    name VARCHAR(255) NOT NULL,
    description TEXT,
    business_type VARCHAR(100), -- retail, restaurant, service, etc
    logo_url TEXT,
    address TEXT,
    city VARCHAR(100),
    province VARCHAR(100),
    postal_code VARCHAR(10),
    country VARCHAR(100) DEFAULT 'Indonesia',
    phone VARCHAR(20),
    email VARCHAR(255),
    website VARCHAR(255),
    tax_number VARCHAR(50), -- NPWP
    currency VARCHAR(3) DEFAULT 'IDR',
    timezone VARCHAR(50) DEFAULT 'Asia/Jakarta',
    business_hours JSONB, -- flexible schedule storage
    is_active BOOLEAN DEFAULT true,
    created_at TIMESTAMPTZ DEFAULT NOW(),
    updated_at TIMESTAMPTZ DEFAULT NOW()
);

-- Business settings
CREATE TABLE public.business_settings (
    id UUID DEFAULT uuid_generate_v4() PRIMARY KEY,
    business_id UUID REFERENCES businesses(id) ON DELETE CASCADE,
    setting_key VARCHAR(100) NOT NULL,
    setting_value JSONB,
    created_at TIMESTAMPTZ DEFAULT NOW(),
    updated_at TIMESTAMPTZ DEFAULT NOW(),
    UNIQUE(business_id, setting_key)
);

-- =============================================================================
-- 4. ROLES & PERMISSIONS
-- =============================================================================

-- Role templates
CREATE TABLE public.role_templates (
    id UUID DEFAULT uuid_generate_v4() PRIMARY KEY,
    name VARCHAR(100) NOT NULL,
    description TEXT,
    permissions JSONB NOT NULL, -- array of permissions
    is_system_role BOOLEAN DEFAULT false,
    created_at TIMESTAMPTZ DEFAULT NOW(),
    updated_at TIMESTAMPTZ DEFAULT NOW()
);

-- Business users (employees)
CREATE TABLE public.business_users (
    id UUID DEFAULT uuid_generate_v4() PRIMARY KEY,
    business_id UUID REFERENCES businesses(id) ON DELETE CASCADE,
    user_id UUID REFERENCES users(id) ON DELETE CASCADE,
    role_template_id UUID REFERENCES role_templates(id),
    custom_permissions JSONB, -- override template permissions
    employee_id VARCHAR(50), -- internal employee ID
    position VARCHAR(100),
    department VARCHAR(100),
    salary DECIMAL(12,2),
    hire_date DATE,
    status VARCHAR(20) DEFAULT 'active', -- active, inactive, terminated
    notes TEXT,
    created_at TIMESTAMPTZ DEFAULT NOW(),
    updated_at TIMESTAMPTZ DEFAULT NOW(),
    UNIQUE(business_id, user_id)
);

-- =============================================================================
-- 5. MASTER DATA
-- =============================================================================

-- Categories
CREATE TABLE public.categories (
    id UUID DEFAULT uuid_generate_v4() PRIMARY KEY,
    business_id UUID REFERENCES businesses(id) ON DELETE CASCADE,
    parent_id UUID REFERENCES categories(id),
    name VARCHAR(255) NOT NULL,
    description TEXT,
    image_url TEXT,
    sort_order INTEGER DEFAULT 0,
    is_active BOOLEAN DEFAULT true,
    created_at TIMESTAMPTZ DEFAULT NOW(),
    updated_at TIMESTAMPTZ DEFAULT NOW()
);

-- Products
CREATE TABLE public.products (
    id UUID DEFAULT uuid_generate_v4() PRIMARY KEY,
    business_id UUID REFERENCES businesses(id) ON DELETE CASCADE,
    category_id UUID REFERENCES categories(id),
    sku VARCHAR(100),
    barcode VARCHAR(100),
    name VARCHAR(255) NOT NULL,
    description TEXT,
    unit VARCHAR(50) NOT NULL, -- pcs, kg, liter, etc
    cost_price DECIMAL(12,2),
    selling_price DECIMAL(12,2) NOT NULL,
    min_stock INTEGER DEFAULT 0,
    max_stock INTEGER,
    current_stock INTEGER DEFAULT 0,
    track_stock BOOLEAN DEFAULT true,
    images JSONB, -- array of image URLs
    tax_rate DECIMAL(5,2) DEFAULT 0,
    is_active BOOLEAN DEFAULT true,
    created_at TIMESTAMPTZ DEFAULT NOW(),
    updated_at TIMESTAMPTZ DEFAULT NOW(),
    UNIQUE(business_id, sku)
);

-- Warehouses
CREATE TABLE public.warehouses (
    id UUID DEFAULT uuid_generate_v4() PRIMARY KEY,
    business_id UUID REFERENCES businesses(id) ON DELETE CASCADE,
    name VARCHAR(255) NOT NULL,
    address TEXT,
    phone VARCHAR(20),
    manager_id UUID REFERENCES business_users(id),
    is_main BOOLEAN DEFAULT false,
    is_active BOOLEAN DEFAULT true,
    created_at TIMESTAMPTZ DEFAULT NOW(),
    updated_at TIMESTAMPTZ DEFAULT NOW()
);

-- Outlets (separate from warehouses - Zahir approach)
CREATE TABLE public.outlets (
    id UUID DEFAULT uuid_generate_v4() PRIMARY KEY,
    business_id UUID REFERENCES businesses(id) ON DELETE CASCADE,
    warehouse_id UUID REFERENCES warehouses(id), -- main source warehouse
    name VARCHAR(255) NOT NULL,
    type VARCHAR(20) DEFAULT 'outlet', -- outlet, warehouse, both
    address TEXT,
    phone VARCHAR(20),
    manager_id UUID REFERENCES business_users(id),
    is_main BOOLEAN DEFAULT false,
    is_active BOOLEAN DEFAULT true,
    pos_settings JSONB, -- POS specific settings
    created_at TIMESTAMPTZ DEFAULT NOW(),
    updated_at TIMESTAMPTZ DEFAULT NOW()
);

-- Product warehouse stock
CREATE TABLE public.product_stocks (
    id UUID DEFAULT uuid_generate_v4() PRIMARY KEY,
    product_id UUID REFERENCES products(id) ON DELETE CASCADE,
    warehouse_id UUID REFERENCES warehouses(id) ON DELETE CASCADE,
    quantity INTEGER DEFAULT 0,
    reserved_quantity INTEGER DEFAULT 0, -- for pending orders
    last_updated TIMESTAMPTZ DEFAULT NOW(),
    UNIQUE(product_id, warehouse_id)
);

-- Product variants (for laptop with different RAM, coffee hot/cold, etc)
CREATE TABLE public.product_variants (
    id UUID DEFAULT uuid_generate_v4() PRIMARY KEY,
    product_id UUID REFERENCES products(id) ON DELETE CASCADE,
    variant_name VARCHAR(255) NOT NULL, -- "RAM 8GB", "Hot", "Size L"
    variant_value VARCHAR(255) NOT NULL, -- "8GB", "Hot", "Large"
    sku_suffix VARCHAR(50), -- additional SKU identifier
    price_adjustment DECIMAL(12,2) DEFAULT 0, -- price difference from base product
    cost_adjustment DECIMAL(12,2) DEFAULT 0, -- cost difference from base product
    is_active BOOLEAN DEFAULT true,
    created_at TIMESTAMPTZ DEFAULT NOW(),
    updated_at TIMESTAMPTZ DEFAULT NOW()
);

-- Product recipes (for assembly/composite products)
CREATE TABLE public.product_recipes (
    id UUID DEFAULT uuid_generate_v4() PRIMARY KEY,
    product_id UUID REFERENCES products(id) ON DELETE CASCADE, -- finished product
    variant_id UUID REFERENCES product_variants(id), -- specific variant recipe
    name VARCHAR(255) NOT NULL,
    description TEXT,
    yield_quantity DECIMAL(10,2) DEFAULT 1, -- how many units this recipe produces
    cost_per_unit DECIMAL(12,2),
    preparation_time INTEGER, -- minutes
    is_active BOOLEAN DEFAULT true,
    created_at TIMESTAMPTZ DEFAULT NOW(),
    updated_at TIMESTAMPTZ DEFAULT NOW()
);

-- Recipe items (components/ingredients)
CREATE TABLE public.recipe_items (
    id UUID DEFAULT uuid_generate_v4() PRIMARY KEY,
    recipe_id UUID REFERENCES product_recipes(id) ON DELETE CASCADE,
    component_product_id UUID REFERENCES products(id), -- component/ingredient
    component_variant_id UUID REFERENCES product_variants(id), -- specific variant
    quantity_needed DECIMAL(10,2) NOT NULL,
    unit VARCHAR(50) NOT NULL,
    cost_per_unit DECIMAL(12,2),
    notes TEXT,
    sort_order INTEGER DEFAULT 0,
    created_at TIMESTAMPTZ DEFAULT NOW()
);

-- Outlet stocks (separate from warehouse stocks)
CREATE TABLE public.outlet_stocks (
    id UUID DEFAULT uuid_generate_v4() PRIMARY KEY,
    product_id UUID REFERENCES products(id) ON DELETE CASCADE,
    variant_id UUID REFERENCES product_variants(id), -- for variant-specific stock
    outlet_id UUID REFERENCES outlets(id) ON DELETE CASCADE,
    quantity DECIMAL(10,2) DEFAULT 0,
    reserved_quantity DECIMAL(10,2) DEFAULT 0, -- for pending sales
    min_stock DECIMAL(10,2) DEFAULT 0, -- outlet-specific minimum
    max_stock DECIMAL(10,2), -- outlet-specific maximum
    last_updated TIMESTAMPTZ DEFAULT NOW(),
    UNIQUE(product_id, outlet_id, variant_id)
);

-- Suppliers
CREATE TABLE public.suppliers (
    id UUID DEFAULT uuid_generate_v4() PRIMARY KEY,
    business_id UUID REFERENCES businesses(id) ON DELETE CASCADE,
    name VARCHAR(255) NOT NULL,
    contact_person VARCHAR(255),
    email VARCHAR(255),
    phone VARCHAR(20),
    address TEXT,
    city VARCHAR(100),
    province VARCHAR(100),
    postal_code VARCHAR(10),
    tax_number VARCHAR(50),
    payment_terms INTEGER DEFAULT 0, -- days
    notes TEXT,
    is_active BOOLEAN DEFAULT true,
    created_at TIMESTAMPTZ DEFAULT NOW(),
    updated_at TIMESTAMPTZ DEFAULT NOW()
);

-- Customers
CREATE TABLE public.customers (
    id UUID DEFAULT uuid_generate_v4() PRIMARY KEY,
    business_id UUID REFERENCES businesses(id) ON DELETE CASCADE,
    customer_code VARCHAR(50),
    name VARCHAR(255) NOT NULL,
    email VARCHAR(255),
    phone VARCHAR(20),
    address TEXT,
    city VARCHAR(100),
    province VARCHAR(100),
    postal_code VARCHAR(10),
    date_of_birth DATE,
    gender VARCHAR(10),
    customer_type VARCHAR(50) DEFAULT 'retail', -- retail, wholesale, vip
    credit_limit DECIMAL(12,2) DEFAULT 0,
    notes TEXT,
    is_active BOOLEAN DEFAULT true,
    created_at TIMESTAMPTZ DEFAULT NOW(),
    updated_at TIMESTAMPTZ DEFAULT NOW(),
    UNIQUE(business_id, customer_code)
);

-- =============================================================================
-- 6. DOCUMENT MANAGEMENT
-- =============================================================================

-- Document types
CREATE TABLE public.document_types (
    id UUID DEFAULT uuid_generate_v4() PRIMARY KEY,
    business_id UUID REFERENCES businesses(id) ON DELETE CASCADE,
    code VARCHAR(20) NOT NULL, -- PO, SO, PI, DO, INV, etc
    name VARCHAR(100) NOT NULL,
    prefix VARCHAR(10),
    numbering_format VARCHAR(50), -- {prefix}{YYYY}{MM}{###}
    next_number INTEGER DEFAULT 1,
    is_active BOOLEAN DEFAULT true,
    created_at TIMESTAMPTZ DEFAULT NOW(),
    updated_at TIMESTAMPTZ DEFAULT NOW(),
    UNIQUE(business_id, code)
);

-- Documents (unified table for all document types)
CREATE TABLE public.documents (
    id UUID DEFAULT uuid_generate_v4() PRIMARY KEY,
    business_id UUID REFERENCES businesses(id) ON DELETE CASCADE,
    document_type_id UUID REFERENCES document_types(id),
    warehouse_id UUID REFERENCES warehouses(id),
    document_number VARCHAR(100) NOT NULL,
    reference_number VARCHAR(100),
    parent_document_id UUID REFERENCES documents(id), -- for related documents
    supplier_id UUID REFERENCES suppliers(id),
    customer_id UUID REFERENCES customers(id),
    created_by UUID REFERENCES business_users(id),
    approved_by UUID REFERENCES business_users(id),
    document_date DATE NOT NULL,
    due_date DATE,
    status VARCHAR(20) NOT NULL, -- draft, pending, approved, cancelled
    subtotal DECIMAL(12,2) DEFAULT 0,
    tax_amount DECIMAL(12,2) DEFAULT 0,
    discount_amount DECIMAL(12,2) DEFAULT 0,
    total_amount DECIMAL(12,2) DEFAULT 0,
    notes TEXT,
    attachments JSONB, -- array of file URLs
    metadata JSONB, -- flexible data storage
    created_at TIMESTAMPTZ DEFAULT NOW(),
    updated_at TIMESTAMPTZ DEFAULT NOW(),
    UNIQUE(business_id, document_number)
);

-- Document items
CREATE TABLE public.document_items (
    id UUID DEFAULT uuid_generate_v4() PRIMARY KEY,
    document_id UUID REFERENCES documents(id) ON DELETE CASCADE,
    product_id UUID REFERENCES products(id),
    quantity DECIMAL(10,2) NOT NULL,
    unit_price DECIMAL(12,2) NOT NULL,
    discount_percentage DECIMAL(5,2) DEFAULT 0,
    discount_amount DECIMAL(12,2) DEFAULT 0,
    tax_rate DECIMAL(5,2) DEFAULT 0,
    line_total DECIMAL(12,2) NOT NULL,
    notes TEXT,
    created_at TIMESTAMPTZ DEFAULT NOW()
);

-- =============================================================================
-- 7. TRANSACTIONS & PAYMENTS
-- =============================================================================

-- Transaction types
CREATE TABLE public.transaction_types (
    id UUID DEFAULT uuid_generate_v4() PRIMARY KEY,
    business_id UUID REFERENCES businesses(id) ON DELETE CASCADE,
    name VARCHAR(100) NOT NULL,
    type VARCHAR(20) NOT NULL, -- sales, purchase, expense, income
    affects_stock BOOLEAN DEFAULT false,
    is_active BOOLEAN DEFAULT true,
    created_at TIMESTAMPTZ DEFAULT NOW()
);

-- Transactions (unified for all business transactions)
CREATE TABLE public.transactions (
    id UUID DEFAULT uuid_generate_v4() PRIMARY KEY,
    business_id UUID REFERENCES businesses(id) ON DELETE CASCADE,
    transaction_type_id UUID REFERENCES transaction_types(id),
    document_id UUID REFERENCES documents(id),
    customer_id UUID REFERENCES customers(id),
    supplier_id UUID REFERENCES suppliers(id),
    warehouse_id UUID REFERENCES warehouses(id),
    outlet_id UUID REFERENCES outlets(id), -- for POS transactions
    cashier_id UUID REFERENCES business_users(id),
    transaction_number VARCHAR(100) NOT NULL,
    transaction_date TIMESTAMPTZ NOT NULL,
    subtotal DECIMAL(12,2) DEFAULT 0,
    tax_amount DECIMAL(12,2) DEFAULT 0,
    discount_amount DECIMAL(12,2) DEFAULT 0,
    total_amount DECIMAL(12,2) DEFAULT 0,
    paid_amount DECIMAL(12,2) DEFAULT 0,
    change_amount DECIMAL(12,2) DEFAULT 0,
    status VARCHAR(20) NOT NULL, -- pending, completed, cancelled, refunded
    payment_status VARCHAR(20) DEFAULT 'unpaid', -- unpaid, partial, paid
    notes TEXT,
    metadata JSONB,
    created_at TIMESTAMPTZ DEFAULT NOW(),
    updated_at TIMESTAMPTZ DEFAULT NOW(),
    UNIQUE(business_id, transaction_number)
);

-- Transaction items
CREATE TABLE public.transaction_items (
    id UUID DEFAULT uuid_generate_v4() PRIMARY KEY,
    transaction_id UUID REFERENCES transactions(id) ON DELETE CASCADE,
    product_id UUID REFERENCES products(id),
    variant_id UUID REFERENCES product_variants(id), -- for variant-specific sales
    recipe_id UUID REFERENCES product_recipes(id), -- for assembled products
    quantity DECIMAL(10,2) NOT NULL,
    unit_price DECIMAL(12,2) NOT NULL,
    discount_percentage DECIMAL(5,2) DEFAULT 0,
    discount_amount DECIMAL(12,2) DEFAULT 0,
    tax_rate DECIMAL(5,2) DEFAULT 0,
    line_total DECIMAL(12,2) NOT NULL,
    notes TEXT
);

-- Payment methods
CREATE TABLE public.payment_methods (
    id UUID DEFAULT uuid_generate_v4() PRIMARY KEY,
    business_id UUID REFERENCES businesses(id) ON DELETE CASCADE,
    name VARCHAR(100) NOT NULL,
    type VARCHAR(20) NOT NULL, -- cash, card, transfer, qris, ewallet
    account_number VARCHAR(100),
    provider VARCHAR(50), -- midtrans, stripe, manual
    is_active BOOLEAN DEFAULT true,
    created_at TIMESTAMPTZ DEFAULT NOW()
);

-- Transaction payments
CREATE TABLE public.transaction_payments (
    id UUID DEFAULT uuid_generate_v4() PRIMARY KEY,
    transaction_id UUID REFERENCES transactions(id) ON DELETE CASCADE,
    payment_method_id UUID REFERENCES payment_methods(id),
    amount DECIMAL(12,2) NOT NULL,
    reference_number VARCHAR(255),
    payment_date TIMESTAMPTZ DEFAULT NOW(),
    status VARCHAR(20) DEFAULT 'completed', -- pending, completed, failed
    provider_response JSONB,
    notes TEXT
);

-- =============================================================================
-- 8. INVENTORY MANAGEMENT
-- =============================================================================

-- Stock movements
CREATE TABLE public.stock_movements (
    id UUID DEFAULT uuid_generate_v4() PRIMARY KEY,
    product_id UUID REFERENCES products(id) ON DELETE CASCADE,
    variant_id UUID REFERENCES product_variants(id), -- for variant-specific movements
    warehouse_id UUID REFERENCES warehouses(id) ON DELETE CASCADE,
    outlet_id UUID REFERENCES outlets(id), -- for outlet stock movements
    transaction_id UUID REFERENCES transactions(id),
    document_id UUID REFERENCES documents(id),
    movement_type VARCHAR(20) NOT NULL, -- in, out, adjustment, transfer, assembly
    quantity DECIMAL(10,2) NOT NULL, -- positive for in, negative for out
    unit_cost DECIMAL(12,2),
    reference_number VARCHAR(255),
    notes TEXT,
    created_by UUID REFERENCES business_users(id),
    created_at TIMESTAMPTZ DEFAULT NOW()
);

-- Stock requests (outlet requests to warehouse)
CREATE TABLE public.stock_requests (
    id UUID DEFAULT uuid_generate_v4() PRIMARY KEY,
    business_id UUID REFERENCES businesses(id) ON DELETE CASCADE,
    outlet_id UUID REFERENCES outlets(id) ON DELETE CASCADE,
    warehouse_id UUID REFERENCES warehouses(id) ON DELETE CASCADE,
    request_number VARCHAR(100) NOT NULL,
    requested_by UUID REFERENCES business_users(id),
    approved_by UUID REFERENCES business_users(id),
    request_date DATE NOT NULL,
    required_date DATE,
    status VARCHAR(20) DEFAULT 'pending', -- pending, approved, rejected, completed
    priority VARCHAR(20) DEFAULT 'normal', -- low, normal, high, urgent
    notes TEXT,
    created_at TIMESTAMPTZ DEFAULT NOW(),
    updated_at TIMESTAMPTZ DEFAULT NOW(),
    UNIQUE(business_id, request_number)
);

-- Stock request items
CREATE TABLE public.stock_request_items (
    id UUID DEFAULT uuid_generate_v4() PRIMARY KEY,
    request_id UUID REFERENCES stock_requests(id) ON DELETE CASCADE,
    product_id UUID REFERENCES products(id),
    variant_id UUID REFERENCES product_variants(id),
    requested_quantity DECIMAL(10,2) NOT NULL,
    approved_quantity DECIMAL(10,2) DEFAULT 0,
    current_outlet_stock DECIMAL(10,2) DEFAULT 0,
    reason TEXT,
    notes TEXT
);

-- Stock transfers (warehouse to outlet transfers)
CREATE TABLE public.stock_transfers (
    id UUID DEFAULT uuid_generate_v4() PRIMARY KEY,
    business_id UUID REFERENCES businesses(id) ON DELETE CASCADE,
    request_id UUID REFERENCES stock_requests(id), -- linked to request
    transfer_number VARCHAR(100) NOT NULL,
    from_warehouse_id UUID REFERENCES warehouses(id),
    from_outlet_id UUID REFERENCES outlets(id),
    to_warehouse_id UUID REFERENCES warehouses(id),
    to_outlet_id UUID REFERENCES outlets(id),
    transfer_date DATE NOT NULL,
    shipped_by UUID REFERENCES business_users(id),
    received_by UUID REFERENCES business_users(id),
    status VARCHAR(20) DEFAULT 'pending', -- pending, shipped, received, cancelled
    transfer_type VARCHAR(20) NOT NULL, -- request_fulfillment, inter_outlet, emergency
    notes TEXT,
    created_at TIMESTAMPTZ DEFAULT NOW(),
    updated_at TIMESTAMPTZ DEFAULT NOW(),
    UNIQUE(business_id, transfer_number)
);

-- Stock transfer items
CREATE TABLE public.stock_transfer_items (
    id UUID DEFAULT uuid_generate_v4() PRIMARY KEY,
    transfer_id UUID REFERENCES stock_transfers(id) ON DELETE CASCADE,
    product_id UUID REFERENCES products(id),
    variant_id UUID REFERENCES product_variants(id),
    shipped_quantity DECIMAL(10,2) NOT NULL,
    received_quantity DECIMAL(10,2) DEFAULT 0,
    unit_cost DECIMAL(12,2),
    notes TEXT
);

-- Stock adjustments
CREATE TABLE public.stock_adjustments (
    id UUID DEFAULT uuid_generate_v4() PRIMARY KEY,
    business_id UUID REFERENCES businesses(id) ON DELETE CASCADE,
    warehouse_id UUID REFERENCES warehouses(id),
    adjustment_number VARCHAR(100) NOT NULL,
    adjustment_date DATE NOT NULL,
    reason VARCHAR(255) NOT NULL,
    created_by UUID REFERENCES business_users(id),
    approved_by UUID REFERENCES business_users(id),
    status VARCHAR(20) DEFAULT 'draft', -- draft, approved, cancelled
    notes TEXT,
    created_at TIMESTAMPTZ DEFAULT NOW(),
    UNIQUE(business_id, adjustment_number)
);

-- Stock adjustment items
CREATE TABLE public.stock_adjustment_items (
    id UUID DEFAULT uuid_generate_v4() PRIMARY KEY,
    adjustment_id UUID REFERENCES stock_adjustments(id) ON DELETE CASCADE,
    product_id UUID REFERENCES products(id),
    current_stock DECIMAL(10,2) NOT NULL,
    adjusted_stock DECIMAL(10,2) NOT NULL,
    difference DECIMAL(10,2) NOT NULL,
    reason TEXT,
    unit_cost DECIMAL(12,2)
);

-- =============================================================================
-- 9. REPORTING & ANALYTICS
-- =============================================================================

-- Report templates
CREATE TABLE public.report_templates (
    id UUID DEFAULT uuid_generate_v4() PRIMARY KEY,
    business_id UUID REFERENCES businesses(id) ON DELETE CASCADE,
    name VARCHAR(255) NOT NULL,
    description TEXT,
    report_type VARCHAR(50) NOT NULL, -- sales, inventory, financial, etc
    query_template TEXT, -- SQL template or config
    parameters JSONB, -- report parameters
    schedule JSONB, -- auto-generation schedule
    created_by UUID REFERENCES business_users(id),
    is_active BOOLEAN DEFAULT true,
    created_at TIMESTAMPTZ DEFAULT NOW(),
    updated_at TIMESTAMPTZ DEFAULT NOW()
);

-- Generated reports
CREATE TABLE public.generated_reports (
    id UUID DEFAULT uuid_generate_v4() PRIMARY KEY,
    template_id UUID REFERENCES report_templates(id),
    business_id UUID REFERENCES businesses(id) ON DELETE CASCADE,
    name VARCHAR(255) NOT NULL,
    period_start DATE,
    period_end DATE,
    format VARCHAR(10) NOT NULL, -- pdf, excel, json
    file_url TEXT,
    file_size INTEGER,
    generated_by UUID REFERENCES business_users(id),
    status VARCHAR(20) DEFAULT 'generating', -- generating, completed, failed
    error_message TEXT,
    created_at TIMESTAMPTZ DEFAULT NOW()
);

-- =============================================================================
-- 10. AI & AUTOMATION
-- =============================================================================

-- AI models configuration
CREATE TABLE public.ai_models (
    id UUID DEFAULT uuid_generate_v4() PRIMARY KEY,
    name VARCHAR(100) NOT NULL,
    provider VARCHAR(50) NOT NULL, -- openai, google, etc
    model_name VARCHAR(100) NOT NULL,
    api_endpoint TEXT,
    capabilities JSONB, -- what this model can do
    cost_per_request DECIMAL(10,6),
    is_active BOOLEAN DEFAULT true,
    created_at TIMESTAMPTZ DEFAULT NOW()
);

-- AI requests log
CREATE TABLE public.ai_requests (
    id UUID DEFAULT uuid_generate_v4() PRIMARY KEY,
    business_id UUID REFERENCES businesses(id) ON DELETE CASCADE,
    user_id UUID REFERENCES users(id),
    model_id UUID REFERENCES ai_models(id),
    request_type VARCHAR(50) NOT NULL, -- analysis, prediction, recommendation
    input_data JSONB NOT NULL,
    output_data JSONB,
    tokens_used INTEGER,
    cost DECIMAL(10,6),
    processing_time INTEGER, -- milliseconds
    status VARCHAR(20) DEFAULT 'completed', -- processing, completed, failed
    error_message TEXT,
    created_at TIMESTAMPTZ DEFAULT NOW()
);

-- AI insights
CREATE TABLE public.ai_insights (
    id UUID DEFAULT uuid_generate_v4() PRIMARY KEY,
    business_id UUID REFERENCES businesses(id) ON DELETE CASCADE,
    insight_type VARCHAR(50) NOT NULL, -- inventory, sales, financial, etc
    title VARCHAR(255) NOT NULL,
    description TEXT,
    data JSONB, -- structured insight data
    priority VARCHAR(20) DEFAULT 'medium', -- low, medium, high, critical
    action_required BOOLEAN DEFAULT false,
    is_read BOOLEAN DEFAULT false,
    expires_at TIMESTAMPTZ,
    created_at TIMESTAMPTZ DEFAULT NOW()
);

-- =============================================================================
-- 11. NOTIFICATIONS & COMMUNICATIONS
-- =============================================================================

-- Notification templates
CREATE TABLE public.notification_templates (
    id UUID DEFAULT uuid_generate_v4() PRIMARY KEY,
    name VARCHAR(100) NOT NULL,
    type VARCHAR(50) NOT NULL, -- email, push, sms, in_app
    event_trigger VARCHAR(100) NOT NULL, -- low_stock, payment_due, etc
    subject_template TEXT,
    body_template TEXT,
    variables JSONB, -- available template variables
    is_active BOOLEAN DEFAULT true,
    created_at TIMESTAMPTZ DEFAULT NOW()
);

-- Notifications
CREATE TABLE public.notifications (
    id UUID DEFAULT uuid_generate_v4() PRIMARY KEY,
    business_id UUID REFERENCES businesses(id) ON DELETE CASCADE,
    user_id UUID REFERENCES users(id),
    template_id UUID REFERENCES notification_templates(id),
    type VARCHAR(50) NOT NULL,
    title VARCHAR(255) NOT NULL,
    message TEXT NOT NULL,
    data JSONB, -- additional notification data
    is_read BOOLEAN DEFAULT false,
    sent_at TIMESTAMPTZ,
    read_at TIMESTAMPTZ,
    expires_at TIMESTAMPTZ,
    created_at TIMESTAMPTZ DEFAULT NOW()
);

-- =============================================================================
-- 12. AUDIT & LOGGING
-- =============================================================================

-- Audit logs
CREATE TABLE public.audit_logs (
    id UUID DEFAULT uuid_generate_v4() PRIMARY KEY,
    business_id UUID REFERENCES businesses(id) ON DELETE CASCADE,
    user_id UUID REFERENCES users(id),
    table_name VARCHAR(100) NOT NULL,
    record_id UUID NOT NULL,
    action VARCHAR(20) NOT NULL, -- insert, update, delete
    old_values JSONB,
    new_values JSONB,
    ip_address INET,
    user_agent TEXT,
    created_at TIMESTAMPTZ DEFAULT NOW()
);

-- System logs
CREATE TABLE public.system_logs (
    id UUID DEFAULT uuid_generate_v4() PRIMARY KEY,
    level VARCHAR(20) NOT NULL, -- info, warning, error, critical
    source VARCHAR(100) NOT NULL, -- api, sync, ai, payment, etc
    message TEXT NOT NULL,
    data JSONB,
    user_id UUID REFERENCES users(id),
    business_id UUID REFERENCES businesses(id),
    created_at TIMESTAMPTZ DEFAULT NOW()
);

-- =============================================================================
-- 13. OFFLINE SYNC
-- =============================================================================

-- Sync queue
CREATE TABLE public.sync_queue (
    id UUID DEFAULT uuid_generate_v4() PRIMARY KEY,
    user_id UUID REFERENCES users(id),
    business_id UUID REFERENCES businesses(id),
    table_name VARCHAR(100) NOT NULL,
    record_id UUID NOT NULL,
    action VARCHAR(20) NOT NULL, -- insert, update, delete
    data JSONB NOT NULL,
    device_id VARCHAR(255),
    sync_status VARCHAR(20) DEFAULT 'pending', -- pending, processing, completed, failed
    retry_count INTEGER DEFAULT 0,
    error_message TEXT,
    created_at TIMESTAMPTZ DEFAULT NOW(),
    processed_at TIMESTAMPTZ
);

-- Device sync status
CREATE TABLE public.device_sync_status (
    id UUID DEFAULT uuid_generate_v4() PRIMARY KEY,
    user_id UUID REFERENCES users(id),
    business_id UUID REFERENCES businesses(id),
    device_id VARCHAR(255) NOT NULL,
    device_name VARCHAR(255),
    last_sync_at TIMESTAMPTZ,
    sync_version INTEGER DEFAULT 0,
    is_active BOOLEAN DEFAULT true,
    created_at TIMESTAMPTZ DEFAULT NOW(),
    updated_at TIMESTAMPTZ DEFAULT NOW(),
    UNIQUE(user_id, business_id, device_id)
);

-- =============================================================================
-- INDEXES FOR PERFORMANCE
-- =============================================================================

-- Users
CREATE INDEX idx_users_email ON users(email);
CREATE INDEX idx_users_active ON users(is_active);

-- Businesses
CREATE INDEX idx_businesses_owner ON businesses(owner_id);
CREATE INDEX idx_businesses_active ON businesses(is_active);

-- Business users
CREATE INDEX idx_business_users_business ON business_users(business_id);
CREATE INDEX idx_business_users_user ON business_users(user_id);
CREATE INDEX idx_business_users_status ON business_users(status);

-- Outlets
CREATE INDEX idx_outlets_business ON outlets(business_id);
CREATE INDEX idx_outlets_warehouse ON outlets(warehouse_id);
CREATE INDEX idx_outlets_type ON outlets(type);
CREATE INDEX idx_outlets_active ON outlets(is_active);

-- Product variants
CREATE INDEX idx_product_variants_product ON product_variants(product_id);
CREATE INDEX idx_product_variants_active ON product_variants(is_active);

-- Product recipes
CREATE INDEX idx_product_recipes_product ON product_recipes(product_id);
CREATE INDEX idx_product_recipes_variant ON product_recipes(variant_id);

-- Recipe items
CREATE INDEX idx_recipe_items_recipe ON recipe_items(recipe_id);
CREATE INDEX idx_recipe_items_component ON recipe_items(component_product_id);

-- Outlet stocks
CREATE INDEX idx_outlet_stocks_outlet ON outlet_stocks(outlet_id);
CREATE INDEX idx_outlet_stocks_product ON outlet_stocks(product_id);
CREATE INDEX idx_outlet_stocks_variant ON outlet_stocks(variant_id);

-- Stock requests
CREATE INDEX idx_stock_requests_outlet ON stock_requests(outlet_id);
CREATE INDEX idx_stock_requests_warehouse ON stock_requests(warehouse_id);
CREATE INDEX idx_stock_requests_status ON stock_requests(status);
CREATE INDEX idx_stock_requests_date ON stock_requests(request_date);

-- Stock transfers
CREATE INDEX idx_stock_transfers_request ON stock_transfers(request_id);
CREATE INDEX idx_stock_transfers_from_warehouse ON stock_transfers(from_warehouse_id);
CREATE INDEX idx_stock_transfers_to_outlet ON stock_transfers(to_outlet_id);
CREATE INDEX idx_stock_transfers_status ON stock_transfers(status);
CREATE INDEX idx_stock_transfers_date ON stock_transfers(transfer_date);

-- Products
CREATE INDEX idx_products_business ON products(business_id);
CREATE INDEX idx_products_category ON products(category_id);
CREATE INDEX idx_products_sku ON products(business_id, sku);
CREATE INDEX idx_products_active ON products(is_active);

-- Transactions
CREATE INDEX idx_transactions_business ON transactions(business_id);
CREATE INDEX idx_transactions_date ON transactions(transaction_date);
CREATE INDEX idx_transactions_customer ON transactions(customer_id);
CREATE INDEX idx_transactions_status ON transactions(status);

-- Documents
CREATE INDEX idx_documents_business ON documents(business_id);
CREATE INDEX idx_documents_type ON documents(document_type_id);
CREATE INDEX idx_documents_date ON documents(document_date);
CREATE INDEX idx_documents_status ON documents(status);

-- Stock movements
CREATE INDEX idx_stock_movements_product ON stock_movements(product_id);
CREATE INDEX idx_stock_movements_warehouse ON stock_movements(warehouse_id);
CREATE INDEX idx_stock_movements_date ON stock_movements(created_at);

-- Audit logs
CREATE INDEX idx_audit_logs_business ON audit_logs(business_id);
CREATE INDEX idx_audit_logs_user ON audit_logs(user_id);
CREATE INDEX idx_audit_logs_table ON audit_logs(table_name);
CREATE INDEX idx_audit_logs_date ON audit_logs(created_at);

-- Notifications
CREATE INDEX idx_notifications_user ON notifications(user_id);
CREATE INDEX idx_notifications_business ON notifications(business_id);
CREATE INDEX idx_notifications_read ON notifications(is_read);

-- =============================================================================
-- ROW LEVEL SECURITY (RLS) POLICIES
-- =============================================================================

-- Enable RLS on all tables
ALTER TABLE users ENABLE ROW LEVEL SECURITY;
ALTER TABLE businesses ENABLE ROW LEVEL SECURITY;
ALTER TABLE business_users ENABLE ROW LEVEL SECURITY;
ALTER TABLE outlets ENABLE ROW LEVEL SECURITY;
ALTER TABLE products ENABLE ROW LEVEL SECURITY;
ALTER TABLE product_variants ENABLE ROW LEVEL SECURITY;
ALTER TABLE product_recipes ENABLE ROW LEVEL SECURITY;
ALTER TABLE outlet_stocks ENABLE ROW LEVEL SECURITY;
ALTER TABLE stock_requests ENABLE ROW LEVEL SECURITY;
ALTER TABLE stock_transfers ENABLE ROW LEVEL SECURITY;
ALTER TABLE transactions ENABLE ROW LEVEL SECURITY;
ALTER TABLE documents ENABLE ROW LEVEL SECURITY;
ALTER TABLE customers ENABLE ROW LEVEL SECURITY;
ALTER TABLE suppliers ENABLE ROW LEVEL SECURITY;
-- ... enable for all other tables

-- Basic RLS policies (simplified - need detailed implementation)
CREATE POLICY "Users can view own data" ON users
    FOR ALL USING (auth.uid() = id);

CREATE POLICY "Business owners can manage their businesses" ON businesses
    FOR ALL USING (auth.uid() = owner_id);

CREATE POLICY "Business users can access their business data" ON business_users
    FOR ALL USING (
        user_id = auth.uid() OR 
        business_id IN (
            SELECT business_id FROM business_users 
            WHERE user_id = auth.uid() AND status = 'active'
        )
    );

-- =============================================================================
-- FUNCTIONS & TRIGGERS
-- =============================================================================

-- Update timestamp function
CREATE OR REPLACE FUNCTION update_updated_at_column()
RETURNS TRIGGER AS $$
BEGIN
    NEW.updated_at = NOW();
    RETURN NEW;
END;
$$ language 'plpgsql';

-- Apply update trigger to all tables with updated_at
CREATE TRIGGER update_users_updated_at BEFORE UPDATE ON users
    FOR EACH ROW EXECUTE PROCEDURE update_updated_at_column();

CREATE TRIGGER update_businesses_updated_at BEFORE UPDATE ON businesses
    FOR EACH ROW EXECUTE PROCEDURE update_updated_at_column();

-- Add triggers for other tables...

-- Stock update function (enhanced for variants and outlets)
CREATE OR REPLACE FUNCTION update_product_stock()
RETURNS TRIGGER AS $$
BEGIN
    IF TG_OP = 'INSERT' THEN
        -- Update warehouse stock if warehouse_id exists
        IF NEW.warehouse_id IS NOT NULL THEN
            UPDATE products 
            SET current_stock = current_stock + NEW.quantity 
            WHERE id = NEW.product_id;
            
            -- Update product_stocks table
            INSERT INTO product_stocks (product_id, warehouse_id, quantity)
            VALUES (NEW.product_id, NEW.warehouse_id, NEW.quantity)
            ON CONFLICT (product_id, warehouse_id)
            DO UPDATE SET 
                quantity = product_stocks.quantity + NEW.quantity,
                last_updated = NOW();
        END IF;
        
        -- Update outlet stock if outlet_id exists
        IF NEW.outlet_id IS NOT NULL THEN
            INSERT INTO outlet_stocks (product_id, variant_id, outlet_id, quantity)
            VALUES (NEW.product_id, NEW.variant_id, NEW.outlet_id, NEW.quantity)
            ON CONFLICT (product_id, outlet_id, variant_id)
            DO UPDATE SET 
                quantity = outlet_stocks.quantity + NEW.quantity,
                last_updated = NOW();
        END IF;
        
        RETURN NEW;
    ELSIF TG_OP = 'DELETE' THEN
        -- Update warehouse stock if warehouse_id exists
        IF OLD.warehouse_id IS NOT NULL THEN
            UPDATE products 
            SET current_stock = current_stock - OLD.quantity 
            WHERE id = OLD.product_id;
            
            -- Update product_stocks table
            UPDATE product_stocks 
            SET quantity = quantity - OLD.quantity,
                last_updated = NOW()
            WHERE product_id = OLD.product_id AND warehouse_id = OLD.warehouse_id;
        END IF;
        
        -- Update outlet stock if outlet_id exists
        IF OLD.outlet_id IS NOT NULL THEN
            UPDATE outlet_stocks 
            SET quantity = quantity - OLD.quantity,
                last_updated = NOW()
            WHERE product_id = OLD.product_id 
            AND outlet_id = OLD.outlet_id 
            AND (variant_id = OLD.variant_id OR (variant_id IS NULL AND OLD.variant_id IS NULL));
        END IF;
        
        RETURN OLD;
    END IF;
    RETURN NULL;
END;
$$ language 'plpgsql';

-- Recipe assembly function (auto reduce components when assembled product is sold)
CREATE OR REPLACE FUNCTION process_recipe_assembly()
RETURNS TRIGGER AS $$
DECLARE
    recipe_record RECORD;
    component_record RECORD;
BEGIN
    -- Only process on INSERT of transaction_items with recipe_id
    IF TG_OP = 'INSERT' AND NEW.recipe_id IS NOT NULL THEN
        -- Get recipe details
        SELECT * INTO recipe_record FROM product_recipes WHERE id = NEW.recipe_id;
        
        -- Loop through recipe components and reduce stock
        FOR component_record IN 
            SELECT ri.*, p.track_stock
            FROM recipe_items ri
            JOIN products p ON ri.component_product_id = p.id
            WHERE ri.recipe_id = NEW.recipe_id
        LOOP
            -- Only reduce stock for tracked products
            IF component_record.track_stock THEN
                -- Calculate total quantity needed
                DECLARE
                    total_component_needed DECIMAL(10,2);
                BEGIN
                    total_component_needed := component_record.quantity_needed * NEW.quantity;
                    
                    -- Create stock movement for component reduction
                    INSERT INTO stock_movements (
                        product_id,
                        variant_id,
                        outlet_id,
                        transaction_id,
                        movement_type,
                        quantity,
                        reference_number,
                        notes,
                        created_by
                    ) VALUES (
                        component_record.component_product_id,
                        component_record.component_variant_id,
                        (SELECT outlet_id FROM transactions WHERE id = (SELECT transaction_id FROM transaction_items WHERE id = NEW.id)),
                        (SELECT transaction_id FROM transaction_items WHERE id = NEW.id),
                        'assembly',
                        -total_component_needed, -- negative because it's consumed
                        'AUTO-ASSEMBLY-' || NEW.id,
                        'Auto component reduction for recipe: ' || recipe_record.name,
                        (SELECT cashier_id FROM transactions WHERE id = (SELECT transaction_id FROM transaction_items WHERE id = NEW.id))
                    );
                END;
            END IF;
        END LOOP;
        
        RETURN NEW;
    END IF;
    RETURN NULL;
END;
$$ language 'plpgsql';

-- Stock movement trigger
CREATE TRIGGER update_stock_on_movement 
    AFTER INSERT OR DELETE ON stock_movements
    FOR EACH ROW EXECUTE PROCEDURE update_product_stock();

-- Recipe assembly trigger
CREATE TRIGGER process_recipe_on_sale
    AFTER INSERT ON transaction_items
    FOR EACH ROW EXECUTE PROCEDURE process_recipe_assembly();

-- =============================================================================
-- INITIAL DATA SEEDING
-- =============================================================================

-- Insert default subscription plans
INSERT INTO subscription_plans (name, description, price, billing_period, max_businesses, max_users_per_business, has_ai_features) VALUES
('Finako Basic', 'Paket dasar untuk 1 bisnis', 99000, 'monthly', 1, 5, false),
('Finako AI', 'Paket lengkap dengan AI untuk 3 bisnis', 299000, 'monthly', 3, 20, true),
('Finako Enterprise', 'Paket kustom untuk enterprise', 999000, 'monthly', null, null, true);

-- Insert default role templates
INSERT INTO role_templates (name, description, permissions, is_system_role) VALUES
('Owner', 'Pemilik bisnis dengan akses penuh', '["*"]', true),
('Manager', 'Manajer dengan akses hampir penuh', '["manage_users", "view_reports", "manage_inventory", "manage_pos"]', true),
('Cashier', 'Kasir dengan akses POS', '["access_pos", "view_products", "process_sales"]', true),
('Admin', 'Admin dengan akses administratif', '["manage_data", "view_reports", "manage_inventory"]', true);

-- Insert default document types
INSERT INTO document_types (code, name, prefix, numbering_format) VALUES
('PO', 'Purchase Order', 'PO', '{prefix}/{YYYY}/{MM}/{###}'),
('SO', 'Sales Order', 'SO', '{prefix}/{YYYY}/{MM}/{###}'),
('PI', 'Proforma Invoice', 'PI', '{prefix}/{YYYY}/{MM}/{###}'),
('INV', 'Invoice', 'INV', '{prefix}/{YYYY}/{MM}/{###}'),
('DO', 'Delivery Order', 'DO', '{prefix}/{YYYY}/{MM}/{###}'),
('SR', 'Stock Request', 'SR', '{prefix}/{YYYY}/{MM}/{###}'),
('ST', 'Stock Transfer', 'ST', '{prefix}/{YYYY}/{MM}/{###}'),
('SA', 'Stock Adjustment', 'SA', '{prefix}/{YYYY}/{MM}/{###}');

-- =============================================================================
-- END OF SCHEMA
-- =============================================================================

tambahan Kelengkapan Fitur
POS & Inventori: Sudah lengkap untuk kebutuhan retail, F&B, dan distribusi (produk, varian, stok, outlet, warehouse, transfer, adjustment).
Akuntansi: Sudah ada struktur transaksi, dokumen, dan pembayaran, namun untuk akuntansi penuh (jurnal, buku besar, COA) perlu tabel tambahan.
HR/Payroll: Untuk fitur seperti Mekari Talenta, perlu modul HR (karyawan, absensi, cuti, payroll, shift, lembur).
AI & Otomasi: Sudah ada pondasi untuk AI, insight, dan automation.
Multi Bisnis & Multi User: Sudah mendukung multi bisnis, multi user, dan role/permission granular.
2. Saran Pengembangan
Modul Akuntansi: Tambahkan tabel chart_of_accounts, journal_entries, journal_lines, dan mapping transaksi ke jurnal otomatis.
Modul HR: Tambahkan tabel employees, attendance, leave_requests, payroll, shifts, dan overtime.
Integrasi API: Siapkan endpoint REST/GraphQL untuk mobile/web, serta webhook untuk integrasi eksternal.
Audit & Compliance: Audit log sudah ada, pastikan semua perubahan penting tercatat.
Performance: Index sudah banyak, namun tetap monitor query lambat di produksi.
Multi-Tenant Security: RLS sudah diterapkan, pastikan tidak ada celah akses antar bisnis.