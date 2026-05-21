# GSAP Choreography

Use when building multi-scene GSAP animations for product demos, hero walkthroughs, or scripted UI sequences in React. Covers timeline architecture, cursor choreography, scene transitions, click systems, typed text, and animation coordination across components.

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

Use `autoAlpha` for cross-fades. Update nav state with classList, not GSAP. State helpers are plain DOM manipulation, not tweens.

```tsx
.add(() => {
  setActiveNav(root, "nav-materials");
  setProgressDots(root, 1);
}, "navMaterials+=0.56")
.to(prevTab, { autoAlpha: 0, duration: 0.24 }, "materialsClick+=0.08")
.to(nextTab, { autoAlpha: 1, duration: 0.28 }, "materialsClick+=0.14")
```

## Typed Text

Animate a counter with `ease: "none"`, slice the string on each update. Reset the counter in the loop reset block.

## Responsive Scaling

Fixed design size + CSS `transform: scale()` + `transformOrigin: "top left"`. Container uses `aspectRatio`. Controls live inside the scaled frame.

## Data Attributes

```
data-film-cursor, data-film-ripple, data-film-caption
data-film-tab="references", data-film-nav="nav-materials"
data-film-ref-card="0", data-film-tool="group"
data-film-typed, data-film-send
```

## Accessibility

Support `prefers-reduced-motion`. Skip the timeline, show a static resting state. Register a paused empty timeline so play/pause controls don't crash.

## Common Mistakes

| Mistake | Fix |
|---------|-----|
| Hardcoded cursor coordinates | `measure()` from DOM elements |
| `opacity` instead of `autoAlpha` | `autoAlpha` sets `visibility: hidden` at 0 |
| Linear cursor movement | `power2.out` or `power3.out` |
| No reset block for loops | `gsap.set()` every property at position 0 |
| Absolute time positions | Labels + offsets |
| Animating `left`/`top` | Use `x`/`y` transforms |
| No breathing room | 0.8–1.5s after state changes |
| Click without ripple+squeeze | Both required |
| Eased typed text | `ease: "none"` |
| Multiple useGSAP hooks | One hook, one timeline |
