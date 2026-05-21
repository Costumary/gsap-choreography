---
applyTo: "**/*{film,demo,animation,choreograph,gsap,timeline}*"
---

# GSAP Choreography

When building multi-scene GSAP product animations in React, follow these patterns.

## Architecture

GSAP code lives in exactly one file. Everything else is inert DOM with `data-film-*` attributes.

```
film-script.ts          → Scene definitions, cursor paths, data (no GSAP)
film-primitives.tsx      → Static DOM: frame, sidebar, cursor SVG (no GSAP)
film-panels.tsx          → Scene content panels (no GSAP)
film-demo.tsx            → Single useGSAP hook: the entire choreography
animation-provider.tsx   → React context: play/pause/restart coordination
```

## Core Rules

1. **One timeline.** One `gsap.timeline()` owns the entire sequence with label-based positioning.
2. **Never hardcode cursor coordinates.** Always measure from DOM via `getBoundingClientRect()` relative to the animation frame.
3. **Labels name user actions.** `"groupClick"` not `"fadeInOverlay"`. Position: `"label+=0.12"`.
4. **`autoAlpha` over `opacity`.** Sets `visibility: hidden` at 0.
5. **Human cursor movement.** `power2.out` for travel, `back.out(2.2)` for click release, `ease: "none"` for typed text.
6. **Click = squeeze + ripple.** Both are required — ripple alone looks like a bug.
7. **Breathing room.** 0.8–1.5s dead time after major state changes.
8. **Reset block at position 0.** For looping timelines, explicitly reset every animated property.
9. **Transforms only.** Animate `x`/`y`/`scale`/`rotation`/`opacity`, never `left`/`top`/`width` for motion.
10. **Stagger 0.05–0.08s** for related items, 0.10–0.15s for independent items.

## Cursor Click Pattern

```tsx
function click(position: { x: number; y: number }, label: string) {
  timeline
    .set(ripple, { x: position.x, y: position.y, scale: 0.2, autoAlpha: 0.55 }, label)
    .to(cursor, { scale: 0.88, duration: 0.08, ease: "power2.out" }, label)
    .to(ripple, { scale: 3.4, autoAlpha: 0, duration: 0.54, ease: "power2.out" }, label)
    .to(cursor, { scale: 1, duration: 0.16, ease: "back.out(2.2)" }, `${label}+=0.09`);
}
```

## Position Measurement

```tsx
const measure = (el: HTMLElement) => {
  const frame = root.getBoundingClientRect();
  const r = el.getBoundingClientRect();
  const s = scale || 1;
  return {
    x: Math.round((r.left + r.width / 2 - frame.left) / s),
    y: Math.round((r.top + r.height / 2 - frame.top) / s),
  };
};
```

## Scene Transitions

```tsx
.add(() => {
  setActiveNav(root, "nav-materials");
  setProgressDots(root, 1);
}, "navMaterials+=0.56")
.to(prevTab, { autoAlpha: 0, duration: 0.24 }, "materialsClick+=0.08")
.to(nextTab, { autoAlpha: 1, duration: 0.28 }, "materialsClick+=0.14")
```

## Responsive Scaling

Fixed design size + CSS transform scale. Container uses `aspectRatio`. Play/pause buttons inside the scaled frame, not outside.

## Accessibility

Support `prefers-reduced-motion` — skip the timeline, show a static resting state, register a paused empty timeline.
