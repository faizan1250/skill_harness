---
name: frontend-excellence
description: Enforces studio-grade frontend standards (Linear, Stripe, Vercel) eliminating AI-slop UI. Triggers on building, styling, refactoring, or reviewing any React/Next.js UI, components, layouts, or design systems.
---

# Frontend Excellence

Standard: every surface must be indistinguishable from work shipped by a top-tier design-engineering team. Assume the reader is a senior engineer — no 101 explanations, no hedging, no "best practices are important" filler. Every line below is a rule to apply, not a topic to consider.

## 1. The AI Slop Test

Before writing code, ask: *if I showed this to someone and said "AI made this," would they believe it instantly?* If yes, stop and redesign. A distinctive interface makes people ask "how was this made," not "which AI made this." The tells below are default outputs of undirected generation — not wrong in isolated cases, but overused to the point of being a fingerprint.

**Never ship, by default:**
- Purple-to-pink or blue-to-cyan gradient heroes. This traces to `bg-indigo-500` as a prominent Tailwind default — it saturated tutorials and demos, and it now reads as "unstyled."
- Inter, Roboto, Arial, system-ui, or (increasingly) Plus Jakarta Sans / Geist as an unexamined default. These are safe, not chosen.
- Oversized, uniform border-radius on every element (cards, buttons, inputs, badges, avatars all at the same exaggerated radius). Corners should vary with element scale.
- Generic shadow stacks (`box-shadow: 0 4px 6px rgba(0,0,0,0.1)`) applied without an elevation system.
- Centered-everything hero + symmetric 3-card feature grid as the default page shape.
- Emoji used as functional icons instead of a proper SVG icon system.
- Uniform 8/16/24px spacing with no rhythm — every gap the same size regardless of relationship.
- Glassmorphism/neumorphism applied decoratively, with no material logic for why a surface is glass.
- Framer Motion fade+slide on every element with identical timing — no hierarchy of motion.
- "John Doe" / "Acme Inc." placeholder content, badge-above-headline, colored-left-border callout cards, numbered 1-2-3 step icons as default layout shorthand.
- Bounce/elastic easing on entrances — reads as 2015-era and amateurish.

**Do instead:** pick one bold, specific aesthetic direction before writing any CSS (e.g., "sharp-engineering minimalism," "high-contrast editorial," "warm polished-corporate") and let every token below derive from that choice, the way Linear's dark-first minimalism and Vercel's pure-black-and-white maximalism are structurally different answers to the same brief.

## 2. Typography

- Max 2 font families: one for display/headings, one for body/UI. A mono face for code/data is a third, acceptable exception.
- Avoid Inter/Roboto/Arial as the *only* choice. Reach for typefaces with actual character: Söhne, GT America, Geist (as a considered choice, not a default), Satoshi, General Sans, Neue Montreal, National, Clash Display (display), Fraunces or Instrument Serif (editorial serif accent). Pick based on the brand's voice, not availability.
- Scaling strategy by surface type:
  - **Marketing, landing pages, & headlines:** use fluid scaling via `clamp()` to scale naturally with viewport width:
    ```css
    --text-sm:   clamp(0.875rem, 0.85rem + 0.15vw, 0.95rem);
    --text-base: clamp(1rem, 0.95rem + 0.25vw, 1.125rem);
    --text-h2:   clamp(1.75rem, 1.4rem + 1.5vw, 2.75rem);
    --text-h1:   clamp(2.5rem, 1.8rem + 3vw, 4.5rem);
    ```
    Mix `rem` (accessible, respects user zoom) with a `vw` term (scales with viewport). Never pure `vw` — it breaks zoom.
  - **App UIs, dashboards, & data-dense tables:** use deliberate, stepped token scales (e.g., Tailwind standard text scale) to prevent unpredictable line wraps, clipping, or height oscillations in dense interfaces.
- Line-height inverts with size: tight (1.0–1.1) for large display type, generous (1.5–1.6) for body copy. Letter-spacing tightens slightly (-0.01 to -0.02em) at display sizes, stays neutral in body text.
- Don't interpolate a scale — let ratios jump deliberately (e.g., 16 → 18 → 24 → 32 → 48 → 72 → 96, not evenly stepped). A scale that feels "skipped" rather than "calculated" reads as designed.
- Test type choices in actual copy, not lorem ipsum — generic filler text hides tone mismatches and produces UI copy that reads as AI-generated.

## 3. Color & Design Tokens

- Color format: Prefer OKLCH for perceptually uniform ramps (`oklch(L C H)`), but standard modern CSS color formats (HSL, RGB, or Hex with alpha channels) are fully acceptable when required by Tailwind configs or project tooling. The critical requirement is uniform perceptual contrast, not the specific color syntax.
- Three-tier token architecture: **global** (raw values, e.g. `--blue-500: oklch(0.6 0.2 250)`) → **alias/semantic** (`--color-primary`, `--color-surface-raised`, `--color-danger`) → **component** (`--button-bg-primary`). Components reference alias tokens, never global or literal values.
- Name by role, not appearance: `--color-surface-raised`, never `--gray-100`. This is what survives a rebrand or theme swap without touching component code.
- Tint neutrals with the brand hue where custom palettes are built: `oklch(0.98 0.01 250)` instead of dead sterile gray (`oklch(0.98 0 0)`) — chroma 0.005–0.02 reads as intentional craft. Established neutral palettes (slate, zinc) are also acceptable when aligned with the design direction.
- Multi-theme & Dark Mode: Whether the project is dark-first (Linear/developer tools), light-first (clean B2B/docs), or supports both, themes must use first-class semantic token sets. Never invert lightness algorithmically with CSS filters (`filter: invert()`). If multi-theme is supported:
  ```css
  :root       { --background: oklch(1 0 0);     --foreground: oklch(0.14 0.005 285); --border: oklch(0.92 0.005 285); }
  .dark       { --background: oklch(0.14 0.005 285); --foreground: oklch(0.98 0.005 285); --border: color-mix(in oklch, white 10%, transparent); }
  ```
  In dark modes, prefer translucent borders (`white/10%` or opacity-backed borders) over solid grays, and shift chart/data-viz hues rather than just dimming them — straight lightness inversions look muddy. Single-theme projects (pure dark or pure light) are fully valid when intentional.
- Decide deliberately between pure black/white (Vercel's "consequential" harshness) vs. near-black/off-white (softer, "playful") — this is a brand decision, not a default. Never land in the 60–80% gray band on a pure background; it reads as murky, not neutral.
- Enforce contrast at the token level: every background/foreground pair must hit WCAG AA (4.5:1 body, 3:1 large text) before it's allowed into the palette — check this when defining tokens, not per-component.

## 4. Spacing, Layout & Elevation

- Disciplined spacing scale: Stick to a consistent base unit (typically 4px / 0.25rem: 4, 8, 12, 16, 24, 32, 48, 64, 96, 128px) matching Tailwind or design system tokens.
- Never hand-write arbitrary one-off pixel values in components (e.g. `p-[17px]`) — always pick the nearest design system token or declare a semantic token deliberately.
- Default to intrinsic, content-driven layout over fixed breakpoints:
  ```css
  .grid { display: grid; grid-template-columns: repeat(auto-fit, minmax(min(280px, 100%), 1fr)); gap: var(--space-lg); }
  ```
  Combine with container queries so components adapt to their container, not the viewport — a sidebar card and a full-width card should never share a viewport breakpoint when their available space is completely different.
  ```css
  .card-wrap { container-type: inline-size; }
  @container (min-width: 400px) { .card { grid-template-columns: 200px 1fr; } }
  ```
- Elevation is a token scale tied to a consistent light source, not ad hoc shadows per component:
  ```css
  --shadow-sm: 0 1px 2px oklch(0% 0 0 / 0.05);
  --shadow-md: 0 4px 6px oklch(0% 0 0 / 0.07);
  --shadow-lg: 0 10px 15px oklch(0% 0 0 / 0.10);
  --shadow-xl: 0 20px 25px oklch(0% 0 0 / 0.12);
  ```
  If a shadow is clearly visible at a glance, it's too strong — elevation should be felt, not seen. In dark mode, replace shadows with subtle borders or glows (`border: 1px solid color-mix(in oklch, white 8%, transparent)`); shadows don't read against dark backgrounds.
- **Subtle 3D-ish interactive surfaces ("painted light" technique):** primary CTAs, selected chips, and key interactive cards must not look flat. Apply restrained perceived depth using a vertical gradient + top highlight + bottom lift shadow. Flat elements sitting next to a 3D-ish primary will read as intentionally secondary — this is hierarchy, not neglect.
  ```css
  /* Primary button — 3D-ish, restrained */
  .btn-primary {
    background: linear-gradient(to bottom, hsl(var(--brand-h) var(--brand-s) 58%), hsl(var(--brand-h) var(--brand-s) 48%));
    box-shadow: inset 0 1px 0 rgba(255, 255, 255, 0.25), /* top highlight */
                0 2px 4px rgba(0, 0, 0, 0.20);           /* lift shadow */
    border-top: 1px solid rgba(255, 255, 255, 0.18);     /* alternative to inset highlight */
  }
  .btn-primary:active {
    box-shadow: inset 0 2px 4px rgba(0, 0, 0, 0.20);    /* sinks on press */
    transform: translateY(1px);
  }
  ```
  Rules:
  - Gradient delta: ~8–12% lightness drop top to bottom. More = skeuomorphic. Less = invisible.
  - The highlight + shadow combo is what reads as 3D; gradient alone is a secondary reinforcer.
  - Apply to: primary CTA buttons, selected state pills/chips, primary interactive cards. Never to: flat secondary buttons, nav links, body text containers.
  - In dark mode: keep the inset highlight (it still reads), reduce the lift shadow opacity to ~0.35 and layer it with a `1px solid rgba(255,255,255,0.08)` border so the edge reads against the dark surface.
- Define a z-index scale semantically (`dropdown < sticky < modal-backdrop < modal < toast < tooltip`), never arbitrary numbers per component.

## 5. Motion & Interaction

Motion earns its place when it clarifies state change, spatial relationship, or causality — not as decoration on every mount.

- **Timing scale:**

| Duration | Use |
|---|---|
| 100–150ms | Micro-feedback: hover, press, toggle, color change |
| 200–300ms | Small transitions: dropdown, tooltip, menu open |
| 300–500ms | Medium: modal, drawer, accordion, page-section change |
| 500–800ms | Entrance/hero reveals only — never for routine UI |

  Exit animations run at ~70% of their matching entrance duration — leaving feels faster than arriving.
- **Easing:** never the CSS default `ease`. Use exponential ease-out for elements entering (mimics real deceleration/friction), ease-in for elements leaving, ease-in-out only for toggles that return to their start state:
  ```css
  --ease-out-quart: cubic-bezier(0.25, 1, 0.5, 1);   /* default entrance */
  --ease-out-expo:  cubic-bezier(0.16, 1, 0.3, 1);   /* snappier, confident */
  --ease-in-quart:  cubic-bezier(0.5, 0, 0.75, 0);   /* exits */
  ```
  Avoid bounce/elastic/spring-overshoot as a default — real objects decelerate, they don't bounce. Reserve spring physics for genuinely playful, brand-appropriate moments, and even then keep overshoot minimal.
- Animate only `transform` and `opacity` — anything touching `width`, `height`, `top`, or `left` forces layout and drops frames. Use `will-change` sparingly, only on elements actively animating.
- `prefers-reduced-motion` is mandatory, not optional, on every non-trivial animation:
  ```css
  @media (prefers-reduced-motion: reduce) {
    *, *::before, *::after { animation-duration: 0.01ms !important; transition-duration: 0.01ms !important; }
  }
  ```
- Every animation must be interruptible — a user re-triggering a hover or closing a modal mid-transition should never fight the animation queue.
- Hover, focus, active, and disabled states are each deliberately designed — never left at browser defaults, and never identical to each other.

## 6. State Management

Pick the tool by *category of state*, not by habit. Reaching for global state (Redux/Context-everywhere) by default is the most common unforced architectural error.

| State category | Owned by | Default tool | Notes |
|---|---|---|---|
| Server state (API data) | The server | TanStack Query / SWR | Handles caching, refetch, dedupe. Never copy server data into a client store — read the cache directly. |
| Client UI state (toggles, modals, selections) | The client | `useState`, colocate first | Lift to Zustand/Jotai only once genuinely shared across distant components |
| Cross-tree client state | The client | Zustand (store-centric) or Jotai (atom-centric) | Zustand for a handful of shared values; Jotai when state is naturally decomposed into independent atoms |
| URL state (filters, tabs, pagination) | The URL | `nuqs` / search params | Anything that should survive a refresh or be shareable via link belongs in the URL, not React state |
| Form state | The form | React Hook Form | Don't manage individual field state by hand |

- TanStack Query's `staleTime` defaults to 0, triggering a background refetch on every mount — set an explicit `staleTime` for anything that isn't truly real-time; this is the single most common unnecessary-request bug.
- Colocate state as close to its usage as possible. Lifting state up is a response to a proven need (a sibling actually needs it), not a precaution.
- Never let server and client state live in the same store — cache invalidation and staleness rules for server data don't belong in Redux/Zustand.
- **RSC vs. Client Boundaries:** Default to React Server Components (RSC) for page shells, data fetching, and static layouts. Push `"use client"` down to the leaf nodes where state, hooks, or event listeners are required. Never mark an entire page or large layout as `"use client"` just to handle an interactive button or dropdown.
- **Next.js App Router Data Flow:** Fetch directly in Server Components where possible; use TanStack Query or SWR on the client when background polling, optimistic updates, or client-side caching are required. Pass server-fetched data as initial props or use hydration boundaries.

## 7. Forms & Validation

- Zod (or equivalent schema library) is the single source of truth for validation — never hand-rolled `if/else` chains. Infer types from the schema (`z.infer<typeof schema>`) so client types, server types, and validation never drift apart.
- Validate `onBlur`, then re-validate touched fields `onChange` — this avoids scolding the user mid-keystroke while still healing errors live once shown:
  ```ts
  useForm({ resolver: zodResolver(schema), mode: 'onBlur', reValidateMode: 'onChange' });
  ```
- Errors are inline, under their field, never summary-only or toast-dumped. Error copy lives in the schema (`z.string().min(8, "Use at least 8 characters")`) so messages stay specific instead of generic "Invalid input."
- Accessible wiring is required, not optional, on every field:
  ```html
  <input aria-invalid={!!error} aria-describedby={error ? 'email-error' : undefined} />
  {error && <p id="email-error" role="alert">{error.message}</p>}
  ```
  `role="alert"` (or an `aria-live="polite"` region for async errors) ensures screen readers announce the error without requiring focus to move there manually; move focus to the first invalid field on failed submit.
- Prefer a submit button that's always tappable and validates on click over a silently disabled button — a disabled button with no visible reason is unde­buggable for the user.
- Optimistic UI for mutations: update the cache immediately, roll back `onError`:
  ```ts
  useMutation({ mutationFn, onMutate: async (v) => { /* setQueryData optimistically */ }, onError: (e, v, ctx) => { /* roll back from ctx */ } });
  ```
- Always re-validate on the server with the same schema — client validation is bypassable and is a UX feature, not a security boundary.

## 8. Accessibility (non-negotiable baseline: WCAG 2.2 AA)

- Semantic HTML first; ARIA only patches gaps native elements can't express. A `<button>` beats a `<div onClick>` with six ARIA attributes bolted on.
- Full keyboard parity: every mouse interaction has a keyboard equivalent, and tab order follows visual/logical order.
- Never `outline: none` without a designed replacement. Build a deliberate `:focus-visible` style — WCAG 2.2 requires it not be obscured by sticky headers/footers/overlays (2.4.11 Focus Not Obscured), and at AAA a 2px perimeter with 3:1 contrast against both states (2.4.13 Focus Appearance).
- Interactive targets are at least 24×24 CSS px (2.5.8 Target Size) — pad hit areas on small icon buttons rather than shrinking the visual element.
- Any drag interaction (reorder, sliders) needs a non-drag alternative (2.5.7 Dragging Movements) — e.g., explicit up/down buttons alongside a draggable list.
- Don't force re-entry of previously-provided information in a multi-step flow (3.3.7 Redundant Entry) — auto-populate or offer selection.
- Color is never the sole signal for state (error, success, required) — pair with an icon, label, or pattern.
- Contrast minimums (4.5:1 body text, 3:1 large text/UI components) are enforced at the token level (see §3), not audited after the fact.

## 9. Performance Discipline

2026 Core Web Vitals thresholds are tightening — design to the stricter bar, not the legacy one:

| Metric | Good | Notes |
|---|---|---|
| LCP | ≤2.0–2.5s | Set `fetchpriority="high"` on the LCP image, never `loading="lazy"` on it. Preload it explicitly. |
| INP | ≤150–200ms | Break up long JS tasks; move heavy computation off the main thread (Web Workers); minimize re-renders on interaction. |
| CLS | ≤0.08–0.1 | Reserve space for every image/embed/ad with explicit dimensions or `aspect-ratio`; never let async content shift already-rendered layout. |

- Images: Leverage `next/image` to automate AVIF/WebP conversion, responsive `srcset`, and zero-CLS placeholders (`placeholder="blur"`). Set `priority` on the LCP hero image to preload and inject high fetch-priority; never lazy-load above-the-fold heroes.
- Fonts: Use `next/font` (Google or local) to eliminate FOIT (Flash of Invisible Text) and ensure self-hosted, zero-layout-shift font preloading.
- Avoid layout shift by reserving space up front for anything that loads asynchronously — dynamic banners, cookie notices, injected content must never push already-visible content.
- Bundle discipline: don't pull a 40kb date library for one format call, a full animation library for a single fade, or a full icon font for three icons. Tree-shake, prefer native CSS (transitions, `:has()`, container queries) over JS where it now suffices.

## 10. Code Quality Bar

- Composition over prop-drilling monoliths: compound components (`<Accordion.Root><Accordion.Item>`) for related UI, not one component with fifteen boolean props.
- Variant APIs via `cva` (class-variance-authority) or equivalent — type-safe, declarative variant definitions instead of manual class-string concatenation:
  ```ts
  const button = cva('rounded-md transition-colors', {
    variants: { intent: { primary: 'bg-primary text-primary-foreground', ghost: 'bg-transparent' }, size: { sm: 'h-8 px-3', lg: 'h-11 px-6' } },
    defaultVariants: { intent: 'primary', size: 'sm' },
  });
  ```
- No inline magic numbers for color, spacing, radius, duration, or z-index — every one traces to a token (§§3–5).
- Every component that fetches or mutates data must explicitly handle **loading, empty, error, and success** states — a component that only renders the happy path is unfinished, not done.
- **Error Boundaries & Route Fallbacks:** Localize failure domains. Wrap complex interactive widgets in React Error Boundaries (`<ErrorBoundary fallback={<WidgetErrorFallback />}>`) so an unhandled render exception in one component does not crash the entire application. Leverage Next.js App Router conventions (`error.tsx`, `not-found.tsx`, and `loading.tsx`) for route-level resilience.
- Consistent naming and file structure across the codebase; a new contributor should be able to guess where a given piece of UI lives.

## 11. Pre-Ship Checklist

Before calling any frontend task done, verify:

- [ ] Fails the AI Slop Test (§1) — no default gradients, default fonts, uniform radii, generic shadows, emoji-as-icons, centered-3-card layout
- [ ] Architecture respects RSC boundaries: Server Components by default, `"use client"` isolated to interactive leaf nodes
- [ ] Typography uses deliberate scaling (fluid `clamp()` for marketing/editorial, precise stepped scale for dense app UIs) and ≤2 font families
- [ ] Colors use semantic tokens with verified WCAG AA contrast; multi-theme apps use dedicated token sets, not inversion filters
- [ ] Spacing and elevation trace to defined design system / Tailwind token scales — zero raw magic-number values
- [ ] Motion respects `prefers-reduced-motion`, uses non-default easing, animates only `transform`/`opacity`
- [ ] State is in the right category (server/client/URL/form) — no server data copied into client stores
- [ ] Forms validate via schema, show inline accessible errors, and re-validate server-side
- [ ] Keyboard nav has full parity with mouse; focus is visible and never obscured; targets ≥24×24px
- [ ] Media & fonts leverage `next/image` (`priority` on LCP) and `next/font` for zero CLS/FOIT
- [ ] Every data-driven component handles loading, empty, error, and success states; Error Boundaries prevent blank screens
- [ ] Verified at 3+ real viewport widths, not just one Figma frame's exact pixel width
- [ ] Primary CTAs and selected interactive states use the 3D-ish "painted light" technique (vertical gradient + inset top highlight + lift shadow) — no primary CTA ships with a flat solid fill