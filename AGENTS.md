# GSAP Choreography — Agent Instructions

When building multi-scene GSAP animations for product demos, hero walkthroughs, or scripted UI sequences in React, follow the patterns in `skills/gsap-choreography/SKILL.md`.

## Key principles

- **One timeline owns everything.** No separate timelines per scene — one `gsap.timeline()` with labels as structural markers.
- **Never hardcode cursor coordinates.** Always `measure()` from actual DOM elements using `getBoundingClientRect()` relative to the animation frame.
- **GSAP code in one file.** Everything else is inert DOM with `data-film-*` attributes. Designers edit visuals, animators edit the timeline.
- **Labels name user actions, not animations.** `"groupClick"` not `"fadeInOverlay"`. Position with `"label+=0.12"` offsets, never absolute seconds.
- **`autoAlpha` over `opacity`.** It also sets `visibility: hidden` at 0.
- **Cursor must feel human.** `power2.out` for movement, `back.out(2.2)` for click release overshoot, `ease: "none"` for typed text.
- **Breathing room.** 0.8–1.5s dead time after major state changes.

## File structure

```
film-script.ts          → Scene definitions, cursor paths, data (no GSAP)
film-primitives.tsx      → Static DOM: frame, sidebar, cursor SVG (no GSAP)
film-panels.tsx          → Scene content: each tab's UI (no GSAP)
film-demo.tsx            → Single useGSAP hook: the entire choreography
animation-provider.tsx   → React context: play/pause/restart coordination
```

## Full reference

See `skills/gsap-choreography/SKILL.md` for complete patterns including click feedback, scene transitions, typed text, stagger timing, layout morphing, multi-cursor, responsive scaling, and accessibility.
