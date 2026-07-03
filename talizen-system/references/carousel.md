# Talizen Carousel Usage (Embla)

This document defines the default carousel/slideshow approach in Talizen projects.

Use `embla-carousel` as the core engine.  
For React projects, use `embla-carousel-react`.

## ImportMap setup (`talizen.config.ts`)

Configure third-party libraries in `importMap.imports`, then import by specifier in components.

```tsx
// talizen.config.ts
export default {
  importMap: {
    imports: {
      "embla-carousel-react": "https://esm.sh/embla-carousel-react?external=react",
      "embla-carousel-autoplay": "https://esm.sh/embla-carousel-autoplay?external=react",
    },
  },
}
```

## React example (no standalone CSS)

Prefer utility classes (for example Tailwind) instead of writing separate CSS files.

```tsx
import useEmblaCarousel from "embla-carousel-react"
import Autoplay from "embla-carousel-autoplay"

const slides = ["Slide 1", "Slide 2", "Slide 3"]

export function HeroCarousel() {
  const [emblaRef] = useEmblaCarousel(
    {
      loop: true,
      align: "start",
      containScroll: "trimSnaps",
    },
    [Autoplay({ delay: 4000, stopOnInteraction: true })],
  )

  return (
    <section aria-label="Featured slides" className="w-full">
      <div ref={emblaRef} className="overflow-hidden">
        <div className="flex">
          {slides.map((slide) => (
            <article
              key={slide}
              className="min-w-0 flex-[0_0_100%] px-4"
              aria-roledescription="slide"
            >
              {slide}
            </article>
          ))}
        </div>
      </div>
    </section>
  )
}
```

## Recommended options

- `loop: true` for seamless cyclical navigation.
- `containScroll: "trimSnaps"` to prevent over-scrolling at edges.
- `Autoplay({ delay, stopOnInteraction })` for optional auto-advance behavior.
- Keep slide basis explicit (`flex-[0_0_100%]`, `flex-[0_0_50%]`, etc.).

## Import rules

- Do not import Embla from full CDN URL directly inside pages/components.
- Always register external libs in `talizen.config.ts` under `importMap.imports` first.
- Keep the `import` specifier exactly the same as the `imports` key.

## Accessibility checklist

- Provide a clear region label (for example `aria-label="Featured slides"`).
- Add previous/next controls that are keyboard reachable.
- Ensure autoplay can be paused/stopped when enabled.
- Respect reduced motion preference for auto-advancing behavior.
