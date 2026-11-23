
# 🎨 DESIGN SYSTEM & UI RULES

## 🧱 1. Layout & Spacing
- **8pt Grid System**: All margins, paddings, gaps, and structural heights must use multiples of 8px (4px, 8px, 12px, 16px, 24px, 32px, 48px, 64px).
- **Whitespace**: Prioritize generous whitespace to separate sections; do not crowd elements.
- **Max Width**: Standard layouts should use `max-w-7xl` (1280px) for pages and `max-w-3xl` (768px) for article/content flows.

## 🔤 2. Typography
- **Font Stack**: Use modern, clean sans-serif families (e.g., Geist Sans, SF Pro, Inter).
- **Scale & Contrast**:
  - Display/H1: `text-4xl` to `text-5xl`, font-bold, line-height compact (`leading-tight`).
  - Headings (H2/H3): `text-xl` to `text-2xl`, font-semibold.
  - Body Text: `text-base` (16px) for main reading, `text-sm` (14px) for captions/subtext.
- **Readability**: Body line-height must be comfortable (`leading-relaxed`, ~1.6). Text color must meet WCAG AA contrast standards.

## 🎨 3. Color Tokens & Semantic Usage
Always map colors to semantic tokens instead of using hardcoded arbitrary hex values:
- **Surface**: Backgrounds must use clean whites, off-whites (`#fafafa`), or deep dark grays (`#09090b`), never pure pitch black.
- **Text**: Primary text must be high-contrast (Zinc-900 or Zinc-50); secondary text must be muted (Zinc-500 or Zinc-400).
- **Accent**: Use a single corporate accent color (e.g., Deep Navy, Slate, or Emerald) sparingly for call-to-actions, active states, and highlights.
- **Borders**: Dividers and borders must be subtle and thin (`1px` width with high transparency or light gray/zinc values).

## ✨ 4. Component Styling & Visuals
- **Borders & Radii**:
  - Buttons & Inputs: Clean, slightly rounded corners (`rounded-md` / `6px` to `8px`).
  - Cards & Modals: Softer, larger corners (`rounded-xl` / `12px` to `16px`).
- **Elevation (Shadows)**: Use layered, soft ambient shadows (`shadow-sm` or `shadow-md`). Avoid harsh, dark, or high-contrast drop shadows.
- **Anti-AI Patterns**: Avoid generic AI design tropes unless explicitly requested (e.g., do not generate purple-to-pink neon gradients, overly colorful card backgrounds, or glowing borders).

## ⚡ 5. Micro-interactions & States
- **Transitions**: Every interactive element (buttons, links, form fields, cards) must have an explicit CSS transition (`transition-colors` or `transition-all` with `duration-200 ease-in-out`).
- **Focus States**: Accessible `:focus-visible` or `.focus-ring` must be styled using an outline with an explicit offset to ensure keyboard navigation works seamlessly.
- **Hover/Active**: Hover states should subtly darken/lighten backgrounds or lift the element slightly using safe hardware-accelerated transforms (`hover:-translate-y-0.5`).

