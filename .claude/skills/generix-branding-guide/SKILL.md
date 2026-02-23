---
name: generix-branding-guide
description: Apply the Generix brand identity (colors, fonts, layout, tone) to any document or artifact. Use this skill whenever creating or styling presentations, documents, web pages, dashboards, reports, emails, or any visual output for Generix. Trigger whenever the user mentions "Generix branding", "brand colors", "company style", "on-brand", or asks to make something look like a Generix document. Also use when creating any Generix-related deliverable that should follow corporate visual identity, even if the user doesn't explicitly mention branding.
---

# Generix Branding Guide

Apply the Generix corporate brand identity to all documents and visual artifacts. This skill encodes the complete Generix Brand Guide 2024 so every deliverable — slides, documents, web pages, dashboards, reports, and emails — looks and feels authentically Generix.

## Brand Overview

**Generix** is a global SaaS company focused on supply chain management software.

- **Tagline**: CONNECTING BUSINESSES TOGETHER
- **Mission**: Digitally connect all businesses together across global value chains
- **Vision**: Help every business serve any customer in any geography in the most profitable and sustainable manner
- **Purpose**: Unlock the combined power of new technologies and data for businesses to best serve their end-customers and grow sustainably in a digital world
- **Ambition**: Become a global SaaS leader & the fastest growing player in all market categories

**Three Strategic Pillars:**
- **People-First** — Empower all employees with a cloud-mindset to deliver the best customer experience
- **Cloud-First** — Deliver best-in-class SaaS solutions by putting Cloud & AI at the heart of innovation
- **Customer-First** — Establish new go-to-market to move customers to the cloud & pursue growth in new markets

## Color Palette

Use the Generix color palette consistently. White space should dominate at ~60% of any design. Read `references/colors.md` for full tint scales and accessibility ratios.

### Primary Color — Dark Navy (dominant brand color)

| Property | Value |
|----------|-------|
| HEX | `#002334` |
| RGB | 0, 35, 52 |
| CMYK | 94, 74, 53, 62 |

Tints: `#334F5D` (80%), `#667B85` (60%), `#99A7AE` (40%), `#CCD3D6` (20%)

### Secondary Color — Orange (accent)

| Property | Value |
|----------|-------|
| HEX | `#F28A48` |
| RGB | 242, 138, 72 |
| CMYK | 1, 56, 81, 0 |

Tints: `#F5A16D` (80%), `#F7B991` (60%), `#FAD0B6` (40%), `#FCE8DA` (20%)

### Tertiary Colors (supporting — max 40% of design)

| Color | HEX | RGB |
|-------|-----|-----|
| Green | `#00A59A` | 0, 165, 154 |
| Pink | `#E2A1AE` | 226, 161, 174 |
| Yellow | `#FFDA88` | 255, 218, 136 |

Tertiary colors support graphics, data graphs, iconography, website, and social media. They must always appear alongside the primary palette. Never use tertiary colors for the logo, wordmark, or brand icon.

### Color Ratio

Apply this distribution across all documents:
1. **White space** — 60% dominance
2. **Dark Navy** (#002334) — primary brand color, used for text and key elements
3. **Orange** (#F28A48) — secondary accent for highlights, CTAs, visual interest
4. **Tertiary colors** — supporting elements, charts, icons (never exceed 40%)

### Background Colors

- **Light backgrounds**: White (`#FFFFFF`) or light grey (`#EDF0F2` / `#CCD3D6`)
- **Dark backgrounds**: Dark Navy (`#002334`) or medium navy (`#334F5D`)
- Use orange sparingly as an accent, never as a full background

## Typography

### Primary Font — General Sans

The primary typeface. A clean, modern sans-serif that works across multiple languages.

| Weight | Usage |
|--------|-------|
| **SemiBold** | Eyebrows, buttons, links, labels |
| **Medium** | Headlines, body copy, subheadings |
| **Regular** | Body copy, descriptions, supporting text |

### Alternate Font — Inter

Use when General Sans is unavailable (technical constraints, compatibility). Same weight structure as General Sans.

### Font Usage Rules

| Element | Font | Weight |
|---------|------|--------|
| Eyebrow / Category label | General Sans | SemiBold |
| Headlines | General Sans | Medium |
| Body copy | General Sans | Medium or Regular |
| Buttons & Links | General Sans | SemiBold |

When implementing in CSS/HTML, load General Sans first with Inter as fallback:
```css
font-family: 'General Sans', 'Inter', sans-serif;
```

For Google Fonts or system fallback contexts where neither is available, use `Inter` (available on Google Fonts) as the primary web-safe alternative.

## Visual Identity

### Logo

The logo has three components: **Icon** (globe/flow lines), **Wordmark** ("generix"), and **Tagline** ("CONNECTING BUSINESSES TOGETHER"). Read `references/logo-usage.md` for full rules on placement, clear space, minimum size, and incorrect usage.

**Key rules:**
- On dark backgrounds: white wordmark + orange icon accents
- On light backgrounds: dark navy wordmark + orange icon accents
- Clear space around logo: at least the height of the letter "x" in the wordmark
- Minimum digital size: 100px (full logo), 25px (icon only)
- Never stretch, distort, rotate, recolor with unapproved colors, or place on busy backgrounds

### Icons

- 88px x 88px canvas with 2px stroke weight
- Two-color system: Dark Navy (`#002334` or `#003B57`) + accent color (`#00A59A` or `#F28A48`)
- On dark backgrounds: White + accent color (`#ECFF1F` or orange)
- Minimum 2px protective space around each icon
- Clean, straightforward aesthetic — balance between simplicity and detail

### Illustrative Flows (Background Patterns)

Delicate flowing lines symbolizing global supply chain pathways. Two-color gradients that fade at the ends. Available in all brand color combinations (orange, green, yellow, pink) on both light and dark backgrounds.

### Photography

- Blend human touch with technology
- Show synergy between people and software
- Apply 16px corner radius to all photos
- Frame with rounded corners or circular elements
- Occasionally use cutout elements for dynamic compositions

## Mandatory Logo Placement (PPTX & PDF)

**This is a hard requirement. The Generix icon MUST appear on every single page/slide of every PPTX and PDF document. No exceptions. No opt-out.**

### Icon Asset Files

Two icon variants are stored as PNG files with transparent backgrounds in the `references/` folder:

- `references/generix-icon-dark.png` — Dark navy + orange icon (use on **light backgrounds**: white, light grey, etc.)
- `references/generix-icon-white.png` — White + orange icon (use on **dark backgrounds**: Dark Navy `#002334`, medium navy `#334F5D`, etc.)

### Placement Rules

| Property | Value |
|----------|-------|
| **Position** | Top-left corner of every slide/page |
| **Margin** | 0.5 inches (36pt) from both the top and left edges |
| **Size** | Proportional — ~4% of the slide/page width (e.g., ~0.53 inches on a standard 13.33" wide slide, ~0.33 inches on an 8.5" wide PDF page) |
| **Minimum size** | 25px (per brand guide minimum for icon-only digital use) |

### Background Detection for Variant Selection

- **Default**: Use `generix-icon-dark.png` (dark navy + orange on transparent)
- **Switch to white variant** when the slide/page background is explicitly set to Dark Navy (`#002334`), medium navy (`#334F5D`), or any color with luminance below 30%
- For PPTX: Read the slide's `<a:solidFill>` or background fill property. If it matches a dark navy hex or has RGB values where `(0.299*R + 0.587*G + 0.114*B) < 76`, use `generix-icon-white.png`
- For PDF: Apply the same logic based on the page's background color

### Implementation Notes

When creating or modifying a PPTX file:
1. Read the icon PNG from the `references/` folder relative to this skill file
2. For each slide, determine the background color and select the appropriate icon variant
3. Insert the icon as an image element positioned at (0.5", 0.5") from the top-left
4. Set the icon width to ~4% of the slide width; maintain aspect ratio for height
5. Set the icon's z-order so it appears above the background but does not obscure critical content

When creating or modifying a PDF file:
1. Follow the same placement, sizing, and variant selection rules
2. Embed the icon on every page at the top-left with 0.5" margins

## Applying the Brand

### For Presentations (PPTX)

1. **Add the Generix icon to every slide** (see "Mandatory Logo Placement" above)
2. Use Dark Navy (`#002334`) for title slides and section dividers
3. White backgrounds for content slides
4. Orange for emphasis, highlights, and CTAs
5. General Sans SemiBold for slide titles, Medium for body
6. Use tertiary colors in charts and data visualizations
7. Apply 16px corner radius to images

### For Documents (DOCX/PDF)

1. **Add the Generix icon to every page** (see "Mandatory Logo Placement" above — applies to PDF output)
2. Dark Navy for headings and titles
3. Body text in Dark Navy or `#334F5D`
4. Orange for accent lines, highlights, callouts
5. Section dividers using orange or navy rules
6. Tables with navy headers, alternating light grey rows
7. General Sans or Inter for all text

### For Web / HTML / Dashboards

1. White or `#EDF0F2` page background
2. Dark Navy for primary text and navigation
3. Orange for CTAs, buttons, and interactive elements
4. Tertiary colors for charts, badges, status indicators
5. `font-family: 'General Sans', 'Inter', sans-serif`
6. 16px border-radius on cards and images
7. Follow web accessibility ratios (see `references/colors.md`)

### For Email / Internal Communications

1. Clean white background
2. Dark Navy headers
3. Orange accent for key links or highlights
4. Keep it minimal — prioritize readability over decoration

## Web Accessibility

Ensure text meets WCAG contrast requirements. Key accessible combinations:

| Foreground | Background | Ratio | Rating |
|------------|------------|-------|--------|
| `#FFFFFF` | `#002334` | 16.27:1 | AAA |
| `#002334` | `#FFFFFF` | 16.27:1 | AAA |
| `#FFFFFF` | `#334F5D` | 8.68:1 | AAA |
| `#002334` | `#CCD3D6` | 10.74:1 | AAA |
| `#002334` | `#F28A48` | 6.59:1 | AA |
| `#002334` | `#00A59A` | 5.3:1 | AA |
| `#002334` | `#E2A1AE` | 7.7:1 | AA/AAA |
| `#002334` | `#FFDA88` | 12.12:1 | AAA |

Minimum contrast requirements:
- 4.5:1 for normal text
- 3:1 for text larger than 24px
- 3:1 for bold text larger than 19px

## References

For detailed specifications, read these reference files:

- `references/colors.md` — Full color palette with all tints, accessibility ratios, and usage rules
- `references/logo-usage.md` — Logo variants, clear space rules, minimum sizes, incorrect usage examples
- `references/generix-icon-dark.png` — Generix icon (dark navy + orange) with transparent background, for use on light slides/pages
- `references/generix-icon-white.png` — Generix icon (white + orange) with transparent background, for use on dark slides/pages
