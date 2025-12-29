####PART1

from flask import request, jsonify
from decimal import Decimal
from sqlalchemy.exc import IntegrityError
from models import db, Product, Warehouse, Inventory

@app.route('/api/products', methods=['POST'])
def create_product():
    data = request.get_json()

    # ---- REQUIRED FIELD CHECK ----
    required_fields = ["name", "sku", "price", "warehouse_id"]
    for field in required_fields:
        if field not in data:
            return jsonify({"error": f"{field} is required"}), 400

    # ---- PRICE VALIDATION ----
    try:
        price = Decimal(str(data["price"]))
    except:
        return jsonify({"error": "Invalid price format"}), 400

    # Optional field
    initial_qty = data.get("initial_quantity", 0)

    # ---- BUSINESS RULES ----
    # SKU must be unique
    if Product.query.filter_by(sku=data["sku"]).first():
        return jsonify({"error": "SKU already exists"}), 409

    # Warehouse must exist
    warehouse = Warehouse.query.get(data["warehouse_id"])
    if not warehouse:
        return jsonify({"error": "Warehouse not found"}), 404

    try:
        # ---- ATOMIC TRANSACTION ----
        with db.session.begin():

            # Create Product
            product = Product(
                name=data["name"],
                sku=data["sku"],
                price=price
            )
            db.session.add(product)
            db.session.flush()   # get product.id

            # Create Inventory Entry
            inventory = Inventory(
                product_id=product.id,
                warehouse_id=data["warehouse_id"],
                quantity=initial_qty
            )

######PART3:

from flask import jsonify
from models import (
    db, Product, Warehouse, Inventory,
    Supplier, ProductSupplier
)

@app.route('/api/companies/<int:company_id>/alerts/low-stock')
def low_stock_alerts(company_id):

    try:
        results = db.session.query(
            Product.id,
            Product.name,
            Product.sku,
            Warehouse.id.label("warehouse_id"),
            Warehouse.name.label("warehouse_name"),
            Inventory.quantity,
            Product.threshold,
            Supplier.id.label("supplier_id"),
            Supplier.name.label("supplier_name"),
            Supplier.contact_email
        ).join(Inventory, Inventory.product_id == Product.id) \
         .join(Warehouse, Warehouse.id == Inventory.warehouse_id) \
         .outerjoin(ProductSupplier, ProductSupplier.product_id == Product.id) \
         .outerjoin(Supplier, Supplier.id == ProductSupplier.supplier_id) \
         .filter(Warehouse.company_id == company_id) \
         .filter(Inventory.quantity < Product.threshold) \
         .all()

        alerts = []

        for r in results:
            # Example simple stock-out estimation logic
            days_until_stockout = 10 if r.quantity > 0 else 0

            alerts.append({
                "product_id": r.id,
                "product_name": r.name,
                "sku": r.sku,
                "warehouse_id": r.warehouse_id,
                "warehouse_name": r.warehouse_name,
                "current_stock": r.quantity,
                "threshold": r.threshold,
                "days_until_stockout": days_until_stockout,
                "supplier": {
                    "id": r.supplier_id,
                    "name": r.supplier_name,
                    "contact_email": r.contact_email
                } if r.supplier_id else None
            })

        return jsonify({
            "alerts": alerts,
            "total_alerts": len(alerts)
        }), 200

    except Exception:
        return jsonify({"error": "Something went wrong"}), 500
