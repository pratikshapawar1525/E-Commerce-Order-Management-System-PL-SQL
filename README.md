# E-Commerce-Order-Management-System-PL-SQL
Build six procedures covering product search, stock validation, order placement, order cancellation, customer history and sales reporting, with special attention to stock consistency.



---

## Files

| File | Contents |
|---|---|
| `database.sql` | Table creation, constraints, sequences, sample data, verification queries |
| `procedures.sql` | All six procedures + the bonus procedure, with a demo script at the bottom |
| `README.md` | This file |

---

## How to run

```sql
-- SQL*Plus / SQL Developer, connected as the schema owner
SQL> SET SERVEROUTPUT ON
SQL> @database.sql
SQL> @procedures.sql
```

`database.sql` starts with a PL/SQL clean-up block that drops the four tables and four
sequences if they already exist, so it is safe to re-run from scratch at any time.

`SET SERVEROUTPUT ON` matters — the procedures print their confirmation messages with
`DBMS_OUTPUT`, and without it you will see the result set but not the messages.

---

## Oracle-specific design notes

Two things work differently here than in MySQL/SQL Server, and both shaped the design:

1. **Procedures cannot return a result grid.** Every reporting procedure therefore takes
   an `OUT SYS_REFCURSOR` parameter. In SQL*Plus you bind a cursor variable and print it:

   ```sql
   VARIABLE rc REFCURSOR
   EXEC SearchProducts('Electronics', 50000, :rc);
   PRINT rc
   ```

   Procedures that mainly confirm an action (`PlaceOrder`, `CancelOrder`) return scalar
   `OUT` parameters instead, and also print a formatted summary via `DBMS_OUTPUT`.

2. **There is no `AUTO_INCREMENT`.** New ids come from sequences
   (`seq_order_id`, `seq_order_item_id`, …), which start after the highest seeded value
   so the sample data keeps readable ids — customers 101…, products 301…, orders 5001…,
   items 9001… The first order created by `PlaceOrder` will be 5016.

---

## Schema

```
customers (8)            products (15)
    |                        |
    | customer_id            | product_id
    v                        v
orders (15) -----------> order_items (25)
             order_id
```

* **customers** — `customer_id` PK, `UNIQUE` email, `CHECK` on status (Active/Inactive).
* **products** — `product_id` PK, `CHECK (stock_quantity >= 0)` so the database itself
  refuses to go negative even if a procedure were bypassed.
* **orders** — `order_id` PK, FK to customers (no `ON DELETE`, i.e. Oracle's default
  `RESTRICT`, so a customer with order history cannot silently disappear),
  `CHECK` on status (Pending/Completed/Cancelled).
* **order_items** — `order_item_id` PK, FK to orders (`ON DELETE CASCADE`) and to
  products (restrict). `UNIQUE (order_id, product_id)` stops the same product appearing
  as two separate lines on one order.

`unit_price` is stored on the line item: prices in `products` change over time, so
copying the price at purchase keeps old orders reporting what was actually charged.

The stock values in the sample data are the **current** stock — quantities consumed by
the seeded orders are already deducted, and the two cancelled orders (5004, 5013) have
already had their stock restored. `database.sql` ends with a query proving every
`orders.total_amount` equals the sum of its line items (it returns no rows).

---

## Procedures

### 1. `SearchProducts(p_category, p_max_price, p_result OUT SYS_REFCURSOR)`
Active products only, sorted by price ascending. Both filters are optional — pass `NULL`
to skip either one, so the same procedure serves "all electronics", "anything under
₹5,000" and "everything". Category matching is case-insensitive. Adds a derived
`stock_status` column (In Stock / Low Stock / Out of Stock).

```sql
EXEC SearchProducts('Electronics', 50000, :rc);
EXEC SearchProducts(NULL, 5000, :rc);
```

### 2. `CheckStock(p_product_id, p_requested_quantity, p_result OUT SYS_REFCURSOR)`
Read-only availability check. Rejects zero/negative quantities (ORA-20002), unknown
products (ORA-20003, caught via `NO_DATA_FOUND`) and inactive products (ORA-20004).
Returns available stock, requested quantity, a Sufficient/Insufficient verdict and the
shortfall.

### 3. `PlaceOrder(p_customer_id, p_product_id, p_quantity, p_order_id OUT, p_total_amount OUT, p_order_status OUT)`
The core business logic. Validates the quantity and the customer first, then:

```
SELECT ... FOR UPDATE       -- lock the product row
check product status + stock
INSERT INTO orders
INSERT INTO order_items
UPDATE products SET stock_quantity = stock_quantity - quantity
COMMIT
```

Stock is reduced **only after both inserts succeed**. `FOR UPDATE` holds the product row
until commit, so two concurrent sessions cannot both pass the stock check and oversell
the same unit. A `WHEN OTHERS THEN ROLLBACK; RAISE;` handler means a failed order leaves
no partial rows and no stock drift, while the caller still receives the real error.

```sql
EXEC PlaceOrder(101, 301, 2, :oid, :amt, :st);   -- stock 25 -> 23
EXEC PlaceOrder(108, 301, 1, :oid, :amt, :st);   -- ORA-20007 inactive customer
EXEC PlaceOrder(101, 309, 500, :oid, :amt, :st); -- ORA-20010 insufficient stock
```

### 4. `CancelOrder(p_order_id, p_cancel_status OUT, p_restored_quantity OUT)`
Only **Pending** orders can be cancelled; Completed (ORA-20013) and already-Cancelled
(ORA-20012) orders are refused with distinct messages, so a double-cancel can never
restore stock twice. The header is locked with `FOR UPDATE` first. Stock is returned for
*every* line of the order in a single set-based `MERGE`, then the status is updated —
both in one transaction, with the same rollback-and-raise handler.

```sql
EXEC CancelOrder(5001, :st, :qty);   -- restores 1 smartphone + 2 earbuds = 3 units
EXEC CancelOrder(5002, :st, :qty);   -- ORA-20013, order is Completed
```

### 5. `GetCustomerOrderHistory(p_customer_id, p_result OUT SYS_REFCURSOR)`
Four-table join (`customers → orders → order_items → products`), grouped per order, with
`LISTAGG` rolling the line items into a readable product list and a `product xQty` detail
string. Shows item count, total quantity, calculated items value, stored total and
status, newest order first. A customer who exists but has no orders gets a friendly
one-row message; an unknown `customer_id` raises ORA-20014.

### 6. `GetSalesReport(p_start_date, p_end_date, p_result OUT SYS_REFCURSOR)`
Statistics for **Completed** orders in the range. Order count and revenue come from the
`orders` header so a multi-line order is never double-counted; units sold come from
`order_items`. Validates that both dates are supplied and that the range is the right way
round. A period with no sales returns a row of zeros plus an explanatory status rather
than an empty grid, and the average order value is guarded against division by zero.

```sql
EXEC GetSalesReport(DATE '2026-09-01', DATE '2026-09-30', :rc);
EXEC GetSalesReport(DATE '2020-01-01', DATE '2020-12-31', :rc);   -- no-sales period
```

### Bonus — `GetTopSellingProducts(p_start_date, p_end_date, p_result OUT SYS_REFCURSOR)`
Top 5 products by quantity sold across Completed orders in the range, with product name,
category, quantity, sales amount and how many orders each appeared in. Implemented as
`ROWNUM <= 5` over a sorted inline view, which is the version-safe Top-N pattern
(12c and above could use `FETCH FIRST 5 ROWS ONLY`).

---

## Error handling

All business-rule violations are raised with `RAISE_APPLICATION_ERROR` using codes
**-20001 to -20018**, so the calling application receives a real Oracle error rather than
a silent no-op. Missing rows are caught with `NO_DATA_FOUND` and converted into a clear
message that names the offending id. Cases covered:

* zero, negative or `NULL` quantities, and negative price filters
* non-existent `customer_id`, `product_id`, `order_id`
* inactive customers and inactive products
* insufficient stock (message includes available vs requested)
* cancelling a Completed or already-Cancelled order
* missing or reversed date ranges

The two procedures that write to more than one table (`PlaceOrder`, `CancelOrder`)
manage their own transaction and end with `WHEN OTHERS THEN ROLLBACK; RAISE;`, so
inventory and order data can never fall out of sync.

---

## Demo checklist

The end of `procedures.sql` holds a ready-to-paste SQL*Plus demo block. For the 2–5
minute video, this order tells the story fastest:

1. `SearchProducts` with and without a category filter
2. `CheckStock` — sufficient, insufficient, then an invalid product
3. Query stock for product 301 → `PlaceOrder(101, 301, 2)` → query again to show the
   deduction, then a rejected order to show the rollback leaves stock untouched
4. Query stock for 301 and 303 → `CancelOrder(5001)` → query again to show both
   quantities restored, then try cancelling a Completed order
5. `GetCustomerOrderHistory(101)`
6. `GetSalesReport` for September, then for a period with no sales
7. `GetTopSellingProducts`

Keep `SERVEROUTPUT` on and the `PRINT rc` lines handy so both the messages and the grids
are visible on screen.
