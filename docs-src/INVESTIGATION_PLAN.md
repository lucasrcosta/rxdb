# Mobile Search Dropdown - Systematic Investigation Plan

## Goal
Understand WHY the dropdown disappears at ≤600px (gather facts, not hunches)

---

## Test 1: Does the dropdown element exist in the DOM?

**At 601px width:**
1. Open DevTools
2. Type in search box
3. Inspect DOM - look for `.algolia-autocomplete .ds-dropdown-menu`
4. **Record**: Does element exist? YES/NO

**At 600px width:**
1. Resize to 600px
2. Type in search box
3. Inspect DOM - look for `.algolia-autocomplete .ds-dropdown-menu`
4. **Record**: Does element exist? YES/NO

**Conclusion**:
- If NO at 600px → JavaScript is not creating the element (investigate autocomplete.js)
- If YES at 600px → Element exists but is hidden/positioned wrong (proceed to Test 2)

---

## Test 2: What are the computed styles?

**Only if element exists at 600px:**

In DevTools, select `.algolia-autocomplete .ds-dropdown-menu` and check:

| Property | Value at 601px | Value at 600px |
|----------|---------------|----------------|
| `display` | ? | ? |
| `position` | ? | ? |
| `top` | ? | ? |
| `left` | ? | ? |
| `right` | ? | ? |
| `visibility` | ? | ? |
| `opacity` | ? | ? |
| `z-index` | ? | ? |
| `width` | ? | ? |
| `height` | ? | ? |
| `transform` | ? | ? |

**Record actual computed values from browser**

---

## Test 3: Check bounding rectangle

**Only if element exists at 600px:**

In DevTools console:
```javascript
const dropdown = document.querySelector('.algolia-autocomplete .ds-dropdown-menu');
const rect = dropdown.getBoundingClientRect();
console.log({
  top: rect.top,
  left: rect.left,
  width: rect.width,
  height: rect.height,
  inViewport: rect.top >= 0 && rect.left >= 0
});
```

**Record**: Is dropdown positioned off-screen? Where is it?

### RESULT

{
    "top": 110,
    "left": -382.3999938964844,
    "width": 366.390625,
    "height": 69.859375,
    "inViewport": false
}

---

## Test 4: Check parent overflow/clipping

**Only if element exists at 600px:**

In DevTools console:
```javascript
let el = document.querySelector('.algolia-autocomplete .ds-dropdown-menu');
let parents = [];
while (el.parentElement && parents.length < 5) {
  el = el.parentElement;
  const computed = window.getComputedStyle(el);
  parents.push({
    class: el.className,
    overflow: computed.overflow,
    clipPath: computed.clipPath,
    position: computed.position
  });
}
console.table(parents);
```

**Record**: Which parent has `overflow: hidden` or `clip-path`?

### RESULT

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
        "overflow": "hidden auto",
        "clipPath": "none",
        "position": "static"
    },
    {
        "class": "navbar-sidebar__items navbar-sidebar__items--show-secondary",
        "overflow": "visible",
        "clipPath": "none",
        "position": "static"
    }
]
---

## Test 5: CSS loading order

In DevTools → Network → Filter by CSS:
1. Find `algolia.css` - note order #
2. Find `custom.css` - note order #

**Record**: Which loads first? (Lower number = loads first)

# RESULT

only loads styles.css

---

## Test 6: Minimal override test

Add to `custom.css`:
```css
@media (max-width: 600px) {
  .algolia-autocomplete .ds-dropdown-menu {
    background: red !important; /* Should be VERY visible */
  }
}
```

Rebuild and test at 600px.

**Record**:
- Is dropdown visible with red background? YES/NO
- If NO → Our CSS isn't applying (specificity/order issue)
- If YES → CSS is applying, but other properties are hiding it

### RESULT

As expected, I can't see the element which we proved is in the DOM.

---

## Test 7: Nuclear override test

Add to `custom.css`:
```css
@media (max-width: 600px) {
  .algolia-autocomplete .ds-dropdown-menu {
    all: unset !important;
    display: block !important;
    position: static !important;
    background: yellow !important;
    border: 5px solid red !important;
    padding: 20px !important;
  }
}
```

**Record**: Is dropdown visible now? This removes ALL styles.

### RESULT

As expected, I can't see the element which we proved is in the DOM.

---

## Test 8: Check JavaScript behavior

In `SearchBar/index.jsx`, add logging at line 92:
```javascript
initAlgolia(searchDocs, searchIndex, DocSearch, options);
console.log('SEARCH INITIALIZED', {
  width: window.innerWidth,
  isMobile: window.innerWidth <= 600
});
setIndexReady(true);
```

**Record**: Does initialization happen differently at mobile width?

### RESULT

SEARCH INITIALIZED {width: 460, isMobile: true}

---

## Results Template

Fill this out after running tests:

```
Test 1: Dropdown exists at 600px? [YES/NO]
Test 2: Computed position at 600px: [value]
Test 3: Dropdown bounding rect: [values]
Test 4: Parent causing clipping: [element class]
Test 5: CSS load order: algolia.css=[#], custom.css=[#]
Test 6: Red background visible? [YES/NO]
Test 7: Yellow background visible? [YES/NO]
Test 8: Any differences in initialization? [description]
```

---

## Next Steps

**Once we have facts, we can:**
1. Determine if it's CSS specificity, JavaScript, or DOM structure
2. Create a targeted fix based on evidence
3. Avoid guesswork

