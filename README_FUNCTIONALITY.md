# Vending Machine API - Complete Functionality Overview

## **Architecture**
- **FastAPI** framework with SQLAlchemy ORM
- **SQLite** database
- **Layered structure**: Routers → Services → Models

---

## **Core Components**

### **1. Configuration (`app/config.py`)**
- `Settings` class manages all config
- `MAX_SLOTS = 10` - max slots allowed
- `MAX_ITEMS_PER_SLOT = 10` - max items per slot
- `SUPPORTED_DENOMINATIONS = [1,2,5,10,20,50,100]` - valid currency
- `DATABASE_URL` - database connection

### **2. Database Models (`app/models.py`)**

**Slot Model:**
- `id` - unique identifier (UUID)
- `code` - slot code (A1, B2, etc.)
- `capacity` - max items slot can hold
- `current_item_count` - tracks current items
- `items` - relationship to Item model

**Item Model:**
- `id` - unique identifier (UUID)
- `name` - item name
- `price` - item price
- `quantity` - available quantity
- `slot_id` - foreign key to Slot
- `slot` - relationship to Slot model

---

## **API Endpoints & Functions**

### **SLOT MANAGEMENT**

**1. POST /slots** - Create new slot
- **Router**: `slots.py → create_slot()`
- **Service**: `slot_service.py → create_slot()`
- **Does**: 
  - Checks if MAX_SLOTS reached
  - Validates unique slot code
  - Creates slot with capacity
  - Returns slot details

**2. GET /slots** - List all slots
- **Router**: `slots.py → list_slots()`
- **Service**: `slot_service.py → list_slots()`
- **Does**: Returns all slots with current item counts

**3. DELETE /slots/{slot_id}** - Remove slot
- **Router**: `slots.py → delete_slot()`
- **Service**: `slot_service.py → delete_slot()`
- **Does**: 
  - Checks if slot has items (prevents deletion)
  - Deletes empty slot

**4. GET /slots/full-view** - Complete view
- **Router**: `slots.py → full_view()`
- **Service**: `slot_service.py → get_full_view()`
- **Does**: Returns all slots with nested items array

---

### **ITEM MANAGEMENT**

**5. POST /slots/{slot_id}/items** - Add single item
- **Router**: `slots.py → add_item_to_slot()`
- **Service**: `item_service.py → add_item_to_slot()`
- **Does**:
  - Validates slot exists
  - Checks capacity not exceeded
  - Creates item
  - Updates slot's current_item_count

**6. POST /slots/{slot_id}/items/bulk** - Add multiple items
- **Router**: `slots.py → bulk_add_items()`
- **Service**: `item_service.py → bulk_add_items()`
- **Does**:
  - Calculates total quantity
  - Validates capacity
  - Adds all items in one transaction
  - Updates slot count

**7. GET /slots/{slot_id}/items** - View slot's items
- **Router**: `slots.py → list_slot_items()`
- **Service**: `item_service.py → list_items_by_slot()`
- **Does**: Returns all items in specific slot

**8. GET /items/{item_id}** - View single item
- **Router**: `items.py → get_item()`
- **Service**: `item_service.py → get_item_by_id()`
- **Does**: Returns item details including slot_id

**9. PATCH /items/{item_id}/price** - Update price
- **Router**: `items.py → update_item_price()`
- **Service**: `item_service.py → update_item_price()`
- **Does**: Updates item price, auto-updates timestamp

**10. DELETE /slots/{slot_id}/items/{item_id}** - Remove items
- **Router**: `items.py → remove_item_from_slot()`
- **Service**: `item_service.py → remove_item_quantity()`
- **Does**:
  - If `?quantity=N` → subtracts N items
  - If no quantity → deletes entire item
  - Updates slot count
  - Deletes item if quantity reaches 0

**11. DELETE /slots/{slot_id}/items** - Bulk remove
- **Router**: `items.py → bulk_remove_items()`
- **Service**: `item_service.py → bulk_remove_items()`
- **Does**:
  - If body has `item_ids` → removes specific items
  - If no body → empties entire slot
  - Updates slot count

---

### **PURCHASE SYSTEM**

**12. POST /purchase** - Buy item
- **Router**: `purchase.py → purchase()`
- **Service**: `purchase_service.py → purchase()`
- **Does**:
  - Validates item exists
  - Checks quantity > 0
  - Validates cash >= price
  - Calculates change
  - Decrements item quantity by 1
  - Decrements slot count by 1
  - Commits transaction atomically
  - Returns purchase details

**13. GET /purchase/change-breakdown** - Calculate change
- **Router**: `purchase.py → change_breakdown()`
- **Service**: `purchase_service.py → change_breakdown()`
- **Does**:
  - Takes change amount
  - Uses greedy algorithm
  - Returns denomination breakdown (100→50→20→10→5→2→1)

---

## **How It Works (Flow Example)**

### **Adding Item to Slot:**
1. User sends POST to `/slots/A1/items` with `{name: "Coke", price: 40, quantity: 5}`
2. Router validates request schema
3. Service checks slot exists
4. Service validates: `current_count(0) + quantity(5) <= capacity(10)` ✓
5. Creates Item in database
6. Updates Slot: `current_item_count = 5`
7. Commits transaction
8. Returns item details

### **Purchasing Item:**
1. User sends POST to `/purchase` with `{item_id: "uuid", cash_inserted: 50}`
2. Service fetches item from DB
3. Validates: item exists ✓, quantity > 0 ✓, cash(50) >= price(40) ✓
4. Calculates: change = 50 - 40 = 10
5. Updates: `item.quantity -= 1`, `slot.current_item_count -= 1`
6. Commits transaction
7. Returns: `{item: "Coke", price: 40, change_returned: 10, remaining_quantity: 4}`

---

## **Key Features**
- ✅ Atomic transactions (all DB operations commit together)
- ✅ Capacity validation (prevents overfilling slots)
- ✅ Cascade delete (items deleted when slot deleted)
- ✅ Real-time inventory tracking
- ✅ Change calculation with denomination breakdown
- ✅ Comprehensive error handling

---

## **Bugs Fixed During Development**

### **1. Missing Denominations in Config**
- **File**: `app/config.py`
- **Issue**: `SUPPORTED_DENOMINATIONS` was `[5, 10, 20, 50, 100]` - missing 1 and 2
- **Fix**: Changed to `[1, 2, 5, 10, 20, 50, 100]` as per specification
- **Why**: Spec explicitly requires all 7 denominations for change calculation

### **2. Price Validation Allowed Zero**
- **File**: `app/schemas.py`
- **Classes**: `ItemCreate`, `ItemBulkEntry`
- **Issue**: `price: int = Field(..., ge=0)` allowed price = 0
- **Fix**: Changed to `price: int = Field(..., gt=0)`
- **Why**: Spec rule states "price > 0"

### **3. Incorrect Capacity Logic**
- **File**: `app/services/item_service.py`
- **Function**: `add_item_to_slot()`
- **Issue**: Had backwards check `if slot.current_item_count + data.quantity < settings.MAX_ITEMS_PER_SLOT`
- **Fix**: Removed this line (correct check already existed above)
- **Why**: Would raise error when count was LESS than max (opposite of intended)

### **4. Missing Capacity Validation in Bulk Add**
- **File**: `app/services/item_service.py`
- **Function**: `bulk_add_items()`
- **Issue**: No capacity validation before adding items
- **Fix**: Added `total_qty = sum(e.quantity for e in entries)` and capacity check
- **Why**: Spec rule "Total new quantity must not exceed slot capacity"

### **5. Timestamp Preservation Bug**
- **File**: `app/services/item_service.py`
- **Function**: `update_item_price()`
- **Issue**: Code preserved old `updated_at` timestamp when updating price
- **Fix**: Removed timestamp preservation, let SQLAlchemy auto-update
- **Why**: Prevents tracking when price was last modified

### **6. Wrong Cascade Delete Configuration**
- **File**: `app/models.py`
- **Class**: `Slot`
- **Issue**: `cascade="save-update, merge"` didn't delete items when slot deleted
- **Fix**: Changed to `cascade="all, delete-orphan"`
- **Why**: Items should be deleted when their slot is deleted

### **7. Wrong Foreign Key Constraint**
- **File**: `app/models.py`
- **Class**: `Item`
- **Issue**: `ondelete="SET NULL"` and `nullable=True` allowed orphaned items
- **Fix**: Changed to `ondelete="CASCADE"` and `nullable=False`
- **Why**: Items must belong to a slot and cascade delete with slot

### **8. Missing Slot Deletion Validation**
- **File**: `app/services/slot_service.py`
- **Function**: `delete_slot()`
- **Issue**: No check if slot contains items before deletion
- **Fix**: Added `if slot.current_item_count > 0: raise ValueError("slot_contains_items")`
- **Why**: Spec rule "Cannot delete if slot contains items"

### **9. Missing Error Handler**
- **File**: `app/routers/slots.py`
- **Function**: `delete_slot()`
- **Issue**: No error handling for slot_contains_items
- **Fix**: Added handler returning 400 error with message
- **Why**: Provide proper error response to user

### **10. Demo Code Delays**
- **Files**: `app/services/item_service.py`, `app/services/purchase_service.py`
- **Issue**: `time.sleep(0.05)` delays in production code
- **Fix**: Removed all sleep statements
- **Why**: Unnecessary delays slow down API responses
