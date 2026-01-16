# Performance Analysis Report

**Project:** Portor Pole Pricing Q4 2025
**File:** `index.html` (318 lines, 98KB)
**Products:** 235 SKUs across 3 regions

---

## Executive Summary

This single-page application is a product pricing catalog with search, filtering, and region switching capabilities. While functional, several performance anti-patterns were identified that could impact user experience, especially on lower-powered devices or with larger datasets.

---

## Critical Performance Issues

### 1. No Debouncing on Search Input ⚠️ HIGH IMPACT

**Location:** `index.html:306`
```javascript
searchInput.addEventListener('input', filterProducts);
```

**Problem:** Every keystroke triggers the full `filterProducts()` function, which:
- Filters up to 235 products
- Rebuilds the entire DOM table
- Executes regex highlighting on every cell

**Impact:** Typing "steel" fires 5 separate filter+render cycles instead of 1.

**Recommendation:** Add debounce (150-300ms):
```javascript
let debounceTimer;
searchInput.addEventListener('input', () => {
    clearTimeout(debounceTimer);
    debounceTimer = setTimeout(filterProducts, 200);
});
```

---

### 2. Full DOM Destruction/Recreation on Every Update ⚠️ HIGH IMPACT

**Location:** `index.html:264`
```javascript
tableBody.innerHTML = html;
```

**Problem:** Using `innerHTML` completely destroys and recreates all DOM nodes on every filter change. For 235 products with 12 columns each, this means:
- Destroying ~2,820+ DOM nodes
- Creating ~2,820+ new DOM nodes
- Triggering full layout reflow and repaint

**Impact:** Causes visible UI jank, especially on mobile devices.

**Recommendation:** Use DOM diffing, virtual DOM, or at minimum:
- Use `DocumentFragment` for batch DOM insertion
- Only update changed rows instead of full table replacement

---

### 3. Multiple Array Iterations in Filter Logic ⚠️ MEDIUM IMPACT

**Location:** `index.html:273-291`
```javascript
let filtered = data[currentRegion];

if (category) {
    filtered = filtered.filter(p => p.main_category === category);
}

if (subCategory) {
    filtered = filtered.filter(p => p.sub_category === subCategory);
}

if (query) {
    filtered = filtered.filter(p =>
        p.sku.toLowerCase().includes(query) || ...
    );
}
```

**Problem:** Three separate `.filter()` calls create intermediate arrays and iterate over data multiple times.

**Impact:** O(3n) instead of O(n) complexity; creates garbage for GC.

**Recommendation:** Combine into single pass:
```javascript
let filtered = data[currentRegion].filter(p => {
    if (category && p.main_category !== category) return false;
    if (subCategory && p.sub_category !== subCategory) return false;
    if (query && !matchesQuery(p, query)) return false;
    return true;
});
```

---

### 4. Regex Recreation on Every Cell ⚠️ MEDIUM IMPACT

**Location:** `index.html:209-212`
```javascript
function highlightText(text, query) {
    if (!query || !text) return text;
    const regex = new RegExp(`(${query.replace(/[.*+?^${}()|[\]\\]/g, '\\$&')})`, 'gi');
    return text.replace(regex, '<span class="highlight">$1</span>');
}
```

**Problem:** A new `RegExp` object is created for every call to `highlightText()`. With 235 products × 3 text fields = 705 regex compilations per filter operation.

**Impact:** Unnecessary CPU cycles and memory allocation.

**Recommendation:** Create regex once and reuse:
```javascript
function renderTable(products, query = '') {
    const highlightRegex = query ?
        new RegExp(`(${query.replace(/[.*+?^${}()|[\]\\]/g, '\\$&')})`, 'gi') : null;

    function highlight(text) {
        if (!highlightRegex || !text) return text;
        return text.replace(highlightRegex, '<span class="highlight">$1</span>');
    }
    // ... use highlight() instead of highlightText()
}
```

---

### 5. Repeated toLowerCase() Calls ⚠️ LOW-MEDIUM IMPACT

**Location:** `index.html:285-289`
```javascript
p.sku.toLowerCase().includes(query) ||
(p.pole_height && p.pole_height.toLowerCase().includes(query)) ||
(p.wall_thickness && p.wall_thickness.toLowerCase().includes(query)) ||
(p.main_category && p.main_category.toLowerCase().includes(query)) ||
(p.sub_category && p.sub_category.toLowerCase().includes(query))
```

**Problem:** Each product field is converted to lowercase on every filter operation. For 235 products × 5 fields = 1,175 `toLowerCase()` calls per keystroke.

**Impact:** Unnecessary string operations and memory allocation.

**Recommendation:** Pre-compute lowercase versions:
```javascript
// On data load, add lowercase versions
data[region].forEach(p => {
    p._searchText = [p.sku, p.pole_height, p.wall_thickness,
                     p.main_category, p.sub_category]
                    .filter(Boolean).join(' ').toLowerCase();
});

// In filter
filtered = filtered.filter(p => p._searchText.includes(query));
```

---

### 6. String Concatenation in Loop ⚠️ LOW IMPACT

**Location:** `index.html:228-258`
```javascript
let html = '';
products.forEach(p => {
    html += `<tr class="category-row">...`;
    html += `<tr class="sub-category-row">...`;
    html += `<tr>...</tr>`;
});
```

**Problem:** String concatenation with `+=` in a loop creates many intermediate strings.

**Impact:** Minor; modern JS engines optimize this well, but array join is cleaner.

**Recommendation:** Use array and join:
```javascript
const rows = [];
products.forEach(p => {
    rows.push(`<tr>...</tr>`);
});
tableBody.innerHTML = rows.join('');
```

---

## Minor Performance Concerns

### 7. Continuous CSS Animation

**Location:** `index.html:33`
```css
@keyframes bounce { 0%, 100% { transform: translateY(0); } 50% { transform: translateY(4px); } }
.prompt-icon { animation: bounce 1.5s infinite; }
```

**Problem:** Animation runs continuously even when the element may be off-screen.

**Impact:** Minor battery/CPU drain on mobile devices.

**Recommendation:** Use `animation-play-state` or Intersection Observer to pause when not visible.

---

### 8. No Virtualization/Pagination

**Location:** Throughout render logic

**Problem:** All 235 products are rendered to DOM simultaneously. While manageable at current size, this doesn't scale.

**Impact:** Current impact is low, but would become critical with 1000+ products.

**Recommendation:** For future scalability:
- Implement virtual scrolling (only render visible rows)
- Or add pagination (show 50 items per page)

---

### 9. Filter Dropdowns Rebuilt on Region Switch

**Location:** `index.html:215-225`
```javascript
function populateFilters() {
    const products = data[currentRegion];
    const categories = [...new Set(products.map(p => p.main_category).filter(Boolean))];
    const subCategories = [...new Set(products.map(p => p.sub_category).filter(Boolean))];

    categoryFilter.innerHTML = '<option value="">All Categories</option>' +
        categories.map(c => `<option value="${c}">${c}</option>`).join('');
    // ...
}
```

**Problem:** Creates intermediate arrays (map, Set, filter) and rebuilds dropdown DOM on every region switch.

**Impact:** Low; only happens on region switch, not every filter.

**Recommendation:** Cache filter options per region on initial load.

---

## Summary Table

| Issue | Severity | Line(s) | Impact |
|-------|----------|---------|--------|
| No search debouncing | High | 306 | Multiple unnecessary renders |
| Full DOM recreation | High | 264 | UI jank, memory churn |
| Multiple filter iterations | Medium | 273-291 | O(3n) instead of O(n) |
| Regex recreation per cell | Medium | 209-212 | CPU/memory overhead |
| Repeated toLowerCase() | Low-Medium | 285-289 | String allocation |
| String concatenation | Low | 228-258 | Minor memory overhead |
| Continuous animation | Low | 33 | Battery drain |
| No virtualization | Low* | - | *Critical if data grows |

---

## Recommended Priority for Fixes

1. **Add debouncing to search input** - Quick win, high impact
2. **Combine filter operations into single pass** - Quick win, medium impact
3. **Cache regex in renderTable scope** - Quick win, medium impact
4. **Pre-compute search text** - Medium effort, cumulative impact
5. **Consider DocumentFragment for DOM updates** - Medium effort, high impact

---

*Analysis generated on 2026-01-16*
