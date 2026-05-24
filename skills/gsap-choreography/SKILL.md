---
name: gsap-choreography
description: Use when building multi-scene GSAP animations for product demos, hero walkthroughs, or scripted UI sequences in React. Covers timeline architecture, cursor choreography, scene transitions, click systems, typed text, and animation coordination across components.
---

# GSAP Choreography

Build scripted, multi-scene product animations using GSAP timelines in React. Not tweens-101 — this is the orchestration layer: how to coordinate cursor movement, click feedback, scene transitions, typed text, and UI state changes into a single looping timeline that feels like a directed film.

**Live example:** [costumary.com](https://www.costumary.com) hero section — 4-scene product walkthrough, pure DOM, no video.

## Architecture

```
film-script.ts          → Scene definitions, cursor paths, data (no GSAP)
film-primitives.tsx      → Static DOM: frame, sidebar, cursor SVG (no GSAP)
film-panels.tsx          → Scene content: each tab's UI (no GSAP)
film-demo.tsx            → Single useGSAP hook: the entire choreography
animation-provider.tsx   → React context: play/pause/restart coordination
```

**The rule:** GSAP code lives in exactly one file. Everything else is inert DOM with `data-film-*` attributes. This separation means designers edit visuals without touching animation, and the timeline can target any element by data attribute.

## Timeline as Central Authority

One `gsap.timeline()` owns the entire sequence. Every animation is a `.to()` or `.set()` call on this timeline, positioned with labels.

```tsx
const timeline = gsap.timeline({
  repeat: -1,
  repeatDelay: 0.75,
  defaults: { ease: "power3.out" },
});

// Labels mark narrative beats — everything positions relative to these
timeline
  .to(cursor, { x: target.x, y: target.y, duration: 0.72 }, "groupMove")
  .to(toolbar, { backgroundColor: "rgba(182,95,114,0.15)", duration: 0.18 }, "groupMove+=0.55");

click(target, "groupClick");  // helper adds ripple + cursor squeeze at label

timeline
  .to(groupOutline, { autoAlpha: 1, scale: 1, duration: 0.38 }, "groupClick+=0.1")
  .to(note, { autoAlpha: 1, y: 0, duration: 0.38 }, "noteClick+=0.12");
```

**Why one timeline:** Multiple timelines desync on tab-switch, resize, or replay. A single timeline with labels gives you scrubbing, replay, and pause for free. Labels are the API between "what happens" and "when."

## Label Naming Convention

Labels are the skeleton. Name them by what the user *does*, not what animates:

```
"dropClick"        → user drops references onto board
"groupClick"       → user clicks the group tool
"noteMove"         → cursor travels to the note tool
"sidebarCollapse"  → sidebar shrinks to icon-only
"navMaterials"     → user clicks Materials in nav
"typeStart"        → cursor reaches the input field
"sendClick"        → user clicks send on the chat
"settle"           → animation winds down, UI resets
```

Position everything relative to labels with `"labelName+=0.12"` offsets. Never use absolute seconds — inserting a scene shifts everything downstream automatically.

## Cursor System

The cursor is a positioned SVG that moves via the timeline. It must feel human, not robotic.

### Movement

```tsx
// Cursor travels to a target — duration scales with distance
.to(cursor, { x: target.x, y: target.y, duration: 0.72, ease: "power2.out" }, "moveLabel")
```

**Duration guidelines:**
- Short hop (same area): 0.4–0.6s
- Cross-screen travel: 0.7–1.0s
- Entering from off-screen: 1.0–1.2s

Never use `ease: "linear"` for cursor movement. Real hands accelerate and decelerate.

### Click Feedback

Every click needs two things: a cursor squeeze and a ripple burst.

```tsx
function click(position: { x: number; y: number }, label: string) {
  timeline
    .set(ripple, { x: position.x, y: position.y, scale: 0.2, autoAlpha: 0.55 }, label)
    .to(cursor, { scale: 0.88, duration: 0.08, ease: "power2.out" }, label)
    .to(ripple, { scale: 3.4, autoAlpha: 0, duration: 0.54, ease: "power2.out" }, label)
    .to(cursor, { scale: 1, duration: 0.16, ease: "back.out(2.2)" }, `${label}+=0.09`);
}
```

**Why `back.out(2.2)` on release:** The cursor slightly overshoots back to full scale, like a finger lifting off a touchpad. Without it, clicks feel mechanical.

### Cursor Position Measurement (CRITICAL)

**Never hardcode cursor coordinates.** This is the single most common mistake. Hardcoded `{ x: 300, y: 240 }` will miss the target on different screen sizes, after layout changes, or when the scaled frame does not match your assumptions. The cursor clicks on empty space and the whole demo looks broken.

Always measure targets from the actual DOM elements:

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

// For resize handles, measure the corner instead of center
const measureCorner = (el: HTMLElement) => {
  const frame = root.getBoundingClientRect();
  const r = el.getBoundingClientRect();
  const s = scale || 1;
  return {
    x: Math.round((r.right - frame.left) / s),
    y: Math.round((r.bottom - frame.top) / s),
  };
};
```

Measure once before building the timeline, then use the measured positions everywhere:

```tsx
const buttonPos = measure(addButton);
const cardPos = measure(recipeCard);
const sidebarItemPos = measure(navItem);

tl.to(cursor, { x: buttonPos.x, y: buttonPos.y, duration: 0.7 }, "moveToButton");
click(buttonPos, "buttonClick");
```

Store fallback coordinates in the film script for SSR/test environments where DOM measurement returns zero. But in the browser, always measure.

## Scene Transitions

Scenes are tabs — only one visible at a time. Transition pattern:

```tsx
// 1. Update nav state (immediate, not animated)
.add(() => {
  setActiveNav(root, "nav-materials");
  setProgressDots(root, 1);
  setCaption(captionEl, scenes[1].caption);
}, "navMaterials+=0.56")

// 2. Cross-fade content
.to(prevTab, { autoAlpha: 0, duration: 0.24 }, "materialsClick+=0.08")
.to(nextTab, { autoAlpha: 1, duration: 0.28 }, "materialsClick+=0.14")
```

**Critical:** Use `autoAlpha` (not `opacity`) — it sets `visibility: hidden` at 0, removing elements from tab order and screen readers.

**State helpers** use classList, not GSAP, for nav highlighting:

```tsx
function setNav(root: HTMLElement, target: string) {
  root.querySelectorAll("[data-film-nav]").forEach((item) => {
    const isActive = item.getAttribute("data-film-nav") === target;
    item.classList.toggle("bg-rose/10", isActive);
    item.classList.toggle("text-rose", isActive);
  });
}
```

## Typed Text Effect

Animate a counter, slice the string on each update:

```tsx
const prompt = "What should I do next so I don't fall behind?";
const counter = { x: 0 };

timeline
  .to(placeholder, { autoAlpha: 0, duration: 0.1 }, "typeStart+=0.56")
  .to(caret, { autoAlpha: 1, duration: 0.1 }, "typeStart+=0.56")
  .to(counter, {
    x: prompt.length,
    duration: 1.65,
    ease: "none",    // constant speed = typewriter feel
    onUpdate: () => {
      target.textContent = prompt.slice(0, Math.round(counter.x));
    },
  }, "typeStart+=0.62");
```

**`ease: "none"` is correct here** — real typing is roughly constant speed, not eased.

**Looping gotcha:** The counter object retains its end value across loops. Reset it in the reset block:

```tsx
timeline.add(() => {
  counter.x = 0;
  target.textContent = "";
}, 0);
```

## Stagger Patterns

Elements appearing in groups should stagger, not pop simultaneously:

```tsx
// Cards landing on a board — slight delay between each
.to(cards, {
  autoAlpha: 1, y: 0, scale: 1,
  rotation: (i: number) => rotations[i],    // per-card rotation for organic feel
  stagger: 0.07,
  duration: 0.46,
}, "dropClick+=0.16")

// Timeline rows revealing — faster stagger, feels like a list loading
.to(timelineCards, {
  autoAlpha: 1, y: 0,
  stagger: 0.06,
  duration: 0.28,
}, "timelineClick+=0.12")
```

**Stagger timing:** 0.05–0.08s for related items (cards, rows). 0.10–0.15s for independent items (import previews).

## Element Initial States

Set every animated element's start state explicitly at timeline position 0. Never rely on CSS defaults:

```tsx
gsap.set(cursor, { x: offScreenX, y: offScreenY });
gsap.set(cards, { autoAlpha: 0, y: 26, scale: 0.82 });
gsap.set(overlays, { autoAlpha: 0, y: 14, scale: 0.96 });
gsap.set(toast, { autoAlpha: 0, y: 14, scale: 0.96 });
```

**For looping timelines:** Add a reset block at position 0 that restores every element. Without this, the second loop starts from the end state of the first.

```tsx
timeline.add(() => {
  gsap.set(cursor, { x: offScreenX, y: offScreenY });
  gsap.set(typed, { textContent: "" });
  gsap.set(allTabs, { autoAlpha: 0 });
  gsap.set(activeTab, { autoAlpha: 1 });
  // ... reset every animated property
}, 0);
```

## Breathing Room

Never chain actions back-to-back. Viewers need time to register what happened.

```tsx
// After the full board is visible, pause before navigating away
.to({}, { duration: 1.4 }, "sidebarCollapse+=3.1")

// After the answer appears, let it sit before winding down
.to(cursor, { x: settle.x, y: settle.y, duration: 0.72 }, "settle")
```

**Rule of thumb:** 0.8–1.5s of dead time after any major state change. Shorter for small reveals (0.3s), longer for full scene changes.

## Layout Morphing

Sidebar collapse is a multi-property transition. Order matters:

```tsx
// 1. Fade text labels first (fast, small)
.to(sidebarText, { autoAlpha: 0, width: 0, duration: 0.28 }, "sidebarCollapse")
// 2. Hide decorative elements
.to(divider, { autoAlpha: 0, height: 0, duration: 0.24 }, "sidebarCollapse")
// 3. Swap logos (instant, no tween)
.set(logoFull, { display: "none" }, "sidebarCollapse")
.set(logoIcon, { display: "block" }, "sidebarCollapse")
// 4. Animate the width (slow, dominant motion)
.to(sidebar, { width: COLLAPSED_W, duration: 0.44, ease: "power2.inOut" }, "sidebarCollapse")
// 5. Workspace expands to fill
.to(workspace, { left: COLLAPSED_W, duration: 0.44, ease: "power2.inOut" }, "sidebarCollapse")
```

## Multi-Cursor Collaboration

Remote cursors move independently on the same timeline:

```tsx
// Maya moves top-right while main cursor goes bottom-left
.to(cursor, { x: 200, y: 530, duration: 0.6 }, "sidebarCollapse+=0.4")
.to(mayaCursor, { x: "+=280", y: "-=10", duration: 0.7 }, "sidebarCollapse+=0.6")
.to(joCursor, { x: "+=580", y: "+=120", duration: 0.7 }, "sidebarCollapse+=1.0")
```

Use `"+="` relative offsets for remote cursors — they drift from their current position rather than jumping to absolute coordinates.

## Animation Manager (React Context)

Coordinate play/pause/restart across multiple animation components:

```tsx
interface AnimationEntry {
  id: string;
  timeline: gsap.core.Timeline;
}

// Provider stores timelines in a ref (no re-renders on timeline creation)
const timelines = useRef(new Map<string, gsap.core.Timeline>());

// Components register their timeline on mount
register({ id: "product-film", timeline });
// And unregister on unmount
return () => unregister("product-film");
```

Expose `play(id)`, `pause(id)`, `restart(id)` through context. External controls (play/pause buttons) call these without knowing timeline internals.

## Responsive Scaling

The film renders at a fixed design size and scales down with a CSS transform:

```tsx
const FILM_WIDTH = 1180;
const FILM_HEIGHT = 760;

// Observe container width, compute scale
const updateScale = () => {
  const width = root.getBoundingClientRect().width;
  setFilmScale(width > 0 ? Math.min(width / FILM_WIDTH, 1) : 1);
};

// Apply with transform-origin: top-left
<div style={{
  width: FILM_WIDTH,
  height: FILM_HEIGHT,
  transform: `scale(${scale})`,
  transformOrigin: "top left",
}} />
```

Container maintains aspect ratio: `style={{ aspectRatio: `${FILM_WIDTH} / ${FILM_HEIGHT}` }}`.

**Control positioning:** Play/pause buttons should live *inside* the scaled frame, not outside it. If placed outside, you need manual positioning math to account for the scale factor. Inside the frame, they scale naturally with everything else:

```tsx
// Inside the film frame div (scales with content)
<div style={{ position: "absolute", bottom: 10, right: 10, zIndex: 60 }}>
  <button onClick={() => playing ? tl.pause() : tl.play()}>
    {playing ? "⏸" : "▶"}
  </button>
</div>
```

## Accessibility

Always support `prefers-reduced-motion`. Here is the hook:

```tsx
function usePrefersReducedMotion() {
  const [reduced, setReduced] = useState(false);
  useEffect(() => {
    const mq = window.matchMedia("(prefers-reduced-motion: reduce)");
    setReduced(mq.matches);
    const handler = (e) => setReduced(e.matches);
    mq.addEventListener("change", handler);
    return () => mq.removeEventListener("change", handler);
  }, []);
  return reduced;
}
```

Use it to skip the entire timeline and show a static resting state:

```tsx
const prefersReducedMotion = usePrefersReducedMotion();

if (prefersReducedMotion) {
  // Show the resting state — all key elements visible, no animation
  buildReducedMotionState(root);
  const paused = gsap.timeline({ paused: true });
  register({ id: FILM_ID, timeline: paused });
  return () => unregister(FILM_ID);
}
```

Register a paused empty timeline so play/pause controls don't crash.

## Data Attribute Convention

Every animated element gets a `data-film-*` attribute. These are the animation's API to the DOM:

```
data-film-cursor          → the cursor SVG
data-film-ripple          → click ripple circle
data-film-caption         → scene caption text
data-film-tab="references"→ scene content panels
data-film-nav="nav-materials" → sidebar nav items
data-film-ref-card="0"    → indexed elements (cards, rows)
data-film-tool="group"    → toolbar buttons
data-film-typed           → typed text target
data-film-send            → send button
```

**Why data attributes over refs:** The timeline targets elements with `gsap.utils.selector(root)` and CSS selectors. Data attributes are queryable, readable, and don't require threading React refs through component boundaries.

## Responsive Scaling with CSS Zoom

Demos that live inside a page layout (story steps, embedded showcases) need to fit containers narrower than their design width. Two scaling strategies exist:

| Strategy | How it scales | Cursor coordinates | Use when |
|----------|--------------|-------------------|----------|
| Internal `transform: scale()` | Film owns its own scale factor | Frame design pixels, no compensation needed | Self-contained films with `FILM_WIDTH`/`FILM_HEIGHT` |
| External CSS `zoom` wrapper | Parent container applies `zoom` | Must compensate: `getBoundingClientRect()` returns visual pixels, `clientWidth` returns CSS pixels | Simpler demos embedded in responsive grids |

### Self-contained films handle their own scale

Films that define `FILM_WIDTH`/`FILM_HEIGHT`, set `aspect-ratio` on the container, and scale an internal frame with `transform: scale(filmScale)` are fully self-contained. Their `measureCenter` already accounts for scale by dividing `getBoundingClientRect()` values by `frameRect.width / DESIGN_WIDTH`. Do not wrap these in an external zoom container.

### Zoom-wrapped demos need coordinate compensation

When CSS `zoom` shrinks a container, `getBoundingClientRect()` returns visual (post-zoom) pixels while `clientWidth` returns logical (pre-zoom) pixels. Every position measurement must divide by the zoom ratio:

```tsx
function measureInContainer(container: HTMLElement, target: HTMLElement) {
  const cr = container.getBoundingClientRect();
  const tr = target.getBoundingClientRect();
  const z = container.clientWidth > 0 ? cr.width / container.clientWidth : 1;
  return {
    x: Math.round((tr.left + tr.width / 2 - cr.left) / z),
    y: Math.round((tr.top + tr.height / 2 - cr.top) / z),
  };
}
```

This applies whether the cursor is positioned via React state + CSS transitions or GSAP `x`/`y` tweens.

### Defer rendering until zoom is known

Child `useGSAP` hooks measure positions on mount. If they mount before zoom is calculated, they measure against the raw viewport width (e.g., 328px on mobile) instead of the zoomed design width (700px). Every cursor position breaks.

Gate children on zoom being resolved:

```tsx
const [zoom, setZoom] = useState<number | null>(null);

useLayoutEffect(() => {
  const w = outerRef.current?.clientWidth ?? 0;
  setZoom(w > 0 && w < DESIGN_W ? w / DESIGN_W : 1);
}, []);

return (
  <div ref={outerRef} className="w-full overflow-hidden">
    {zoom !== null && (
      <div style={zoom < 1 ? { width: DESIGN_W, zoom } : undefined}>
        {children}
      </div>
    )}
  </div>
);
```

`useLayoutEffect` runs synchronously after DOM mount but before paint. Children mount into the correctly-sized container on the first visible frame.

### Guard ResizeObserver callbacks against feedback loops

Setting a style property (like `zoom` or `height`) from inside a `ResizeObserver` callback can re-trigger the observer. Always compare against the previous width:

```tsx
let prevW = 0;
const update = () => {
  const w = el.clientWidth;
  if (w === prevW || w <= 0) return;
  prevW = w;
  // ... recalculate
};
```

## Common Mistakes

| Mistake | Fix |
|---------|-----|
| **Hardcoded cursor coordinates** | **Always `measure()` from DOM elements. Hardcoded positions miss targets after any layout change.** |
| Using `opacity` instead of `autoAlpha` | `autoAlpha` also sets `visibility: hidden` at 0 |
| Linear cursor movement | Use `power2.out` or `power3.out` for natural deceleration |
| No reset block for looping timelines | Add explicit `gsap.set()` for every property at position 0 |
| Absolute time positions (`2.5`) | Use labels + offsets (`"labelName+=0.12"`) |
| Animating `left`/`top` instead of `x`/`y` | Layout properties force reflow; transforms are compositor-friendly |
| Missing breathing room between scenes | Add `0.8–1.5s` dead time after major state changes |
| Click without ripple+squeeze | Both are needed — ripple alone looks like a UI bug |
| `ease: "power2.out"` on typed text | Typed text should be `ease: "none"` (constant speed) |
| Multiple useGSAP hooks for one sequence | One hook, one timeline, labels for structure |
| Raw `getBoundingClientRect()` inside zoomed container | Divide by zoom ratio (`cr.width / container.clientWidth`) to get CSS pixels |
| Mounting GSAP children before zoom is calculated | Gate children on `zoom !== null` so `useGSAP` measures the correct container |
| ResizeObserver without width guard | Compare `clientWidth` to previous value to prevent style-triggers-observer loops |
| Mixing `clientWidth` and `getBoundingClientRect` without compensation | Pick one coordinate space. Inside zoom wrappers, always divide visual pixels by zoom |
