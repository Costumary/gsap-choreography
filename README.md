# gsap-choreography

Claude Code skill for building multi-scene, scripted GSAP product animations in React.

Not tweens-101 — this is the orchestration layer: cursor choreography, click systems, scene transitions, typed text, layout morphing, and multi-cursor collaboration on a single coordinated timeline.

## Install

```bash
claude /plugin install github.com/Costumary/gsap-choreography
```

## What it teaches

- **Timeline as central authority** — one timeline, label-based positioning, no desync
- **Cursor choreography** — natural movement, click squeeze + ripple feedback, position measurement
- **Scene transitions** — cross-fading tabs, nav state updates, progress indicators
- **Typed text** — constant-speed character reveal with caret animation
- **Stagger patterns** — organic card placement, list reveals
- **Layout morphing** — sidebar collapse/expand with multi-property orchestration
- **Multi-cursor collaboration** — independent cursor paths on one timeline
- **Animation manager** — React context for play/pause/restart coordination
- **Responsive scaling** — fixed design size with transform-based scaling
- **Accessibility** — `prefers-reduced-motion` support with static fallback

## Live example

See this pattern in production at [costumary.com](https://www.costumary.com) — the hero section runs a 4-scene product walkthrough built entirely with these techniques.

## Requirements

- GSAP 3.x + `@gsap/react` (free since Webflow acquisition)
- React 18+ with `useGSAP` hook
- Works with Next.js, Vite, or any React setup

## License

MIT
