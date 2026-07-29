# Restocking Feature - Implementation Verification

## Overview
Complete "Restocking" tab implementation with budget-based item selection and order placement.

## Backend Implementation

### New Endpoints

#### GET `/api/restocking/recommendations`
- **Description:** Returns low-stock inventory items (quantity_on_hand < reorder_point)
- **Query Params:** `warehouse` (optional), `category` (optional)
- **Response:** List of RestockingRecommendation objects
- **Example:**
```bash
curl http://localhost:8001/api/restocking/recommendations?warehouse=Tokyo&category=Power%20Supplies
```

#### POST `/api/restocking/orders`
- **Description:** Creates a new restocking order and appends it to the in-memory orders list
- **Request Body:**
```json
{
  "items": [
    {"sku": "PSU-508", "name": "Battery Backup Power Supply", "quantity": 25, "unit_price": 185.5}
  ],
  "warehouse": "Tokyo",
  "category": "Power Supplies"
}
```
- **Response:** New Order object with status "Restocking", order_date (now), expected_delivery (14 days out)
- **Example:**
```bash
curl -X POST http://localhost:8001/api/restocking/orders \
  -H "Content-Type: application/json" \
  -d '{"items":[{"sku":"PSU-508","name":"Battery Backup Power Supply","quantity":25,"unit_price":185.5}],"warehouse":"Tokyo","category":"Power Supplies"}'
```

### Pydantic Models

**RestockingRecommendation**
- `id: str` - Inventory item ID
- `sku: str` - Stock keeping unit
- `name: str` - Item name
- `category: str` - Product category
- `warehouse: str` - Warehouse location
- `quantity_on_hand: int` - Current stock
- `reorder_point: int` - Reorder threshold
- `quantity_to_order: int` - Quantity to reorder (reorder_point - quantity_on_hand)
- `unit_cost: float` - Cost per unit
- `line_cost: float` - Total cost (quantity_to_order * unit_cost)

**RestockingOrderRequest**
- `items: List[dict]` - Items to order (each with sku, name, quantity, unit_price)
- `warehouse: Optional[str]` - Warehouse (defaults to first item's warehouse or "Multiple")
- `category: Optional[str]` - Category (defaults to first item's category or "Multiple")

## Frontend Implementation

### New View: `Restocking.vue`

**Features:**
1. **Budget Slider** - $1,000 to $50,000 (step $1,000)
2. **Real-time Stats** - Selected items count, excluded items, estimated cost, remaining budget
3. **Recommendations Table** - Displays low-stock items sorted by line_cost (descending)
   - Columns: Item Name, SKU, Category, Warehouse, In Stock, Reorder Point, Qty to Order, Unit Cost, Line Total
   - Visual feedback: Selected items highlighted, excluded items grayed out
4. **Greedy Selection Algorithm** - Client-side selection that includes items in budget order
5. **Place Order Button** - Submits selected items, shows success message, reloads recommendations

**Composable Usage:**
- `useI18n()` - For all UI text translations
- `useFilters()` - For warehouse and category filtering

### Route & Navigation
- Route: `/restocking` (added to Vue Router in `main.js`)
- Nav Link: Added to `App.vue` navigation bar
- Label: Internationalized ("Restocking" in EN, "再発注" in JA)

### Translation Keys (New)
**English (`en.js`):**
- `nav.restocking: 'Restocking'`
- `restocking.title`, `restocking.description`, `restocking.budgetSlider`, etc. (18 strings total)

**Japanese (`ja.js`):**
- `nav.restocking: '再発注'`
- Mirror all English keys in Japanese

### API Client Updates (`api.js`)
```javascript
getRestockingRecommendations(filters = {}) // GET /restocking/recommendations
placeRestockingOrder(orderData) // POST /restocking/orders
```

## Orders Tab Enhancement

### New Section: "Submitted Restocking Orders"
- **Location:** Top of Orders view (above status stats)
- **Visibility:** Only shown if restocking orders exist
- **Content:** 
  - Separate table showing orders with status "Restocking"
  - Columns: Order #, Date Submitted, Items, Total Value, Expected Delivery, Lead Time
  - Lead time displayed as "14 days" badge
  
### Main Orders Table Update
- **Filter:** Excludes orders with status "Restocking" (to prevent duplication)
- **Status Stats:** Only count non-restocking orders

### Implementation
- `restockingOrders` computed property - filters orders by status === "Restocking"
- `regularOrders` computed property - filters out restocking orders
- Stats updated to use `regularOrders` for counts

## Test Results

### Backend Endpoint Tests
```
✓ GET /api/restocking/recommendations - Returns 4 low-stock items
✓ POST /api/restocking/orders - Creates order with status "Restocking"
✓ Existing endpoints still functional (inventory, orders, dashboard, etc.)
```

### Workflow Test
1. Budget: $15,000
2. Recommendations: 4 items available
3. Selected items (within budget): All 4 items ($10,550 total)
4. Order placed: RST-0254 created with status "Restocking"
5. Expected delivery: 2026-08-12 (14 days from order date)
6. Appears in /api/orders: ✓ Confirmed

### Data Integrity
- Low-stock items identified correctly (4 items below reorder point)
- Line costs calculated accurately
- Budget filtering works correctly
- 14-day lead time calculated from current date

## Files Modified

| File | Changes |
|------|---------|
| `server/main.py` | Added datetime import, RestockingRecommendation & RestockingOrderRequest models, 2 new endpoints |
| `client/src/api.js` | Added getRestockingRecommendations & placeRestockingOrder methods |
| `client/src/main.js` | Added Restocking import and route |
| `client/src/App.vue` | Added restocking nav link |
| `client/src/views/Restocking.vue` | **NEW** - Complete restocking view component |
| `client/src/views/Orders.vue` | Added restocking orders section, separated from regular orders |
| `client/src/locales/en.js` | Added nav.restocking + 18 restocking.* strings |
| `client/src/locales/ja.js` | Added nav.restocking + 18 restocking.* strings (Japanese) |

## URLs for Testing

**Backend APIs:**
- GET recommendations: `http://localhost:8001/api/restocking/recommendations`
- POST order: `http://localhost:8001/api/restocking/orders`
- API docs: `http://localhost:8001/docs`

**Frontend:**
- Restocking tab: `http://localhost:3000/restocking`
- Orders tab: `http://localhost:3000/orders` (shows Submitted Restocking Orders section)

## Key Design Decisions

1. **Greedy Selection Algorithm:** Items sorted by line_cost (most expensive first) ensures budget is meaningful and transparent
2. **Client-side Calculation:** No extra API calls when slider moves - all computation in computed properties
3. **Separate Restocking Status:** New "Restocking" status clearly differentiates from regular order statuses
4. **14-day Fixed Lead Time:** Realistic for demo purposes, simplifies implementation
5. **In-memory Persistence:** Orders appended to runtime list, persist for session duration
6. **Filter Integration:** Warehouse/category filters apply to recommendations, matching existing patterns
