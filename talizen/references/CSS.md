# Site-level `index.css`

This document explains how to add CSS and Tailwind v4 configuration in the Talizen platform.

## Purpose

A dedicated site file lets authors define Tailwind directives (`@theme`, `@utility`, `@layer`, etc.), theme tokens, and custom utilities without inline styles on every component.

This file supports both plain CSS and Tailwind directives.

## Minimal example

Define brand tokens and custom utilities:

```css
/* index.css */

@theme inline {
  --color-brand: #005aa7;
  --color-brand-foreground: #ffffff;
  --radius: 0.75rem;
}

@utility shadow-soft {
  box-shadow: 0 18px 40px -25px rgba(0, 0, 0, 0.35);
}

@utility btn-primary {
  background-color: var(--color-brand);
  color: var(--color-brand-foreground);
  padding: 0.75rem 1.25rem;
  border-radius: var(--radius);

  &:hover {
    opacity: 0.92;
  }
}
```
