---
name: emil-kowalski
description: Design-engineering playbook for UI animation and interaction polish, based on the publicly shared principles of Emil Kowalski (Sonner, Vaul, animations.dev). Use whenever building, reviewing, or refining motion, transitions, micro-interactions, icon swaps, toasts, drawers, popovers, modals, tooltips, buttons, or gestures — or when the user asks to make UI "feel better", "smoother", "more polished", or "like Emil Kowalski would". Covers when NOT to animate, easing and duration choices, transform-origin, interruptibility, performance, springs and drag gestures, clip-path tricks, accessibility, and a review checklist.
---

# Emil Kowalski–style design engineering

A distillation of the craft principles Emil Kowalski teaches publicly (his blog,
animations.dev, and the source of Sonner and Vaul). This is not written or
endorsed by him; it's a working checklist so Claude makes the same kind of
decisions he advocates.

The core belief: **great interfaces are made of many small details that nobody
notices individually but everybody feels together.** A good animation is one the
user doesn't consciously see — the UI just feels fast, responsive, and alive.

## 1. Decide whether to animate at all

Before writing any motion, answer: *what is this animation for?* Valid purposes:

- **Spatial continuity** — showing where something came from or went (a popover
  growing from its trigger, a drawer sliding from the edge).
- **Feedback** — confirming an action happened (button press, copy → check).
- **State change explanation** — making a change easy to follow instead of jarring.
- **Delight** — only for rare moments, never for things seen hundreds of times.

Do **not** animate when:

- The action is **high-frequency** (command palettes, context menus opened via
  keyboard, list navigation, tab switching in a tool used all day). Repetition
  turns delight into delay. A command menu opened with ⌘K should appear instantly.
- The action is **keyboard-initiated** and the user is moving fast.
- The animation only exists because it looks cool in isolation.

When in doubt, make it faster or remove it. "No animation" beats a bad one.

Marketing / landing pages can be more expressive; product UI should be restrained.

## 2. Easing

- **ease-out** for anything entering or exiting (dropdowns, modals, toasts,
  tooltips). It starts fast, so the UI feels instantly responsive.
- **ease-in-out** for elements that move or morph *while already on screen*
  (a slider thumb, a layout shift, a tab indicator).
- **Avoid ease-in** for UI — it starts slow and feels sluggish exactly when the
  user is watching for a response.
- **linear** only for constant motion: progress bars, spinners, marquees, the
  "hold to confirm" fill.
- The built-in CSS curves are weak. Use stronger custom curves:

```css
:root {
  --ease-out: cubic-bezier(0.23, 1, 0.32, 1);
  --ease-in-out: cubic-bezier(0.77, 0, 0.175, 1);
  --ease-drawer: cubic-bezier(0.32, 0.72, 0, 1); /* iOS-like sheet curve (Vaul) */
}
```

## 3. Duration

- Most UI transitions: **150–250ms**. Keep UI motion **under 300ms**.
- Small elements (tooltips, icons, button feedback) are shorter: ~100–200ms.
- Larger surfaces travelling further (drawers, sheets, page-level) can go to
  ~400–500ms with a strong ease-out curve.
- Exits may be **faster** than entrances; the user already decided to dismiss.
- Asymmetry is fine and often better: hold-to-delete fills slowly (e.g. 2s
  linear) but snaps back quickly (~200ms ease-out) on release.

## 4. Make it feel physical

- **Never animate from `scale(0)`.** Nothing in the real world appears from
  nothing. Start from `scale(0.9–0.97)` combined with `opacity: 0`.
- **Origin-aware popovers.** Dropdowns, popovers and tooltips should scale from
  their trigger, not their center. With Radix use
  `transform-origin: var(--radix-popover-content-transform-origin)` (or the
  equivalent var for dropdown/tooltip/select). **Modals are the exception** —
  they're not anchored to a trigger, so keep them centered.
- **Press feedback.** Buttons get `transform: scale(0.97)` on `:active` with a
  short transition. It makes the interface feel like it's listening.
- **Blur to hide imperfection.** When cross-fading two states (icon swaps, text
  changes, morphs) add a small `filter: blur(2px–4px)` to the outgoing/incoming
  state; the eye reads it as a single smooth transformation.
- **Stagger** groups subtly (≈30–80ms between items); never make the user wait
  for a long stagger to finish before they can act.

## 5. Interruptibility

UI must respond to the user mid-animation.

- Prefer **CSS transitions** (which retarget from the current value) over
  `@keyframes` (which restart) for anything the user can toggle quickly — toasts,
  hover states, open/close. Sonner uses transitions for exactly this reason.
- For gesture-driven or highly interactive motion, use **springs** (Motion /
  framer-motion, react-spring). Springs carry velocity through interruptions
  and feel natural. A good default for UI: high stiffness, enough damping that
  it settles without visible wobble unless the bounce is the point.
- Use `@starting-style` (or mount-then-transition) for CSS entry animations so
  they can still be interrupted.

## 6. Performance

- Animate only **`transform` and `opacity`** (and `filter`/`clip-path` with
  care). Never animate `width`, `height`, `top`, `left`, `margin`, `padding`.
- Never use `transition: all`; list the exact properties.
- **Framer Motion / Motion:** the `x`, `y`, `scale` shorthands are driven by JS
  on the main thread; under load they drop frames. When it matters, animate a
  full `transform` string (e.g. `animate={{ transform: "translateX(100px)" }}`)
  so the browser can hardware-accelerate it.
- **Avoid updating CSS variables on a parent during drags.** Changing a variable
  invalidates styles for every descendant. Set `transform` directly on the
  moving element (a lesson from Vaul).
- Use `will-change` sparingly and only on elements that are about to animate.
- Verify at 60fps: open DevTools → Performance, and use the Animations panel to
  slow animations down to 10–25% to see every frame.

## 7. Gestures and drag (drawers, sheets, swipe-to-dismiss)

- Dismiss based on **velocity, not just distance** — a quick flick should close
  even if it travelled a short way (Vaul uses a small velocity threshold, ~0.4).
- Apply **friction / rubber-banding** when dragging past a boundary instead of
  hard-stopping; resistance should increase the further you pull.
- Once the user starts dragging, the element must follow the pointer 1:1 with no
  transition; re-enable the transition only on release.
- Ignore drags that start on scrollable content that isn't at its scroll edge.
- Use pointer capture so the gesture survives the pointer leaving the element.

## 8. clip-path tricks

`clip-path` is animatable, GPU-friendly, and avoids layout. Use it for:

- **Tabs with perfect color transitions:** render a duplicate tab list with the
  "active" styling on top, clipped with `clip-path: inset(...)` to the active
  tab; transition the inset. The text color changes exactly at the indicator edge.
- **Hold-to-confirm buttons:** an overlay revealed with `inset(0 100% 0 0)` →
  `inset(0 0 0 0)` while pressed.
- **Image comparison sliders and reveal-on-scroll** effects.

## 9. Small details that compound

- **Tooltips:** delay the first one (~300–700ms), then show subsequent ones
  instantly *and without animation* while the user moves across a toolbar.
- **Toasts (Sonner patterns):** pause the dismiss timer on hover and when the
  tab is hidden (`document.visibilityState`); stack with scale + offset and
  expand on hover; fill gaps between stacked toasts with a pseudo-element so the
  hover state doesn't flicker; allow swipe to dismiss.
- **Hover on touch:** wrap hover effects in
  `@media (hover: hover) and (pointer: fine)` so taps don't leave sticky hovers.
- **Icon swaps** (copy → check, play → pause, menu → close): cross-fade with
  `opacity` + `scale(0.25–0.5 → 1)` + small `blur`, ~150–200ms ease-out. Keep both
  icons in the DOM (absolutely stacked) or use `AnimatePresence` with
  `mode="popLayout"`/`initial={false}` so nothing jumps.
- **Numbers and text that change:** animate per-character or use a blur +
  translateY roll; keep `font-variant-numeric: tabular-nums` so width is stable.
- Prevent layout shift: reserve space, keep dimensions stable between states.

## 10. Accessibility

- Respect `prefers-reduced-motion: reduce`. Remove movement (translate, scale,
  springs, parallax) but **keep opacity/color transitions** so state changes
  remain understandable. In Motion use `useReducedMotion()` or `MotionConfig
  reducedMotion="user"`.
- Never convey meaning through motion alone.
- Animations must never block input or focus.

## 11. Developing taste (how to review your own work)

- Slow it down (DevTools Animations panel) and scrub frame by frame.
- Look at it again later with fresh eyes; most problems become obvious.
- Compare against products known for craft (Linear, Vercel, Raycast, Apple) and
  ask *why* theirs feels better — usually it's easing, duration, or origin.
- Test on a real phone and with a trackpad, mouse, and keyboard.

## Review checklist

When asked to build or review UI motion, walk through this list and call out
violations explicitly:

1. Does this animation have a purpose? Is it on a high-frequency or
   keyboard-driven path where it should be removed?
2. Enter/exit uses ease-out with a custom curve? No ease-in? No default `ease`?
3. Duration ≤ 300ms for UI (longer only for large surfaces/drawers)?
4. Nothing scales from 0? Popovers originate from their trigger; modals centered?
5. Buttons have press feedback?
6. Interruptible (transitions/springs rather than restarting keyframes)?
7. Only `transform`/`opacity` (and careful `filter`/`clip-path`) animated? No
   `transition: all`?
8. Hover effects gated for fine pointers?
9. `prefers-reduced-motion` handled?
10. No layout shift; dimensions stable across states?

See `references/snippets.md` for copy-paste implementations of these patterns.
