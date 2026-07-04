# Topic 15 — HTML5 & CSS3

> Modern HTML and CSS are not just markup — they're tools for accessibility, performance, and rich UIs.

---

## HTML5 Semantic Elements

Semantic HTML tells the browser AND screen readers what content means, not just how it looks.

```html
<!-- ❌ Non-semantic — div soup: -->
<div class="header">
    <div class="nav">
        <div class="nav-item">Home</div>
    </div>
</div>
<div class="main">
    <div class="article">
        <div class="article-heading">My Post</div>
    </div>
</div>

<!-- ✅ Semantic HTML5: -->
<header>
    <nav>
        <ul>
            <li><a href="/">Home</a></li>
            <li><a href="/about">About</a></li>
        </ul>
    </nav>
</header>

<main>
    <article>
        <header>
            <h1>My Blog Post</h1>
            <time datetime="2026-07-05">July 5, 2026</time>
        </header>
        <section>
            <h2>Introduction</h2>
            <p>Content here...</p>
        </section>
        <aside>Related links</aside>
        <footer>Author info</footer>
    </article>
</main>

<footer>
    <address>Contact: me@example.com</address>
</footer>
```

### Key Semantic Elements
| Element | Purpose |
|---|---|
| `<header>` | Introductory content for page or section |
| `<nav>` | Navigation links |
| `<main>` | Primary content (one per page) |
| `<article>` | Self-contained, redistributable content |
| `<section>` | Thematically grouped content |
| `<aside>` | Tangentially related content (sidebar) |
| `<footer>` | Closing content for page or section |
| `<figure>` + `<figcaption>` | Media with caption |
| `<time>` | Machine-readable date/time |
| `<mark>` | Highlighted/relevant text |

---

## HTML5 Forms & Validation

```html
<form novalidate>  <!-- disable browser default UI, use custom JS -->
    <!-- Input types with built-in validation: -->
    <input type="email"  required placeholder="Email" />
    <input type="url"    placeholder="Website" />
    <input type="number" min="0" max="1000000" step="100" />
    <input type="date"   min="2020-01-01" />
    <input type="range"  min="1" max="10" value="5" />
    <input type="color"  value="#8b5cf6" />
    <input type="tel"    pattern="[0-9]{11}" placeholder="09XXXXXXXXX" />
    <input type="search" />
    <input type="password" minlength="8" autocomplete="new-password" />
    
    <!-- Datalist — autocomplete suggestions: -->
    <input list="currencies" placeholder="Currency" />
    <datalist id="currencies">
        <option value="PHP">Philippine Peso</option>
        <option value="USD">US Dollar</option>
        <option value="SAR">Saudi Riyal</option>
    </datalist>
    
    <!-- Grouped fields: -->
    <fieldset>
        <legend>Account Type</legend>
        <label><input type="radio" name="type" value="savings" /> Savings</label>
        <label><input type="radio" name="type" value="checking" /> Checking</label>
    </fieldset>
</form>
```

---

## HTML5 APIs

```javascript
// Local Storage & Session Storage:
localStorage.setItem("theme", "dark");
const theme = localStorage.getItem("theme"); // "dark"
localStorage.removeItem("theme");

// sessionStorage — cleared when tab closes:
sessionStorage.setItem("draft", JSON.stringify(formData));

// Web Workers — run JS in background thread:
const worker = new Worker("heavy-computation.js");
worker.postMessage({ data: largeArray });
worker.onmessage = (e) => console.log("Result:", e.data);

// Intersection Observer — detect visibility (infinite scroll, lazy load):
const observer = new IntersectionObserver((entries) => {
    entries.forEach(entry => {
        if (entry.isIntersecting) {
            entry.target.src = entry.target.dataset.src; // lazy load image
            observer.unobserve(entry.target);
        }
    });
}, { threshold: 0.1 });
document.querySelectorAll("img[data-src]").forEach(img => observer.observe(img));

// Custom Elements (Web Components):
class TransactionCard extends HTMLElement {
    connectedCallback() {
        this.innerHTML = `<div class="card">${this.getAttribute("amount")}</div>`;
    }
}
customElements.define("transaction-card", TransactionCard);
// <transaction-card amount="500"></transaction-card>

// Drag and Drop API:
element.setAttribute("draggable", "true");
element.addEventListener("dragstart", (e) => e.dataTransfer.setData("text", element.id));
dropZone.addEventListener("drop", (e) => {
    const id = e.dataTransfer.getData("text");
    dropZone.appendChild(document.getElementById(id));
});
```

---

## CSS3 Box Model

```css
/* Box model — total space an element takes: */
/* content + padding + border + margin */

.box {
    width: 300px;           /* content width */
    padding: 20px;          /* space inside border */
    border: 2px solid;      /* border */
    margin: 10px;           /* space outside border */
    
    /* box-sizing: content-box (default) — width = content only */
    /* Total = 300 + 40 + 4 = 344px */
    
    box-sizing: border-box; /* width INCLUDES padding + border */
    /* Total = 300px exactly — ALWAYS use this */
}

/* Global reset: */
*, *::before, *::after { box-sizing: border-box; }
```

---

## Flexbox

```css
.flex-container {
    display: flex;
    flex-direction: row;          /* row | column | row-reverse | column-reverse */
    justify-content: space-between; /* main axis: flex-start|center|flex-end|space-between|space-around|space-evenly */
    align-items: center;          /* cross axis: flex-start|center|flex-end|stretch|baseline */
    flex-wrap: wrap;              /* wrap to next line if overflow */
    gap: 16px;                    /* space between items */
}

.flex-item {
    flex-grow: 1;   /* how much extra space it takes (0 = none) */
    flex-shrink: 1; /* how much it shrinks when short on space */
    flex-basis: auto; /* initial size before grow/shrink */
    flex: 1;        /* shorthand: grow=1 shrink=1 basis=0 */
    
    align-self: flex-end; /* override container's align-items for this item */
    order: 2;             /* visual order (default 0) */
}

/* Common patterns: */
/* Perfect center: */
.center { display: flex; justify-content: center; align-items: center; }

/* Sidebar + content: */
.layout { display: flex; }
.sidebar { flex: 0 0 250px; } /* fixed width */
.content { flex: 1; }         /* takes remaining space */

/* Navbar: */
.nav { display: flex; justify-content: space-between; align-items: center; }
```

---

## CSS Grid

```css
/* Grid — 2D layout system: */
.grid-container {
    display: grid;
    grid-template-columns: repeat(3, 1fr);      /* 3 equal columns */
    grid-template-columns: 250px 1fr 200px;     /* fixed | flex | fixed */
    grid-template-rows: auto 1fr auto;
    gap: 16px;
    
    /* Named template areas: */
    grid-template-areas:
        "header header header"
        "sidebar main aside"
        "footer footer footer";
}

/* Place items in named areas: */
.header  { grid-area: header; }
.sidebar { grid-area: sidebar; }
.main    { grid-area: main; }
.aside   { grid-area: aside; }
.footer  { grid-area: footer; }

/* Span cells: */
.featured {
    grid-column: 1 / 3;    /* span from col 1 to 3 */
    grid-row: 1 / 3;       /* span from row 1 to 3 */
    /* or: grid-column: span 2; */
}

/* Auto-responsive grid without media queries: */
.card-grid {
    display: grid;
    grid-template-columns: repeat(auto-fill, minmax(280px, 1fr));
    /* fills as many 280px columns as fit, stretch to fill space */
}
```

---

## CSS Custom Properties (Variables) & Modern CSS

```css
:root {
    /* Design tokens: */
    --color-primary: #8b5cf6;
    --color-primary-dark: #7c3aed;
    --color-bg: #0f0f1a;
    --color-text: #e2e8f0;
    --spacing-sm: 8px;
    --spacing-md: 16px;
    --spacing-lg: 24px;
    --radius-card: 12px;
    --shadow-card: 0 4px 20px rgba(0, 0, 0, 0.3);
}

/* Dark/light theme toggle: */
[data-theme="light"] {
    --color-bg: #ffffff;
    --color-text: #1a1a2e;
}

.card {
    background: var(--color-bg);
    color: var(--color-text);
    border-radius: var(--radius-card);
    box-shadow: var(--shadow-card);
    padding: var(--spacing-md);
}

/* CSS Nesting (modern CSS — no preprocessor needed): */
.card {
    background: var(--color-bg);
    
    &:hover { transform: translateY(-2px); }
    
    & .card-title { font-size: 1.25rem; color: var(--color-primary); }
    
    &.highlighted { border: 2px solid var(--color-primary); }
}

/* Container queries — responsive to PARENT, not viewport: */
@container (min-width: 600px) {
    .card { display: grid; grid-template-columns: 1fr 2fr; }
}
.card-wrapper { container-type: inline-size; }

/* CSS :has() — "parent selector": */
.form-group:has(input:invalid) label { color: red; }
.nav:has(.dropdown:hover) { z-index: 100; }
```

---

## Responsive Design & Media Queries

```css
/* Mobile-first approach: */
.container { width: 100%; padding: 16px; }

/* Tablet: */
@media (min-width: 768px) {
    .container { max-width: 768px; margin: 0 auto; }
    .grid { grid-template-columns: repeat(2, 1fr); }
}

/* Desktop: */
@media (min-width: 1024px) {
    .container { max-width: 1280px; }
    .grid { grid-template-columns: repeat(3, 1fr); }
}

/* Feature queries: */
@supports (display: grid) {
    .layout { display: grid; }
}

/* Preference queries: */
@media (prefers-color-scheme: dark) { :root { --color-bg: #0f0f1a; } }
@media (prefers-reduced-motion: reduce) { * { animation: none !important; } }
@media print { nav, .ads { display: none; } }
```

---

## CSS Animations & Transitions

```css
/* Transitions — smooth state changes: */
.button {
    background: var(--color-primary);
    transition: background 200ms ease, transform 150ms ease;
}
.button:hover {
    background: var(--color-primary-dark);
    transform: scale(1.02);
}

/* Keyframe animations: */
@keyframes fadeSlideIn {
    from { opacity: 0; transform: translateY(20px); }
    to   { opacity: 1; transform: translateY(0); }
}

.modal {
    animation: fadeSlideIn 300ms ease-out;
}

/* Loading spinner: */
@keyframes spin {
    to { transform: rotate(360deg); }
}
.spinner {
    width: 24px; height: 24px;
    border: 3px solid rgba(139, 92, 246, 0.2);
    border-top-color: var(--color-primary);
    border-radius: 50%;
    animation: spin 0.8s linear infinite;
}

/* Performance — use only transform and opacity for animations */
/* (composited by GPU — doesn't trigger reflow/repaint): */
/* ✅ transform, opacity */
/* ❌ width, height, top, left, margin (triggers layout) */
```

---

## Accessibility (a11y)

```html
<!-- ARIA attributes: -->
<button aria-label="Close dialog" aria-expanded="false">×</button>
<div role="dialog" aria-modal="true" aria-labelledby="dialog-title">
    <h2 id="dialog-title">Confirm Delete</h2>
</div>

<!-- Images: -->
<img src="chart.png" alt="Monthly spending chart showing ₱5,000 in July" />
<img src="decorative-divider.png" alt="" role="presentation" /> <!-- decorative -->

<!-- Focus management: -->
<!-- Tab order follows DOM order — use tabindex="0" for custom interactive elements -->
<!-- tabindex="-1" removes from tab order but keeps focusable via JS -->
<div tabindex="0" role="button" onkeydown="handleKeyDown">Custom Button</div>

<!-- Skip navigation: -->
<a href="#main-content" class="skip-link">Skip to main content</a>
```

```css
/* Focus visible styles — never remove outline without replacement: */
:focus-visible {
    outline: 3px solid var(--color-primary);
    outline-offset: 2px;
}

/* Hide visually but keep for screen readers: */
.sr-only {
    position: absolute;
    width: 1px; height: 1px;
    padding: 0; margin: -1px;
    overflow: hidden;
    clip: rect(0, 0, 0, 0);
    white-space: nowrap;
    border: 0;
}
```

---

## Interview Q&A

**Q1: What is the difference between `display: none`, `visibility: hidden`, and `opacity: 0`?**
> `display: none` — removes element from layout, takes no space, inaccessible to screen readers. `visibility: hidden` — element is invisible but still takes space in layout. `opacity: 0` — fully transparent but still takes space, still accessible/interactive. For animations, use `opacity`; for show/hide toggling, use `display` or `visibility`.

**Q2: What is the CSS specificity order?**
> Inline styles (1000) > ID selectors (100) > Class/attribute/pseudo-class selectors (10) > Element/pseudo-element selectors (1) > Universal selector (0). `!important` overrides all but is considered bad practice.

**Q3: What is `position: relative` vs `position: absolute`?**
> `relative` — positioned relative to its normal flow position; can use `top/left/right/bottom` as offsets without removing from flow. `absolute` — removed from normal flow, positioned relative to nearest `position: relative/absolute/fixed` ancestor. `fixed` — positioned relative to viewport. `sticky` — relative until it hits a scroll threshold, then acts like fixed.

**Q4: What is the difference between Flexbox and Grid?**
> Flexbox is **one-dimensional** (row OR column). Grid is **two-dimensional** (rows AND columns simultaneously). Use Flexbox for component-level layout (navbars, card internals, alignment). Use Grid for page-level layout (overall page structure, card grids, complex 2D arrangements).

**Q5: What is semantic HTML and why does it matter?**
> Semantic elements (`<article>`, `<nav>`, `<aside>`, etc.) describe the meaning of content, not just its appearance. Benefits: better accessibility (screen readers navigate by landmarks), improved SEO (search engines understand structure), more maintainable code, browser reading mode support.
