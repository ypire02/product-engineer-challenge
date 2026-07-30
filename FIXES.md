# 🐛 Product Engineer Challenge - Bug Fixes

## Overview
This document outlines the 5 bugs found and fixed in the NestJS backend API for an e-commerce platform.

---

## Bug #1: Cache Search - Duplicate Key 🔴

**File:** `src/products/products.service.ts` (line 53)

**Problem:**
All search queries cached under the same key `'product-search'`, causing wrong results.

```typescript
// ❌ BEFORE - All searches use the same cache key
const cacheKey = 'product-search';
```

**Impact:**
- User searches "gato" → gets results cached
- User searches "perro" → gets "gato" results instead
- Cache consistency broken

**Solution:**
Make cache key unique based on query:

```typescript
// ✅ AFTER - Dynamic cache key with query
const cacheKey = `product-search-${query.toLowerCase()}`;
```

**How to test:**
POST /products/search?q=gato → caches under `product-search-gato`
POST /products/search?q=perro → caches under `product-search-perro`

---

## Bug #2: Retry Loop Infinite - Payment Service 🔴

**File:** `src/orders/orders.service.ts` (line 26)

**Problem:**
1000 retries with 100ms delay = 100 seconds frozen if payment fails.

```typescript
// ❌ BEFORE - Excessive retries
private maxRetries = 1000;
```

**Impact:**
- Requests "extremely slow or never complete"
- User creates order → payment fails → waits 100 seconds
- Poor UX, wasted resources

**Solution:**
Reduce to reasonable retry count:

```typescript
// ✅ AFTER - Practical retry limit
private maxRetries = 3;
```

**How to test:**
POST /orders/:id/pay → Max waits ~300ms (3 retries × 100ms)

---

## Bug #3: Missing Await - Stock Update 🔴

**File:** `src/orders/orders.service.ts` (line 89)

**Problem:**
Stock update not awaited → async operation not synchronized.

```typescript
// ❌ BEFORE - Missing await
this.productsService.updateStock(product.id, product.stock - itemDto.quantity);
```

**Impact:**
- Order created but stock not updated yet
- Race condition: multiple orders can oversell same product
- Data inconsistency

**Solution:**
Add `await` to wait for completion:

```typescript
// ✅ AFTER - Awaited async operation
await this.productsService.updateStock(product.id, product.stock - itemDto.quantity);
```

**How to test:**
POST /orders → Product stock decrements before order completes

---

## Bug #4: Circular Reference - JSON Serialization 🔴

**File:** `src/orders/orders.service.ts` (line 154)

**Problem:**
Creating circular reference in object graph breaks JSON serialization.

```typescript
// ❌ BEFORE - Circular reference
const enriched: any = { ...order };
enriched.user = { ...order.user };
enriched.user.latestOrder = enriched;  // ← Creates cycle
return JSON.parse(JSON.stringify(enriched));  // ← Fails or hangs
```

**Impact:**
- GET /orders/:id/full → JSON.stringify fails silently
- Unexpected behavior, difficult to debug
- Crashes or hangs serialization

**Solution:**
Remove circular reference:

```typescript
// ✅ AFTER - No circular reference
const enriched: any = { ...order };
enriched.user = { ...order.user };
// enriched.user.latestOrder = enriched;  // ← Removed
return JSON.parse(JSON.stringify(enriched));  // ← Works fine
```

**How to test:**
GET /orders/:id/full → Returns JSON without hanging

---

## Bug #5: Recursion Without Protection - Category Tree 🔴

**File:** `src/products/products.service.ts` (lines 94-110)

**Problem:**
Recursive function has no protection against circular references in category hierarchy.

```typescript
// ❌ BEFORE - No cycle detection
private buildCategoryTree(category: Category): any {
  const tree: any = {
    id: category.id,
    name: category.name,
    children: [],
  };

  if (category.parentId) {
    tree.parent = this.buildCategoryTree(category.parent);  // ← Can infinite loop
  }

  if (category.children && category.children.length > 0) {
    tree.children = category.children.map(child => this.buildCategoryTree(child));
  }

  return tree;
}
```

**Impact:**
- If category A → parent is category B, and B → parent is A
- Stack overflow / Maximum call stack size exceeded
- GET /categories/:id/tree crashes

**Solution:**
Add `visited` Set to track visited categories:

```typescript
// ✅ AFTER - Cycle detection
private buildCategoryTree(category: Category, visited = new Set<number>()): any {
  if (visited.has(category.id)) {
    return { id: category.id, name: category.name, children: [] };
  }
  
  visited.add(category.id);
  
  const tree: any = {
    id: category.id,
    name: category.name,
    children: [],
  };

  if (category.parentId) {
    tree.parent = this.buildCategoryTree(category.parent, visited);
  }

  if (category.children && category.children.length > 0) {
    tree.children = category.children.map(child => this.buildCategoryTree(child, visited));
  }

  return tree;
}
```

**How to test:**
GET /categories/:id/tree → Works even with circular parent-child relationships

---

## Summary Table

| Bug | Type | Severity | Fix |
|-----|------|----------|-----|
| #1 | Cache Logic | High | Dynamic cache key |
| #2 | Performance | Critical | Reduce retries 1000→3 |
| #3 | Race Condition | High | Add `await` |
| #4 | JSON Serialization | Medium | Remove circular ref |
| #5 | Stack Overflow | High | Add cycle detection |

---

## Testing Checklist

- [ ] Cache search returns correct results for different queries
- [ ] Payment retries complete quickly (< 1 second on success)
- [ ] Product stock decrements correctly when order created
- [ ] GET /orders/:id/full returns valid JSON
- [ ] GET /categories/:id/tree doesn't crash with circular hierarchies

---

**Fixed on:** 2026-07-30  
**Bugs fixed:** 5/5 ✅
