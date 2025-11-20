# Mobile Search Dropdown - Complete Investigation Report

## Executive Summary

**Problem**: Search dropdown disappears at viewport widths ≤600px
**Root Cause**: CSS over-constraint - element has both `left: 0px` AND `right: 32px` set simultaneously on a `position: fixed` element
**Status**: CSS fix applies correctly (`computedLeft: "0px"`), but element still renders off-screen at `getBoundingClientRect().left: -375px`

---

## Investigation Timeline

### Test 1: Element Existence ✅
**Result**: Dropdown element EXISTS in DOM at ≤600px

### Test 2: Computed Styles
**Result at 600px**:
- `position: fixed`
- `left: auto` (later overridden to `0px` by our fix)
- `right: 1rem`
- `top: 50px`
- `z-index: 100`

### Test 3: Bounding Rectangle ❌
```javascript
{
  "top": 110,
  "left": -382.3999938964844,  // OFF-SCREEN LEFT
  "width": 366.390625,
  "height": 69.859375,
  "inViewport": false
}
```

**Finding**: Element positioned ~380px to the LEFT of viewport (off-screen)

### Test 4: Parent Chain Analysis
```javascript
[
  {
    "class": "algolia-autocomplete algolia-autocomplete-left",
    "overflow": "visible",
    "clipPath": "none",
    "position": "absolute"
  },
  {
    "class": "navbar__search",
    "overflow": "visible",
    "clipPath": "none",
    "position": "static"
  },
  {
    "class": "theme-doc-sidebar-menu menu__list",
    "overflow": "visible",
    "clipPath": "none",
    "position": "static"
  },
  {
    "class": "theme-layout-navbar-sidebar-panel navbar-sidebar__item menu",
    "overflow": "hidden auto",  // ← Has overflow hidden
    "clipPath": "none",
    "position": "static"
  }
]
```

**Finding**: Parent `.algolia-autocomplete-left` has inline style `left: 515.969px` set by JavaScript

### Test 5: CSS Loading Order
**Result**: All CSS bundled into single `styles.css` file by Docusaurus

**Critical Finding**:
- `custom.css` content appears BEFORE `algolia.css` in bundle
- When both rules use `!important` with equal specificity, LATER rule wins
- Algolia.css mobile styles override custom.css unless custom.css has HIGHER specificity

### Test 6 & 7: CSS Override Tests ❌
Nuclear CSS overrides (red background, yellow background) did NOT make element visible

**Reason**: Element positioned off-screen regardless of styling

---

## CSS Cascade Analysis

### Algolia.css Default (Mobile ≤600px)
**Location**: styles.css:8821
**Selector**: `.algolia-autocomplete .ds-dropdown-menu`
**Specificity**: 0,0,2,0

```css
@media (max-width: 600px) {
  .algolia-autocomplete .ds-dropdown-menu {
    z-index: 100;
    position: fixed !important;
    top: 50px !important;
    left: auto !important;
    right: 1rem !important;
    width: 600px;
    max-width: calc(100% - 2rem);
    max-height: calc(100% - 5rem);
    display: block;
  }
}
```

### Our CSS Fix (Attempt 1) ❌
**Location**: styles.css:7575
**Selector**: `.algolia-autocomplete .ds-dropdown-menu`
**Specificity**: 0,0,2,0
**Result**: OVERRIDDEN (equal specificity, but loads before algolia.css)

```css
@media (max-width: 600px) {
  .algolia-autocomplete .ds-dropdown-menu {
    left: 0 !important;
    right: auto !important;
  }
}
```

### Our CSS Fix (Attempt 2) ✅ (CSS Applies, but element still off-screen)
**Location**: styles.css:7579
**Selector**: `.algolia-autocomplete.algolia-autocomplete-left .ds-dropdown-menu`
**Specificity**: 0,0,3,0 - HIGHER than algolia.css
**Result**: CSS APPLIES (`computedLeft: "0px"`), but element still renders at `-375px`

```css
@media (max-width: 600px) {
  .algolia-autocomplete.algolia-autocomplete-left {
    left: 0 !important;
  }

  .algolia-autocomplete.algolia-autocomplete-left .ds-dropdown-menu {
    left: 0 !important;
    right: auto !important;
  }
}
```

---

## The Mystery: Computed vs Rendered Position

### Final Diagnostic Results

**Viewport**: 453px
**Dropdown Width**: 344px

**Computed Styles** (from `window.getComputedStyle`):
```javascript
{
  computedLeft: "0px",      // ✅ Our CSS is working!
  computedRight: "32px",    // From algolia.css (2rem)
  position: "fixed"
}
```

**Rendered Position** (from `getBoundingClientRect`):
```javascript
{
  left: -375.989990234375,  // ❌ Still off-screen!
  width: 344
}
```

### The Over-Constraint Problem

The element has BOTH:
- `left: 0px` (from our CSS)
- `right: 32px` (from algolia.css at line 8821)

For a `position: fixed` element, having both `left` and `right` set creates an **over-constrained** situation.

**Expected behavior**:
- If `left: 0` and `right: 32px` on 453px viewport
- Available width: 453 - 32 - 0 = 421px
- Element is 344px wide → should fit
- Should be positioned at left: 0

**Actual behavior**:
- Element renders at left: -375px (off-screen)
- Browser appears to be resolving the constraint differently

---

## JavaScript Interaction

### DocSearch.js `handleShown` Method

**File**: `docusaurus-lunr-search-main/src/theme/SearchBar/DocSearch.js`
**Lines**: 283-307

```javascript
handleShown(input) {
  const middleOfInput = input.offset().left + input.width() / 2;
  let middleOfWindow = $(document).width() / 2;

  const alignClass =
    middleOfInput - middleOfWindow >= 0
      ? "algolia-autocomplete-right"
      : "algolia-autocomplete-left";  // ← Applied on mobile

  const autocompleteWrapper = $(".algolia-autocomplete");
  if (!autocompleteWrapper.hasClass(alignClass)) {
    autocompleteWrapper.addClass(alignClass);
  }
}
```

**What it does**:
- Calculates whether search input is in left or right half of window
- On mobile sidebar, input is in left half
- Adds `algolia-autocomplete-left` class
- This class gets inline `left: 515px` style (source unknown)

---

## Architecture Understanding

### Desktop (≥601px)
```
DocSidebar/Desktop/Content/index.tsx
  └─ <div style={{padding: 10, ...}}>
      └─ <SearchBar />
          └─ .navbar__search
              └─ .algolia-autocomplete (position: relative)
                  └─ .ds-dropdown-menu (position: relative, top: -6px)
```

**Positioning**: Relative to parent sidebar container
**Works**: ✅ Dropdown appears below search input

### Mobile (≤600px)
```
DocSidebar/Mobile/index.tsx
  └─ NavbarSecondaryMenuFiller (Docusaurus mobile drawer)
      └─ <ul className="menu__list">
          └─ <SearchBar />
              └─ .navbar__search
                  └─ .algolia-autocomplete.algolia-autocomplete-left
                      │  (position: absolute, left: 515.969px ← set by JS)
                      └─ .ds-dropdown-menu
                          (position: fixed, left: 0px, right: 32px)
```

**Positioning**: Fixed to viewport (not relative to sidebar)
**Problem**: Over-constrained (`left: 0` + `right: 32px`), renders at `-375px`

---

## Current Status

### What We Know:
1. ✅ Dropdown element EXISTS in DOM
2. ✅ Our CSS IS APPLYING (`computedLeft: "0px"`)
3. ❌ Element STILL RENDERS OFF-SCREEN (`getBoundingClientRect().left: -375px`)
4. ⚠️ Over-constraint: Both `left` and `right` properties set simultaneously
5. ⚠️ Parent `.algolia-autocomplete` has inline `left: 515.969px` from JavaScript

### What We Don't Know:
1. **Why** does `computedLeft: "0px"` not match `boundingRect.left: -375px`?
2. **How** is the browser resolving the `left: 0` + `right: 32px` conflict?
3. **Is** there a scroll container or transform we're missing?
4. **Should** we remove `right: 32px` instead of setting `left: 0`?

---

## Next Steps to Try

### Option 1: Remove Right Constraint
Instead of setting `left: 0`, remove the `right` property:

```css
@media (max-width: 600px) {
  .algolia-autocomplete.algolia-autocomplete-left .ds-dropdown-menu {
    left: 0 !important;
    right: unset !important;  /* Remove the right constraint */
  }
}
```

### Option 2: Use Width Instead
Set explicit width and only one side:

```css
@media (max-width: 600px) {
  .algolia-autocomplete.algolia-autocomplete-left .ds-dropdown-menu {
    left: 1rem !important;
    right: unset !important;
    width: calc(100% - 2rem) !important;
  }
}
```

### Option 3: Change Position Context
Change to relative positioning (like desktop):

```css
@media (max-width: 600px) {
  .algolia-autocomplete.algolia-autocomplete-left .ds-dropdown-menu {
    position: relative !important;
    left: 0 !important;
    right: auto !important;
    top: -6px !important;
  }
}
```

### Option 4: Fix Parent Positioning
Target the parent's inline style:

```css
@media (max-width: 600px) {
  .algolia-autocomplete.algolia-autocomplete-left {
    left: 0 !important;
    right: 0 !important;
    position: static !important;
  }
}
```

---

## Files Modified

1. **`docs-src/src/css/custom.css`** (lines 4466-4478)
   - Added mobile search dropdown fix with higher specificity
   - Currently applies (`computedLeft: "0px"`) but element still off-screen

---

## Tools Created

1. **`search-debug.js`** - Debug helper script
2. **`INVESTIGATION_PLAN.md`** - Systematic test plan (this file)

---

## Lessons Learned

1. **CSS bundling order matters** - Custom CSS loads before algolia.css
2. **Specificity must be higher** - Need 3+ classes to override algolia.css
3. **`!important` + later source wins** - When specificity is equal
4. **Computed ≠ Rendered** - `getComputedStyle` shows CSS, `getBoundingClientRect` shows reality
5. **Over-constraint matters** - Setting both `left` and `right` on fixed elements is problematic
6. **DevTools is essential** - Only way to see actual CSS cascade

---

## Key Debugging Commands

```javascript
// Check if element exists
const dropdown = document.querySelector('.algolia-autocomplete .ds-dropdown-menu');

// Check computed styles
const computed = window.getComputedStyle(dropdown);
console.log(computed.left, computed.right, computed.position);

// Check actual position
const rect = dropdown.getBoundingClientRect();
console.log(rect.left, rect.width);

// Check viewport
console.log(window.innerWidth);

// Check parent chain
let el = dropdown;
while (el && el !== document.body) {
  console.log(el.className, window.getComputedStyle(el).position);
  el = el.parentElement;
}
```

---

**Last Updated**: Investigation ongoing
**Next Action**: Try Option 1 (remove right constraint with `right: unset !important`)
