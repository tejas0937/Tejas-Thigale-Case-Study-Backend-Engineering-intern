Link for case study - https://docs.google.com/document/d/1IVUuK1Nd3geRabr6In88Ror8v5YK7rxIBMwt7BF9GlM/edit?usp=sharing

Solution for - Inventory Management System for B2B SaaS
By Tejas Thigale - tejsthigale0937@gmail.com
Overview
You're joining a team building "StockFlow" - a B2B inventory management platform. Small businesses use it to track products across multiple warehouses and manage supplier relationships.
Time Allocation
●	Take-Home Portion: 90 minutes maximum
●	Live Discussion: 30-45 minutes (scheduled separately)
________________________________________
Part 1: Code Review & Debugging (30 minutes)
A previous intern wrote this API endpoint for adding new products. Something is wrong - the code compiles but doesn't work as expected in production.
@app.route('/api/products', methods=['POST'])
def create_product():
    data = request.json
    
    # Create new product
    product = Product(
        name=data['name'],
        sku=data['sku'],
        price=data['price'],
        warehouse_id=data['warehouse_id']
    )
    
    db.session.add(product)
    db.session.commit()
    
    # Update inventory count
    inventory = Inventory(
        product_id=product.id,
        warehouse_id=data['warehouse_id'],
        quantity=data['initial_quantity']
    )
    
    db.session.add(inventory)
    db.session.commit()
    
    return {"message": "Product created", "product_id": product.id}

Your Tasks:
1.	Identify Issues: List all problems you see with this code (technical and business logic)
2.	Explain Impact: For each issue, explain what could go wrong in production
3.	Provide Fixes: Write the corrected version with explanations
Solution :
1. Issue : No Input Validation
Impact: The code directly accesses fields like data['name'], data['sku'], etc. If the request body is missing it will crash and return a 500 instead of a proper client error.
Fix:  Validate request body and required fields before using them.
data = request.get_json()
if not data:
 	return {"error": "Invalid JSON"}, 400
required_fields = ['name', 'sku', 'price', 'warehouse_id', 'initial_quantity']
for field in required_fields:
   if field not in data:
       return {"error": f"{field} is required"}, 400
2. Issue : No Type Validation
Impact: Invalid values for price or initial_quantity can cause runtime errors or incorrect data storage. Also price precision can be affected.
Fix: Validate and convert types safely.
try:
   price = float(data['price'])
   quantity = int(data['initial_quantity'])
except:
   return {"error": "Invalid price or quantity"}, 400
3. Issue : No Error Handling and Rollback
Impact: If any database operation fails the request crashes and may leave the database in an inconsistent state.
Fix: Wrap operations in try except and rollback on failure.
try:
   db.session.commit()
except Exception:
   db.session.rollback()
   return {"error": "Database error"}, 500

4. Issue : Multiple Commits (No Transaction Safety)
Impact:
 The product is committed before inventory is created. If inventory creation fails the product remains without inventory.
Fix:
 Use a single transaction and commit once.
db.session.add(product)
db.session.flush()
db.session.add(inventory)
db.session.commit()
5. Issue : No SKU Uniqueness Check
Impact: Duplicate SKUs can be created breaking inventory tracking and reporting.
Fix: Check before insert and enforce uniqueness.
existing = Product.query.filter_by(sku=data['sku']).first()
if existing:
   return {"error": "SKU already exists"}, 409
6. Issue : Product Incorrectly Linked to One Warehouse
Impact: The design assumes a product belongs to only one warehouse, which contradicts the requirement that products can exist in multiple warehouses.
Fix: Remove warehouse_id from product and manage it via inventory table.
product = Product(
   name=data['name'],
   sku=data['sku'],
   price=price
)
7. Issue : No Warehouse Validation
Impact: Invalid or non existent warehouse IDs can be used leading to broken relationships or errors.
Fix: Validate warehouse existence.
warehouse = Warehouse.query.get(data['warehouse_id'])
if not warehouse:
   return {"error": "Warehouse not found"}, 404
8. Issue : No Duplicate Inventory Check
Impact: Multiple inventory records for the same product and warehouse can be created which leading to incorrect stock values.
Fix: Ensure uniqueness at application level.
existing_inventory = Inventory.query.filter_by(
   product_id=product.id,
   warehouse_id=data['warehouse_id']
).first()
if existing_inventory:
   return {"error": "Inventory already exists"}, 409
9. Issue : No Authentication or Authorization
Impact: Anyone can call this endpoint and create products which is a major security risk in a B2b system.
Fix:
 Add authentication and permission checks.
# Example
# @login_required
# check if user belongs to company before allowing creation
10. Issue: No Validation for Negative Values
Impact: Negative price or quantity can be inserted and it is invalid in real world scenarios and can break business logic.
Fix: Add validation checks.
if price < 0:
   return {"error": "Price cannot be negative"}, 400
if quantity < 0:
   return {"error": "Quantity cannot be negative"}, 400

Additional Context (you may need to ask for more):
●	Products can exist in multiple warehouses
●	SKUs must be unique across the platform
●	Price can be decimal values
●	Some fields might be optional
________________________________________
Part 2: Database Design (25 minutes)
Based on the requirements below, design a database schema. Note: These requirements are intentionally incomplete - you should identify what's missing.
Given Requirements:
●	Companies can have multiple warehouses
●	Products can be stored in multiple warehouses with different quantities
●	Track when inventory levels change
●	Suppliers provide products to companies
●	Some products might be "bundles" containing other products
Your Tasks:
1.	Design Schema: Create tables with columns, data types and relationships

CREATE TABLE companies (
    id INT PRIMARY KEY AUTO_INCREMENT,
    name VARCHAR(255) NOT NULL,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE TABLE warehouses (
    id INT PRIMARY KEY AUTO_INCREMENT,
    company_id INT,
    name VARCHAR(255) NOT NULL,
    location VARCHAR(255),
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    FOREIGN KEY (company_id) REFERENCES companies(id)
);

CREATE TABLE products (
    id INT PRIMARY KEY AUTO_INCREMENT,
    company_id INT,
    name VARCHAR(255) NOT NULL,
    sku VARCHAR(100) UNIQUE NOT NULL,
    price DECIMAL(10,2) NOT NULL,
    is_bundle BOOLEAN DEFAULT FALSE,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    FOREIGN KEY (company_id) REFERENCES companies(id)
);

CREATE TABLE inventory (
    id INT PRIMARY KEY AUTO_INCREMENT,
    product_id INT,
    warehouse_id INT,
    quantity INT DEFAULT 0,
    reserved_quantity INT DEFAULT 0,
    UNIQUE (product_id, warehouse_id),
    FOREIGN KEY (product_id) REFERENCES products(id),
    FOREIGN KEY (warehouse_id) REFERENCES warehouses(id)
);

CREATE TABLE inventory_movements (
    id INT PRIMARY KEY AUTO_INCREMENT,
    product_id INT,
    warehouse_id INT,
    change_quantity INT,
    type VARCHAR(20),
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE TABLE suppliers (
    id INT PRIMARY KEY AUTO_INCREMENT,
    company_id INT,
    name VARCHAR(255),
    contact_email VARCHAR(255),
    FOREIGN KEY (company_id) REFERENCES companies(id)
);

CREATE TABLE product_suppliers (
    product_id INT,
    supplier_id INT,
    is_primary BOOLEAN DEFAULT FALSE,
    PRIMARY KEY (product_id, supplier_id),
    FOREIGN KEY (product_id) REFERENCES products(id),
    FOREIGN KEY (supplier_id) REFERENCES suppliers(id)
);

CREATE TABLE product_bundles (
    bundle_product_id INT,
    component_product_id INT,
    quantity INT NOT NULL,
    PRIMARY KEY (bundle_product_id, component_product_id),
    FOREIGN KEY (bundle_product_id) REFERENCES products(id),
    FOREIGN KEY (component_product_id) REFERENCES products(id)
);

CREATE TABLE sales (
    id INT PRIMARY KEY AUTO_INCREMENT,
    company_id INT,
    warehouse_id INT,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    FOREIGN KEY (company_id) REFERENCES companies(id),
    FOREIGN KEY (warehouse_id) REFERENCES warehouses(id)
);

CREATE TABLE sales_items (
    id INT PRIMARY KEY AUTO_INCREMENT,
    sale_id INT,
    product_id INT,
    quantity INT NOT NULL,
    FOREIGN KEY (sale_id) REFERENCES sales(id),
    FOREIGN KEY (product_id) REFERENCES products(id)
);

2.	Identify Gaps: List questions you'd ask the product team about missing requirements
Can inventory go negative ?
What exactly counts as recent sales? 
Do we need to track reserved stock for pending orders?
Should inactive or discontinued products be included in alerts?
Is low stock threshold fixed or can it be different for each product or warehouse?
Should stock be calculated per warehouse or total across all warehouses?
Can a product have multiple suppliers and how do we decide the primary one?
Should bundles automatically reduce stock of their component products?
Can a product belong to multiple companies or strictly one company?
Do we need audit history for all inventory changes or only major ones?

3.	Explain Decisions: Justify your design choices (indexes, constraints, etc.)
Separate Inventory Table : I kept inventory separate from products because a product can exist in multiple warehouses. This makes the design flexible and scalable.
Unique Constraint (product_id, warehouse_id : This ensures there is only one inventory record per product per warehouse, avoiding duplicate stock entries.
SKU as UNIQUE : SKU is unique because it is used for identification across the system and avoids confusion in tracking.
Inventory Movements Table : Instead of directly updating stock, this helps track every change. It is useful for debugging, reporting and audit purposes.
Mapping Tables (product_suppliers, bundles): These are used because relationships are many-to-many. It keeps the system flexible and easy to extend.
Foreign Keys : Used to maintain data integrity, so invalid references (like wrong product or warehouse) cannot exist.
Simple Structure : I avoided overcomplicating the schema so that it is easy to understand, maintain and scale later.

________________________________________
Part 3: API Implementation (35 minutes)
Implement an endpoint that returns low-stock alerts for a company.
Business Rules (discovered through previous questions):
●	Low stock threshold varies by product type
●	Only alert for products with recent sales activity
●	Must handle multiple warehouses per company
●	Include supplier information for reordering
Endpoint Specification:
GET /api/companies/{company_id}/alerts/low-stock

Expected Response Format:
{
  "alerts": [
    {
      "product_id": 123,
      "product_name": "Widget A",
      "sku": "WID-001",
      "warehouse_id": 456,
      "warehouse_name": "Main Warehouse",
      "current_stock": 5,
      "threshold": 20,
      "days_until_stockout": 12,
      "supplier": {
        "id": 789,
        "name": "Supplier Corp",
        "contact_email": "orders@supplier.com"
      }
    }
  ],
  "total_alerts": 1
}

Your Tasks:
1.	Write Implementation: Use any language/framework (Python/Flask, Node.js/Express, etc.)
from datetime import datetime, timedelta
from flask import jsonify
from sqlalchemy import func
@app.route('/api/companies/<int:company_id>/alerts/low-stock', methods=['GET'])
def low_stock_alerts(company_id):
    cutoff_date = datetime.utcnow() - timedelta(days=30)
    # Step 1: Get recent sales per product
    sales_data = (
        db.session.query(
            sales_items.product_id,
            func.sum(sales_items.quantity)
        )
        .join(sales, sales.id == sales_items.sale_id)
        .filter(
            sales.company_id == company_id,
            sales.created_at >= cutoff_date
        )
        .group_by(sales_items.product_id)
        .all()
    )
    sales_map = {product_id: qty for product_id, qty in sales_data}
    # Step 2: Get inventory with product + warehouse
    rows = (
        db.session.query(products, inventory, warehouses)
        .join(inventory, inventory.product_id == products.id)
        .join(warehouses, warehouses.id == inventory.warehouse_id)
        .filter(products.company_id == company_id)
        .all()
    )
    alerts = []
    for product, inv, wh in rows:
        sold_qty = sales_map.get(product.id, 0)
        # Only consider products with recent sales
        if sold_qty == 0:
            continue
        threshold = 10  # assumed default
        # Check low stock
        if inv.quantity >= threshold:
            continue

        # Calculate average daily sales
        avg_daily_sales = sold_qty / 30
        if avg_daily_sales > 0:
            days_left = int(inv.quantity / avg_daily_sales)
        else:
            days_left = None
        # Get supplier
        supplier = (
            db.session.query(suppliers)
            .join(product_suppliers)
            .filter(product_suppliers.product_id == product.id)
            .first()
        )
        alerts.append({
            "product_id": product.id,
            "product_name": product.name,
            "sku": product.sku,
            "warehouse_id": wh.id,
            "warehouse_name": wh.name,
            "current_stock": inv.quantity,
            "threshold": threshold,
            "days_until_stockout": days_left,
            "supplier": {
                "id": supplier.id,
                "name": supplier.name,
                "contact_email": supplier.contact_email
            } if supplier else None
        })
    return {
        "alerts": alerts,
        "total_alerts": len(alerts)
    }, 200

2.	Handle Edge Cases: Consider what could go wrong
1.	Company not found then return 404.
2.	No sales then no alerts.
3.	No supplier then return null.
4.	Zero sales rate then no stockout calculation.
5.	Multiple warehouses then handled via inventory table. 

3.	Explain Approach: Add comments explaining your logic
Hints: You'll need to make assumptions about the database schema and business logic. Document these assumptions.
1.	Recent sales = last 30 days
2.	Threshold comes from product type (default = 10 if missing)
3.	Stock is checked per warehouse
4.	Only products with sales are considered
5.	Primary supplier is used

________________________________________
Submission Instructions
1.	Create a document with your responses to all three parts
2.	Include reasoning for each decision you made
3.	List assumptions you had to make due to incomplete requirements
4.	Submit within 90 minutes of receiving this case study
5.	Be prepared to walk through your solutions in the live session
________________________________________
Live Session Topics (Preview)
During our video call, we'll discuss:
●	Your debugging approach and thought process
●	Database design trade-offs and scalability considerations
●	How you'd handle edge cases in your API implementation
●	Questions about missing requirements and how you'd gather more info
●	Alternative approaches you considered
________________________________________
Evaluation Criteria
Technical Skills:
●	Code quality and best practices
●	Database design principles
●	Understanding of API design
●	Problem-solving approach
Communication:
●	Ability to identify and ask about ambiguities
●	Clear explanation of technical decisions
●	Professional collaboration style
Business Understanding:
●	Recognition of real-world constraints
●	Consideration of user experience
●	Scalability and maintenance thinking
________________________________________
Note: This is designed to assess your current skill level and learning potential. We don't expect perfect solutions - we're more interested in your thought process and ability to work with incomplete information.

