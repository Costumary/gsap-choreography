# Landing Page Animation Reference

## Stack Decision

| Layer | Tool | Bundle | Purpose |
|-------|------|--------|---------|
| Scroll engine | Lenis v1 | ~8kb | Smooth momentum scroll, syncs with GSAP ticker |
| Scroll choreography | GSAP 3 + ScrollTrigger | ~30kb | Scrub/pin/snap, direct DOM, 60fps |
| UI transitions | Motion (LazyMotion + domAnimation) | ~19kb | Declarative enter/exit, stagger, layout |
| Simple reveals | CSS scroll-timeline | 0kb | Compositor-thread, zero JS |
| Product demo | Programmatic DOM + fake cursor | 0kb extra | Responsive, maintainable, no video |
| Styling | Tailwind CSS 4 | already in stack | motion-safe: prefixes |

Total animation JS budget: ~57kb gzipped (vs 2-5MB for a hero video).

### Packages to install

```bash
pnpm add gsap @gsap/react lenis
```

`motion` is already installed (v12.38.0). Use `LazyMotion` + `domAnimation` to keep initial load at ~19kb.

GSAP is 100% free for commercial use since fall 2024 (Webflow acquired GreenSock).

## Architecture

```
fe/src/
  components/
    landing/
      hero-section.tsx        # "use client", entrance animations
      demo-section.tsx         # "use client", GSAP ScrollTrigger product demo
      features-section.tsx     # "use client", staggered cards
      cta-section.tsx          # "use client", entrance animation
    landing/shared/
      fake-cursor.tsx          # Reusable animated cursor
      scroll-reveal.tsx        # Reusable scroll-triggered wrapper
      animated-counter.tsx     # Number ticker for stats
  hooks/
    use-scroll-animation.ts   # Custom hook: useGSAP + ScrollTrigger
  lib/
    gsap-register.ts          # One-time plugin registration
    animation.ts              # Shared easing presets, variant defs
```

### Principles

1. **Server Components for structure, Client Components for animation.** The page.tsx stays a Server Component importing client sections.
2. **Each section owns its animations.** No monolithic animation file. Each section registers its own ScrollTrigger within `useGSAP`, cleans up on unmount.
3. **Animation constants outside components** (never recreated on re-render).
4. **Plugin registration once at module scope** in `gsap-register.ts`.
5. **Dynamic imports for heavy sections** with skeleton loading states.

### GSAP + React pattern

```tsx
// gsap-register.ts
import { gsap } from "gsap";
import { ScrollTrigger } from "gsap/ScrollTrigger";
gsap.registerPlugin(ScrollTrigger);

// In component:
import { useGSAP } from "@gsap/react";
import "@/lib/gsap-register";

function DemoSection() {
  const containerRef = useRef<HTMLDivElement>(null);

  useGSAP(() => {
    const tl = gsap.timeline({
      scrollTrigger: {
        trigger: containerRef.current,
        start: "top center",
        end: "bottom center",
        scrub: true,
        pin: true,
      },
    });
    tl.to(".cursor", { x: 340, y: 120, duration: 1 });
    tl.to(".cursor", { scale: 0.9, duration: 0.1 });
    tl.to(".cursor", { scale: 1, duration: 0.1 });
    tl.to(".panel", { opacity: 1, y: 0, duration: 0.5 }, "<");
  }, { scope: containerRef });

  return <div ref={containerRef}>...</div>;
}
```

### Lenis + GSAP sync

```tsx
useEffect(() => {
  function update(time: number) {
    lenisRef.current?.lenis?.raf(time * 1000);
  }
  gsap.ticker.add(update);
  return () => gsap.ticker.remove(update);
}, []);
```

## Performance Rules

| Rule | Why |
|------|-----|
| Only animate `transform` and `opacity` | Compositor-only = no layout/paint = 60fps |
| `will-change: transform` sparingly | Creates GPU layer; too many wastes VRAM |
| `filter: drop-shadow()` over animated `box-shadow` | Avoids paint |
| `motion-safe:` on all non-essential animations | ~25% of macOS/iOS users enable Reduce Motion |
| Disable/pause off-screen animations | Free GPU resources |
| Keep ScrollTrigger count under 20-30 per page | Frame time degrades beyond this |
| Test on 4x CPU throttle in DevTools | Catches jank invisible on fast machines |
| `gsap.quickSetter()` for scroll-linked updates | Caches selector lookups |
| Lazy-load below-fold sections with `next/dynamic` | Faster initial paint |

## Responsive

- `ScrollTrigger.matchMedia({ "(min-width: 768px)": () => { ... } })` for breakpoint-specific
- Reduce parallax intensity on mobile (0.5-1.5 range)
- Disable smooth scroll on low-end devices
- Consider disabling demo cursor on <768px

## Choreography: "From inspiration to finished build"

The demo tells one story: a maker finds inspiration, collects references, plans the project, tracks materials and budget, logs progress, and gets help from the AI. Every scene earns the next. The viewer should feel the momentum of real work taking shape.

The demo auto-plays on a timer loop. Each scene has two beats: (1) cursor moves to a sidebar nav item and clicks, (2) cursor glides into the content area while the scene content animates in. Spring physics on the cursor make it feel like a real hand.

### The Window

- **Title bar:** `h-10`, `border-b border-border/60 bg-[#F7F0E8]`, traffic lights: `h-[11px] w-[11px] rounded-full`, colors `#FF5F57` / `#FEBC2E` / `#28C840`, gap `gap-[7px]`
- **Sidebar:** `w-[168px]`, `bg-surface`, `border-r border-border`
- **Content area:** fills remaining, `bg-paper`
- **Outer container:** `rounded-2xl border border-border/60 bg-surface shadow-[0_32px_80px_rgba(35,32,30,0.14)]`
- **Internal dimensions:** 1100 x 700, scaled via `transform: scale(calc(100cqw / 1100))`
- **Container query:** `containerType: "inline-size"`, `aspectRatio: "1100/700"`

### Sidebar Styling (matches `workbench/sidebar.tsx`)

- **Logo area:** `h-14 px-5`, logo `h-5 w-5`, label `font-ui text-[11px] font-semibold uppercase tracking-[0.12em] text-ink`
- **Nav items:** `gap-3 rounded-lg px-3 py-2.5 font-ui text-[11px] font-medium`
  - Active: `bg-rose/10 text-rose`
  - Inactive: `text-ink-muted`
  - Active bar: `h-5 w-[3px] rounded-r-full bg-rose` (animated via `layoutId="demo-active"`)
  - Icons: `h-4 w-4`, `weight="bold"` when active, `weight="regular"` when inactive
- **Bottom area:** `border-t border-border p-3`, Settings icon `h-4 w-4`
- **Items:** Projects, References, Budget, Timeline, Materials, Build Log, Notes, Assistant

### Cursor Behavior

- Spring physics: `stiffness: 90, damping: 18, mass: 0.6` (slightly floaty, human-feeling)
- Click press: `scale: 0.88` for 80ms, spring back to 1
- Click ripple: `h-3 w-3 rounded-full bg-rose/40`, scales from 0 to 3 with opacity 0.5→0, 600ms
- Nav dwell: 1300ms (cursor arrives and rests before content switches)
- Cursor SVG: `#23201E` fill, white stroke 1.5px, 18x22px, tip at top-left

### Scene Flow

---

#### ACT 1: "The Spark" — Reference Canvas (5 beats, ~8.8s total)

**Nav:** Cursor clicks "References" in sidebar. Rose active bar slides to References. Click ripple.

**Content:** Dot-grid background fades in. Then a 5-beat micro-choreography plays:

**Beat 0 — "The Paste" (3.0s)**
5 reference cards scatter onto the canvas with slight rotation and scale-in (stagger 70ms). Simultaneously, a 6th card appears at bottom-right with a **loading spinner** and "Uploading..." text. After 3.0s the spinner resolves — the paint reference image fades in and a green check + "Pasted" label appears.

**Beat 1 — "The Drag" (1.8s)**
Cursor moves to "Boot detail" card. Press (scale 0.88). Card lifts (shadow deepens, scale 1.03). Cursor glides card 120px right + 40px down. Card settles.

**Beat 2 — "The Checkbox" (1.4s)**
Cursor moves to Paint Checklist card. Clicks "Weathering pass". Checkbox fills rose with spring pop. Text gets strikethrough.

**Beat 3 — "The Notes" (2.2s)**
Cursor moves to Build Notes card. Placeholder "Click to add notes..." disappears. Text types char-by-char (28ms/char): "Heat-form before gluing. Test range of motion at elbow joints." Blinking cursor follows.

**Beat 4 — "Settle" (1.4s)**
Cursor drifts to center. Viewer absorbs the full board.

**Element styling (matches real `reference-canvas/nodes/image-node.tsx`):**
- Reference cards: `rounded-[12px] border border-border bg-surface shadow-sm`
- Image area: fills card minus 30px for label
- Label bar: `border-t border-border bg-surface px-2 py-1.5`, text `font-ui text-[10px] font-semibold uppercase tracking-wider text-ink` (centered)
- Cards have slight rotation (`-1.5°` to `1.2°`) for scattered pinboard feel
- Paste card: `SpinnerGap` icon `h-5 w-5 text-rose/60` rotating, success shows `Check` icon `h-2.5 w-2.5 text-status-done` + "Pasted" `text-[8px] text-status-done`
- Checklist: checkboxes `h-3.5 w-3.5 rounded-[3px]`, checked=`bg-rose border-rose` with white `Check` icon, unchecked=`bg-surface border-border`, spring pop `stiffness: 400, damping: 15`
- Build notes: `rounded-[12px] border-rose/20 bg-[#FFF9F5]`, heading `text-[8px] uppercase tracking-wider text-rose`, typing cursor `h-[11px] w-[1px] bg-ink` blinking 0.6s
- Color swatch: `rounded-[12px] border border-border bg-surface`, swatches `h-3.5 w-3.5 rounded-sm border border-border/60`

**Emotion:** "Oh, this is like Milanote but for my builds. And it actually does things."

---

#### ACT 2: "The Plan" — Timeline (3.8s)

**Nav:** Cursor glides to "Timeline" in sidebar (1.3s dwell). Click. Rose bar slides down.

**Content:** `bg-paper p-6` panel fades in. Then:

1. **Header:** `Clock` icon `h-4 w-4 text-taupe weight="regular"` + `font-display text-lg font-semibold text-ink`
2. **Progress bar:** label `text-[10px] font-medium uppercase tracking-wider text-taupe` + "60%" `text-xs font-semibold text-ink`. Bar `h-1.5 rounded-full bg-border/60`, fill `bg-rose`, 0→60% over 700ms.
3. **Milestones cascade:** `space-y-3`, each row `flex items-center justify-between`, stagger 60ms, slide from x:-12:
   - Status dots: `h-5 w-5 rounded-full bg-surface` containing `CheckCircle h-4 w-4 text-status-done weight="fill"` (done), `Circle h-3.5 w-3.5 text-rose weight="bold"` (active), `Circle h-3.5 w-3.5 text-border` (upcoming)
   - Title: `font-ui text-[11px] font-medium text-ink` (or `text-taupe line-through` if done)
   - Date: `font-ui text-[9px] text-taupe`
   - Status label: `font-ui text-[9px] font-medium`, `text-status-done` / `text-rose` / `text-taupe`

**Emotion:** "I can actually see where I am in my build."

---

#### ACT 3: "The Numbers" — Budget (3.8s)

**Nav:** Cursor to "Budget" (1.3s). Click. Rose bar slides.

**Content:** `bg-paper p-6` panel. Then:

1. **Header:** `CurrencyDollar` icon `h-4 w-4 text-taupe weight="regular"` + `font-display text-lg font-semibold text-ink`
2. **Budget summary card** (matches `materials-budget-section.tsx`): `rounded-lg border border-border bg-paper px-4 py-3`
   - Label: `font-ui text-[10px] font-medium uppercase tracking-wider text-taupe`
   - Amount: `font-display text-2xl font-semibold text-ink`
   - Remaining: `font-ui text-sm font-medium text-status-done`
   - Bar: `h-1.5 rounded-full bg-border/60`, fill `bg-rose`, 0→56% over 700ms
3. **Material rows:** `divide-y divide-border`, each `py-3 flex items-center justify-between`, stagger 60ms
   - Name: `font-ui text-[11px] font-medium text-ink`
   - Category badge: `rounded-full bg-surface-warm px-2 py-0.5 font-ui text-[9px] font-medium text-taupe`
   - Status badge: `rounded-full border px-2 py-0.5 font-ui text-[9px] font-medium`
     - Needed: `border-rose/20 bg-rose/10 text-rose`
     - Owned: `border-border bg-surface-warm text-taupe`
     - Arrived/Tested: `border-status-done/20 bg-status-done/10 text-status-done`
   - Cost: `font-ui text-[11px] font-semibold tabular-nums text-ink`

**Emotion:** "Finally, I can track what I'm spending."

---

#### ACT 4: "The Grid" — Projects Overview (3.8s)

**Nav:** Cursor to "Projects" (1.3s). Click. Rose bar slides to top.

**Content:** `bg-paper p-6`. Then:

1. **Title:** `font-display text-xl font-semibold text-ink`
2. **Project cards** (matches `project-card.tsx`): `grid grid-cols-3 gap-5`, stagger 80ms, slide from y:12
   - Card: `rounded-[12px] border border-border bg-surface shadow-sm hover:shadow-md`
   - Cover image: `h-[110px] bg-surface-warm`, `object-cover`
   - Content: `space-y-2.5 p-3.5`
   - Title: `font-display text-[12px] font-semibold text-ink`
   - Status badge: `rounded-full border px-2 py-0.5 font-ui text-[8px] font-medium`
     - In Progress: `border-rose/20 bg-rose/10 text-rose`
     - Planning: `border-border bg-surface-warm text-taupe`
   - Progress bar: `h-1.5 rounded-full bg-border/60`, fill `bg-rose`, only shown if progress > 0
3. **New project placeholder:** `rounded-[12px] border-dashed border-border bg-surface/50`, "+" `font-display text-2xl text-taupe`

**Emotion:** "I can manage ALL my builds in one place."

---

### Scene Timing Summary

| Scene | Nav Key | Nav Dwell | Content Dwell | Total |
|-------|---------|-----------|---------------|-------|
| References | references | 1,300ms | 9,800ms (5 beats: 3000+1800+1400+2200+1400) | 11.1s |
| Timeline | progress | 1,300ms | 3,800ms | 5.1s |
| Budget | budget | 1,300ms | 3,800ms | 5.1s |
| Projects | projects | 1,300ms | 3,800ms | 5.1s |

Full loop: ~26.4 seconds. References gets the most time (5 micro-interactions). Nav dwell is 1.3s so cursor arrives before content switches.

### Cursor Coordinates (relative to 1100x700 canvas, sidebar 168px wide)

| Scene | Nav Target (x, y) | Content Targets | Action |
|-------|-------------------|-----------------|--------|
| References | (84, 188) | Beat 0: (620, 400), Beat 1: drag to (348, 420), Beat 2: (710, 320), Beat 3: (735, 130), Beat 4: (500, 300) | Paste → Drag → Check → Type → Settle |
| Timeline | (84, 252) | (500, 310) | Hover milestones |
| Budget | (84, 220) | (450, 320) | Hover materials |
| Projects | (84, 156) | (400, 280) | Hover project cards |

### Sidebar Navigation Items

| Index | Icon | Label | Key |
|-------|------|-------|-----|
| 0 | FolderOpen | Projects | projects |
| 1 | ImageIcon | References | references |
| 2 | CurrencyDollar | Budget | budget |
| 3 | Clock | Timeline | progress |
| 4 | Stack | Materials | materials |
| 5 | BookOpen | Build Log | buildlog |
| 6 | Note | Notes | notes |
| 7 | ChatCircle | Assistant | assistant |

## Design System Quick Reference

- **Display font:** Cormorant Garamond (headlines)
- **UI font:** Inter (body, buttons, labels)
- **Hand font:** Caveat (annotations only)
- **Paper:** #F6F1EA | **Surface:** #FBF8F2 | **Warm surface:** #EFE6DC
- **Ink:** #23201E | **Muted ink:** #5E5650 | **Taupe:** #8D8178
- **Rose:** #B65F72 | **Rose deep:** #A24E62
- **Border:** #D8CCC1 | **Charcoal:** #171614
- **Card radius:** ~12px | **Control radius:** ~8px

## Inspiration Sources

- **Milanote** landing page: scroll-driven fake UI demo with cursor
- **Linear**: scroll-scrubbed section transitions (cinematic narrative)
- **Stripe**: lightweight WebGL mesh gradient + CSS skew transforms
- **Vercel**: modular animated beam/gradient components

### Open source references
- nextjs-animated-components (155+ components, Next.js 14 + GSAP + Motion + Lenis)
- Aceternity UI (200+ copy-paste components, React + Tailwind + Motion)
- Magic UI (shadcn companion, animated beams, number tickers)
