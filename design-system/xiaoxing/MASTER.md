# Design System Master File

> **LOGIC:** When building a specific page, first check `design-system/pages/[page-name].md`.
> If that file exists, its rules **override** this Master file.
> If not, strictly follow the rules below.

---

**Project:** Xiaoxing
**Generated:** 2026-08-22 12:27:55
**Category:** Magazine/Blog
**Design Dials:** Variance 6/10 (Balanced / Modern) | Motion 3/10 (Subtle) | Density 4/10 (Standard)

---

## Global Rules

### Color Palette

| Role | Hex | CSS Variable |
|------|-----|--------------|
| Primary | `#18181B` | `--color-primary` |
| On Primary | `#FFFFFF` | `--color-on-primary` |
| Secondary | `#3F3F46` | `--color-secondary` |
| On Secondary | `#FFFFFF` | `--color-on-secondary` |
| Accent/CTA | `#BE185D` | `--color-accent` |
| On Accent/CTA | `#FFFFFF` | `--color-on-accent` |
| Background | `#FAFAFA` | `--color-background` |
| Foreground | `#09090B` | `--color-foreground` |
| Card | `#FFFFFF` | `--color-card` |
| Card Foreground | `#09090B` | `--color-card-foreground` |
| Muted | `#E8ECF0` | `--color-muted` |
| Muted Foreground | `#475569` | `--color-muted-foreground` |
| Border | `#E4E4E7` | `--color-border` |
| Destructive | `#DC2626` | `--color-destructive` |
| On Destructive | `#FFFFFF` | `--color-on-destructive` |
| Ring | `#18181B` | `--color-ring` |

**Color Notes:** Editorial black + accessible raspberry accent. Dark mode uses `#F472B6` with `#09090B` text.

### Typography

- **Heading Font:** Nacelle (Semibold 600, reused from the Cruip Open template)
- **Display Accent:** Libre Bodoni (logo and manifesto)
- **Body Font:** Public Sans
- **Mood:** magazine, editorial, publishing, refined, journalism, print
- **Delivery:** Self-hosted WOFF2 files under `/static/fonts/`; do not add Google Fonts network requests.

**CSS setup:**
```css
@font-face {
  font-family: "Libre Bodoni";
  src: url("/fonts/libre-bodoni-latin-normal.woff2") format("woff2");
  font-style: normal;
  font-weight: 500 600;
  font-display: swap;
}

@font-face {
  font-family: "Nacelle";
  src: url("/fonts/nacelle-regular.woff2") format("woff2");
  font-style: normal;
  font-weight: 400;
  font-display: swap;
}

@font-face {
  font-family: "Nacelle";
  src: url("/fonts/nacelle-semibold.woff2") format("woff2");
  font-style: normal;
  font-weight: 600;
  font-display: swap;
}
```

### Spacing Variables

*Density: 4/10 — Standard*

| Token | Value | Usage |
|-------|-------|-------|
| `--space-xs` | `4px` / `0.25rem` | Tight gaps |
| `--space-sm` | `8px` / `0.5rem` | Icon gaps, inline spacing |
| `--space-md` | `16px` / `1rem` | Standard padding |
| `--space-lg` | `24px` / `1.5rem` | Section padding |
| `--space-xl` | `32px` / `2rem` | Large gaps |
| `--space-2xl` | `48px` / `3rem` | Section margins |
| `--space-3xl` | `64px` / `4rem` | Hero padding |

### Shadow Depths

| Level | Value | Usage |
|-------|-------|-------|
| `--shadow-sm` | `0 1px 2px rgba(0,0,0,0.05)` | Subtle lift |
| `--shadow-md` | `0 4px 6px rgba(0,0,0,0.1)` | Cards, buttons |
| `--shadow-lg` | `0 10px 15px rgba(0,0,0,0.1)` | Modals, dropdowns |
| `--shadow-xl` | `0 20px 25px rgba(0,0,0,0.15)` | Hero images, featured cards |

---

## Component Specs

### Buttons

```css
/* Primary Button */
.btn-primary {
  min-height: 50px;
  background: #18181B;
  color: white;
  border: 1px solid #18181B;
  border-radius: 0;
  padding: 0 18px;
  font-weight: 600;
  transition: color 180ms ease, background 180ms ease, transform 180ms ease;
  cursor: pointer;
}

.btn-primary:hover {
  color: white;
  background: #BE185D;
  border-color: #BE185D;
  transform: translateY(-2px);
}

/* Secondary Button */
.btn-secondary {
  min-height: 50px;
  background: transparent;
  color: #18181B;
  border: 1px solid #18181B;
  padding: 0 18px;
  border-radius: 0;
  font-weight: 600;
  transition: color 180ms ease, background 180ms ease, transform 180ms ease;
  cursor: pointer;
}
```

### Cards

```css
.card {
  background: transparent;
  border: 1px solid #DEDEE3;
  border-radius: 0;
  padding: 32px;
  box-shadow: none;
  transition: color 180ms ease, background 180ms ease, border-color 180ms ease;
}

.card:hover {
  background: color-mix(in srgb, #BE185D 7%, transparent);
  border-color: #BE185D;
}
```

### Inputs

```css
.input {
  padding: 12px 16px;
  border: 1px solid #E2E8F0;
  border-radius: 8px;
  font-size: 16px;
  transition: border-color 200ms ease;
}

.input:focus {
  border-color: #18181B;
  outline: none;
  box-shadow: 0 0 0 3px #18181B20;
}
```

### Modals

```css
.modal-overlay {
  background: rgba(0, 0, 0, 0.5);
  backdrop-filter: blur(4px);
}

.modal {
  background: white;
  border-radius: 16px;
  padding: 32px;
  box-shadow: var(--shadow-xl);
  max-width: 500px;
  width: 90%;
}
```

---

## Style Guidelines

**Style:** Swiss Modernism 2.0 + Cruip Open-inspired glow system

**Keywords:** Grid system, Helvetica, modular, asymmetric, international style, rational, clean, mathematical spacing, indigo glow, rounded panels, spotlight interaction

**Reference:** [Cruip Open React template](https://github.com/cruip/open-react-template) — visual patterns adapted to Hugo/PaperMod rather than copied as a React app.

**Best For:** Corporate sites, architecture, editorial, SaaS, museums, professional services, documentation

**Key Effects:** display: grid, grid-template-columns: repeat(12 1fr), gap: 1rem, mathematical ratios, clear hierarchy

### Page Pattern

**Pattern Name:** Editorial Homepage + Content Index

- **Content Strategy:** Lead with a concise personal statement and make the newest writing immediately discoverable. Preserve a complete DOM reading order without JavaScript.
- **CTA Placement:** Hero actions, category map, learning path, workflow panel, then a final reading CTA before the latest-post index.
- **Section Order:** Identity/navigation > Editorial hero > Category map > Content data dashboard > Learning path > Workflow > CTA > Latest posts > Footer.
- **Dashboard Pattern:** Category counts use proportional bars; the annual activity map uses a five-step numeric legend, tooltips, and an accessible data table instead of color alone.
- **Directory Pattern:** Taxonomy terms use bordered index cards; the tag directory adds regex search, a category select, and visible category badges. Empty taxonomies show an explanation and a clear action instead of blank space.
- **Interaction Pattern:** Articles expose an anchored Utterances comment panel and a like control. The homepage aggregates comment and like totals without exposing article names for likes; a shared likes endpoint is optional for static hosting.

---

## Motion

**Scroll Reveal** (Subtle) — Trigger: scroll (viewport enter) | Duration: 300-400ms | Easing: `power1.out`

```js
gsap.from(el, { opacity: 0, y: 12, duration: 0.35, ease: 'power1.out', scrollTrigger: { trigger: el, start: 'top 90%', toggleActions: 'play none none reverse' } });
```

**Framework notes:** Requires the ScrollTrigger plugin registered once via gsap.registerPlugin(ScrollTrigger); Use matchMedia('(prefers-reduced-motion: reduce)') to skip non-essential motion and render the final state immediately

- ✅ Keep the y offset small (8-16px) so it reads as a fade, not a slide
- ❌ Don't reveal below-the-fold content needed for SEO/crawlers as invisible-by-default without a no-JS fallback
- ⚡ toggleActions 'play none none reverse' avoids re-triggering on every scroll direction change

---

## Anti-Patterns (Do NOT Use)

- ❌ Poor typography
- ❌ Slow loading

### Additional Forbidden Patterns

- ❌ **Emojis as icons** — Use SVG icons (Heroicons, Lucide, Simple Icons)
- ❌ **Missing cursor:pointer** — All clickable elements must have cursor:pointer
- ❌ **Layout-shifting hovers** — Avoid scale transforms that shift layout
- ❌ **Low contrast text** — Maintain 4.5:1 minimum contrast ratio
- ❌ **Instant state changes** — Always use transitions (150-300ms)
- ❌ **Invisible focus states** — Focus states must be visible for a11y

---

## Pre-Delivery Checklist

Before delivering any UI code, verify:

- [ ] No emojis used as icons (use SVG instead)
- [ ] All icons from consistent icon set (Heroicons/Lucide)
- [ ] `cursor-pointer` on all clickable elements
- [ ] Hover states with smooth transitions (150-300ms)
- [ ] Light mode: text contrast 4.5:1 minimum
- [ ] Focus states visible for keyboard navigation
- [ ] `prefers-reduced-motion` respected
- [ ] Responsive: 375px, 768px, 1024px, 1440px
- [ ] No content hidden behind fixed navbars
- [ ] No horizontal scroll on mobile
