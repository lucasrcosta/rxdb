# Fix: Search Dropdown Clipping & Positioning

## 1. Root Cause Analysis
The search dropdown was being clipped because it was rendered inside the sidebar container, which has a `clip-path` property set by the Docusaurus theme.
*   **Culprit:** `.theme-doc-sidebar-container` (specifically the internal `docSidebarContainer` class) has `clip-path: inset(0)`.
*   **Reason:** This style is used by Docusaurus to mask content during the sidebar collapse/expand animation.
*   **Secondary Issue:** The dropdown also needed to be in a higher stacking context to appear above the main page content.

## 2. The Solution
We implemented a minimal fix that addresses the clipping while keeping the dropdown anchored to the search bar.

### Changes

#### 1. JS Configuration (`SearchBar/index.jsx`)
We updated the `autocompleteOptions` to append the dropdown to the search bar's container instead of `body`. This ensures the dropdown moves with the header and maintains the correct width.

```javascript
autocompleteOptions: {
  hint: false,
  appendTo: '.navbar__search', // Critical for correct positioning and width
  debug: true
}
```

#### 2. CSS Override (`custom.css`)
We applied a targeted override to the sidebar container to remove the clipping and adjust the stacking order.

```css
/* Fix for search dropdown clipping and stacking context */
.theme-doc-sidebar-container {
  clip-path: none !important;
  z-index: 10;
}
```

## 3. Trade-offs & Justification

### `clip-path: none`
*   **Trade-off:** Disables the content masking that happens during the sidebar collapse animation.
*   **Justification:** This documentation site **does not use the collapsible sidebar feature** (the toggle is not exposed). Since the animation never triggers, the `clip-path` is effectively unused code. Removing it is harmless and solves the clipping issue without side effects.

### `z-index: 10`
*   **Trade-off:** Creates a new stacking context.
*   **Justification:** A minimal `z-index` of `10` is sufficient to place the sidebar container (and thus the overflowing dropdown) above the main content area, ensuring the search results are fully visible.
