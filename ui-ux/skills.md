---
name: ui-ux-elevation
description: Use this skill whenever asked to improve, polish, modernize, redesign, elevate, "make less AI-generated," de-slop, or make an EXISTING UI/UX feel premium — not build new screens from a blank slate. Trigger on phrases like "make this look better/more professional/less generic," "this looks AI-made," "elevate the design," "audit the UI," "make this pop more," "add depth," "make it feel physical/tactile," "premium/professional UI," or any flat-design critique, and when reviewing/refactoring any shipped frontend for visual quality — including tactile 3D-ish buttons, cards, tabs, badges, toggles, or panels. Do NOT trigger for net-new greenfield screens with no existing design to diagnose.
---

# UI/UX Elevation

Goal: the result reads as deliberate work by a senior product designer at a top studio — zero visible fingerprints of "an AI touched this." Not a reskin. Not a different set of defaults.

## The Core Law: Audit Before You Touch Anything

No edits until the existing UI has actually been looked at — screens, components, states, flows. Elevating a UI without first diagnosing what's wrong just replaces one arbitrary look with another.

Before proposing changes, be able to name specifics, not vibes:
- Exact hierarchy failures (what competes with what, what has no focal point)
- Spacing/sizing values that don't belong to any coherent scale
- Missing or neglected states (loading, empty, error, disabled)
- Dated patterns (skeuomorphic depth, unearned skeuomorphism, decorative-only motion)
- Accessibility gaps (missing focus indicators, insufficient contrast, no keyboard path)
- Where the same component is styled differently in different places (drift)
- Whether depth is being spent on things that should not win the eye — e.g. one hyper-3D button next to flat siblings, or 3D applied to static text and labels, which drowns hierarchy (see Depth Tiers below)

Use a lightweight heuristic pass modeled on Nielsen's 10 usability heuristics (visibility of system status, user control, consistency, error prevention, recognition-over-recall, flexibility, minimalist design, error recovery, help/docs, real-world match) to structure the audit — this is the industry-standard method for catching issues systematically rather than impressionistically. A single reviewer catches ~35% of issues; be thorough rather than fast.

Also check layout-quality issues a human eye catches that no rule list will: text clipping/overflowing containers, near-misalignment, mixed one-off spacing values with no scale, sections that are all the same visual weight (no hierarchy at all).

## Slop Detection — And the Trap of "Fixing" It Wrong

**The critical meta-rule: never replace one unearned default with another unearned default.** The purple-gradient/Inter/3-card look was the 2024 tell. By 2026 the "fix" for it — warm cream background + serif display font (Instrument Serif, Fraunces, Playfair) + sage/forest-green accent — became its own instantly-recognizable AI tell, precisely because it was *also* nobody's actual decision, just the model's new median. Anti-slop advice itself goes stale. Do not treat any specific palette or font pairing in this document (or anywhere) as "the fix" — treat the *process* below as the fix.

**Current tells to eliminate on sight (both waves — check for either):**
- Indigo/violet/purple as the primary/CTA color with no brand rationale (`#6366f1` and neighbors)
- The "tasteful default": cream/beige page background + serif display headline font + sage/forest green accent, used with equally no rationale
- Untouched shadcn/Tailwind defaults: `slate`/`zinc` neutral cards, `rounded-lg border shadow-sm` repeated everywhere, default `--radius` and `--primary` left generated
- Default sans (Inter, Geist, Roboto, system-ui) with no deliberate hierarchy — but note the "tasteful" serifs above are now equally overexposed, not a safe swap
- Gradient text (`bg-clip-text text-transparent`) on headlines or body copy
- Symmetric 3–6 identical rounded cards as the default answer to "a list of things"
- One large border-radius and one shadow value applied uniformly with no elevation logic
- Fade-in-up / scale-on-hover applied to every element regardless of whether it communicates anything — and hover glow/lift, which are the biggest tells of a cheap or gamified UI
- Emoji standing in for functional UI icons
- Centered-everything hero + 3 feature cards + CTA band as the page skeleton, chosen by default rather than by what the page is actually for
- Unprompted neon glow on dark mode (`shadow-[0_0_*]` on saturated colors)
- Placeholder microcopy ("Transform your workflow," generic empty states, generic button labels)
- Glassmorphism/neumorphism applied decoratively rather than to solve a real depth/layering problem
- Candy-colored 3D buttons, 16px+ pill radius on controls under 200px wide, huge soft shadow blurs, and shadows tinted to match the fill (see the depth calibration table below)

**The deeper principle:** these patterns are the statistical median of training data — the visible absence of a decision, not an ugly choice. The fix is never "swap in a different unearned default" (e.g., emerald instead of violet, a serif instead of Inter). The fix is a specific, sourced decision:
1. Anchor to a real reference — a brand asset, an existing product screenshot, a named direction (editorial, brutalist, utilitarian-dense, warm-consumer, technical-mono) — not "modern and clean," which is not a direction.
2. State the color decision and where it came from (brand guideline, sampled from an existing asset, a stated reason) — not the framework default, not the currently-fashionable alternative.
3. State the type decision and why — a pairing chosen for tone, not whatever autocompletes.
4. Let layout structure follow what the page is actually for, not a template skeleton.

If a flagged pattern turns out to be a real, working brand decision — not a default — leave it. The target is unspecified defaults, not specific colors.

## What "Premium" Actually Means

Premium is restraint, consistency, and resolution across every state — not more visual effects. Flashy decoration on top of inconsistent fundamentals reads as cheaper, not more premium; fix fundamentals first. A second, separate failure mode exists: flat design is efficient but reads as cheap when a product wants to feel crafted, physical, precise. The cure is not "add a shadow" — it is simulating a light source hitting a physical object with thickness. Done right (Stripe dashboards, Linear, macOS controls) it reads as expensive. Done wrong (candy-colored gradient buttons, glowing everything, shadows on every card) it reads as a 2014 skeuomorphic parody or a childish mobile game. Depth is the section below.

Concrete techniques top studios actually use:
- **Double-whitespace rule**: when separating distinct sections or nested component groups, use roughly double the padding used for elements within a single group. Inconsistent "medium" spacing everywhere is what makes layouts feel cramped and undifferentiated.
- **Grayscale-first pass**: when reworking a layout, prove hierarchy, spacing, and alignment in black/white/gray before reintroducing color. Color can hide a weak layout; grayscale can't.
- Every layer — message, spacing, type, imagery, motion, empty states — must feel like someone decided it on purpose. Generic stock imagery or placeholder-feeling copy undercuts everything else, no matter how refined the layout is.
- AI-native products in 2026 are moving toward AI as invisible infrastructure, not a badged feature — "Powered by AI" banners and AI-sparkle iconography on every touchpoint now read as dated, not cutting-edge, if the product isn't otherwise excellent.

## Depth: Simulating Light and Thickness

### The Physical Model

Every "3D" surface fakes the same four things at once. Skip one and it looks flat; overdo one and it looks fake.

1. **A light source.** Pick one direction — almost always from above — for the entire interface, and every highlight and shadow must agree with it. A button lit from above-left and a card lit from below in the same screen is the #1 tell of an unconsidered design. Skip harsh black drop shadows: combine a subtle top highlight (light inset/border) with a soft bottom shadow, the way real light behaves — that raises cards and modals off the page convincingly.
2. **Material thickness.** A real button has a top face and edges. Simulate this with **split borders**, not one uniform border: top border lightest (catches the light), left/right mid tone, bottom darkest (in shadow, grounds the object).
3. **Surface curvature.** A subtle top-down gradient across the face — not the border — suggests the surface is slightly domed or brushed metal rather than a flat-colored rectangle.
4. **Cast shadow + contact.** The object sits above the page, so it casts a soft shadow onto what's behind it, separate from the inner bevel effects on the object itself.

Every technique below is one of these four things. If you can't name which of the four an effect is doing, cut it.

### The Layered Recipe

Flat buttons and cards look unfinished compared to counterparts with even a hint of perceived depth. Apply a restrained 3D feel to interactive surfaces using this layered stack, in order, low to high:

```
1. Base fill:          gradient, top to bottom, ~8–12% lightness drop   (or a neutral surface)
2. Top border:         lightest of the three border tones              ← the highlight
3. Side borders:       mid tone, same hue                             ← neutral edge
4. Bottom border:      darkest of the three                           ← the grounding shadow
5. Inner top sheen:    1px inset highlight, white/light at low opacity  ← light catching the top inner edge
6. Cast shadow:        soft, tight, neutral dark, low opacity         ← lifts it off the page
7. Active state (on click, not hover): 1px downward shift + swap the inset sheen for a dark inset  ← physically depresses
```

The structure is fixed; the exact numbers are a starting point to tune per surface until the shadow language matches the surrounding UI. A lightness drop over 12% reads skeuomorphic, under 8% is invisible — the 8–12% band is the whole signal that the eye reads as "pressable," not "bevelled." Keep shadow color neutral black-based even on colored elements; never tint it to match the fill (see the calibration table below).

Tailwind, dark UI (e.g. a blue primary button on a dark-zinc dashboard):

```html
<button class="
  bg-gradient-to-b from-blue-500 via-blue-600 to-blue-700
  border-t border-t-blue-300/60 border-x border-x-blue-600/60 border-b border-b-blue-900
  shadow-[inset_0_1px_0_rgba(255,255,255,0.25),0_2px_6px_rgba(0,0,0,0.25)]
  hover:shadow-[inset_0_1px_0_rgba(255,255,255,0.3),0_3px_8px_rgba(0,0,0,0.3)]
  active:translate-y-[0.5px] active:shadow-[inset_0_2px_4px_rgba(0,0,0,0.35)]
  transition-all duration-150 rounded-[7px]
">
```

Plain CSS, same button, for non-Tailwind stacks:

```css
.btn-primary {
  background: linear-gradient(to bottom, #3b82f6, #2563eb, #1d4ed8);
  border-top: 1px solid rgba(147, 197, 253, 0.6);
  border-left: 1px solid rgba(37, 99, 235, 0.6);
  border-right: 1px solid rgba(37, 99, 235, 0.6);
  border-bottom: 1px solid #1e3a8a;
  border-radius: 7px;
  box-shadow: inset 0 1px 0 rgba(255,255,255,0.25), 0 2px 6px rgba(0,0,0,0.25);
  transition: all 0.15s ease;
}
.btn-primary:hover  { box-shadow: inset 0 1px 0 rgba(255,255,255,0.3), 0 3px 8px rgba(0,0,0,0.3); }
.btn-primary:active { transform: translateY(0.5px); box-shadow: inset 0 2px 4px rgba(0,0,0,0.35); }
```

**Light-mode inversion.** The stack above assumes a dark surface. On a light background, invert the polarity — don't reuse the dark numbers verbatim:
- Base gradient still runs top→bottom but stays *light* (e.g. `#ffffff` → `#f3f4f6`), barely darkening: light surfaces tolerate far less gradient contrast before looking dirty.
- The highlight border (top) becomes near-white or the surface's lightest tint; the shadow border (bottom) becomes light-mid gray, never near-black — a light card with a near-black bottom border looks like a mistake, not a shadow.
- The inset top sheen can often be dropped entirely; light surfaces don't need a highlight to read as lit since the page is already bright.
- Keep the cast shadow neutral dark (`rgba(0,0,0,0.08–0.15)`) but with larger blur / lower opacity than the dark-mode version — light-mode shadows should read as soft ambient occlusion, not a hard drop shadow.

For dark-UI chrome (toolbars, secondary buttons, table headers) use the same stack in neutral zinc/gray instead of a saturated hue — this is what separates "premium dashboard" from "toy app." Reserve saturated-hue depth for the one or two actions per screen that should command attention: primary CTA, destructive action, live/active state.

### Calibrating Depth: Premium vs. Childish

| Signal | Premium (good) | Childish (bad) |
|---|---|---|
| Border radius | 5–9px. Enough to soften, not enough to look like a bubble; small controls (chips, small buttons) stay at 5–7px, only large surfaces (modals, hero cards) go bigger. | 16px+ on small controls — "pill" radius on a button under 200px wide reads as a toy. |
| Shadow blur/spread | Tight: `0 2px 6px rgba(0,0,0,.25–.35)`. Shadow stays close to the object. | Huge soft blur, `0 20px 40px`, shadow color tinted to match the fill — looks inflated, like a balloon. |
| Gradient contrast | Subtle: 1–2 shade steps within the same hue (`from-zinc-800 to-zinc-900`, `from-blue-500 to-blue-700`). | High-contrast glossy "candy button" gradient (bright highlight streak top, saturated color bottom) — too much dimensionality on a low-importance control. |
| Border treatment | Split by edge (top light / bottom dark), each 1px, low opacity (`/40–/70`). | Single uniform thick border, or a glow/outer-ring border. |
| Inner highlight | One thin inset top sheen, `rgba(255,255,255,.08–.25)`. Suggests light, doesn't compete with content. | Full glassy overlay gradient across the whole face — looks like a sticker, not a material. |
| Hover/active feedback | Minimal or none. At most one hover signal (shade or border-opacity shift), fast/undramatic transition. Active/pressed feedback on click can be slightly more (see the recipe's step 7). | Large scale/bounce animation, color flip, glow pulse, growing shadow, slow/bouncy easing — feels like a mobile-game UI. |
| Where it's used | Applied consistently to *all* interactive chrome at one intensity tier, with reserved intensity for 1–2 primary actions. | Applied unevenly — one hyper-3D button next to flat siblings — or applied to *everything* including static text/labels, which drowns hierarchy. |
| Color of the effect | Neutral/monochrome shadows (black-based, low opacity) even on colored elements. | Colored shadows/glows matching the fill — the single fastest way to look amateur. |

**Rule of thumb:** if removing the effect wouldn't tell you where a person's eye should go next, the effect was structural (keep it). If removing it just leaves a calmer design with no loss of hierarchy, it was decorative (cut it).

A secondary action beside a primary one (a "Know More" beside a "Get Started" CTA) usually shouldn't get the full recipe at all — it belongs at Tier 1 below: near-flat fill, one thin uniform border, no gradient, no glossy highlight, minimal or no cast shadow. Elevation is a budget spent on the things that should win the eye; a secondary button competing in glossiness with the primary defeats the point of having a primary.

```css
/* Before (childish): glossy gradient, big diffuse shadow, pill radius on a small
   text button — competes with the primary CTA instead of stepping back */
.btn-secondary-bad {
  background: linear-gradient(to bottom, #ffffff, #f0f4ff);
  border-radius: 24px;
  box-shadow: 0 8px 20px rgba(100, 140, 255, 0.35); /* diffuse + color-tinted */
}
/* After (premium): near-flat, quiet, clearly subordinate to the primary button */
.btn-secondary-good {
  background: #ffffff;
  border: 1px solid #e5e7eb;
  border-radius: 7px;
  box-shadow: 0 1px 2px rgba(0, 0, 0, 0.06); /* tight, neutral, barely there */
}
.btn-secondary-good:hover { border-color: #d1d5db; }
```

### Depth Tiers: A Hierarchy Signal, Not a Blanket Texture

Assign depth intensity by importance — the same treatment everywhere stops meaning anything:

- **Tier 0 — flat.** Page background, static text, dividers, table body rows at rest. No shadow, no gradient.
- **Tier 1 — subtle lift.** Cards, panels, input fields, table headers, *and secondary/tertiary buttons*. Thin border, faint gradient (2 stops max) or none, a soft ambient shadow or top sheen only — `shadow-[inset_0_1px_0_rgba(255,255,255,0.03)]` is enough for a resting card, and is also the right ceiling for a "cancel" or "learn more" button.
- **Tier 2 — interactive chrome.** Primary/important buttons, dropdowns, tabs, toggles — the full recipe above, neutral hue by default unless it's also the screen's main action.
- **Tier 3 — primary action / status.** The one or two things per screen that should visually win: primary CTA, an active/live pulse indicator, a selected-state highlight. Saturated hue, the fullest version of the recipe, the only place an `animate-pulse` or glow (`shadow-[0_0_6px_rgba(...)]`) belongs.

A screen where every card, button, and badge is Tier 3 has no hierarchy left — that's what makes an interface feel busy and cheap. A well-executed dashboard mostly lives in Tier 0–1, with Tier 2 on real controls and Tier 3 spent on maybe three elements total. (Reference: on the amber "Pipeline Live" dashboard, the amber live dot, the primary batch-delete button, and the selected-row highlight are the only high-saturation 3D moments on the whole page — everything else is neutral zinc at Tier 1–2.)

### Reference Pattern: Status and Selection Accents

Tinted translucent chips for state (status badges, active filters, "fix" branch tags) from the same family: background is the accent color at 5–15% opacity layered as a gradient (`from-amber-500/15 via-amber-500/10 to-amber-500/5`), border is the same hue at 30–40% opacity, inner sheen stays neutral white at low opacity. This reads as "a colored glass chip sitting on the surface" rather than a solid sticker, and scales down to small badge sizes where a full bevel stack would be too heavy.

## Visual Hierarchy

- One clear focal point per screen — if everything is emphasized, nothing is. Choreograph secondary focal points in a deliberate order if the screen has more than one job.
- Build hierarchy from size + weight + color + spacing together, not size alone.
- Whitespace is a hierarchy tool, not leftover space: isolate what matters, use proximity to group what's related, tighten spacing for secondary details near a primary action.
- Depth is the fourth hierarchy channel, alongside size, weight, color, and spacing: assign tiers by importance, and confirm on the walkthrough that nothing low-priority is out-competing the primary action. Flat surfaces next to a depth-tiered primary CTA look *intentionally* flat — that's hierarchy, not neglect.
- Progressive disclosure on dense/complex screens: surface only what's needed now (common options, primary state), reveal advanced/secondary options on demand rather than showing everything at once. This is the standard technique for reconciling "powerful" with "simple."

## Typography Elevation

- Establish or fix a real type scale using a deliberate ratio (common: 1.125, 1.25, 1.5) from a single base size — not sizes scattered ad hoc across the codebase.
- Define 5–7 named text roles (Title, Subtitle, Body, Label, Caption, etc.) rather than raw sizes; each needs a clear, distinct purpose.
- 1–2 font families with clear roles (display/heading vs. body). If choosing a pairing, have a stated reason (tone, contrast) — and specifically avoid reaching for either overexposed default cluster: Inter/Geist/Roboto ("I didn't choose") or Instrument Serif/Fraunces/Playfair ("I tried to look tasteful"). Either is fine as an actual, justified decision; neither is fine as a fallback.
- Fix line-height and letter-spacing per size tier — default/unset values are a tell at any size beyond body text. Larger display sizes generally want tighter letter-spacing and tighter line-height; body text wants more generous line-height for readability.
- Use fluid, clamp-based sizing for responsive hierarchy where the stack supports it, instead of hard breakpoint jumps.

## Color & Theming Elevation

- Move from literal hard-coded color values to a three-tier token architecture if not already present: **primitive** (raw values, e.g. `blue-500: #3b82f6`) → **semantic** (intent, e.g. `action-primary`, `surface-default`, `text-muted`) → **component** (local overrides). UI code should reference semantic tokens, not primitives, so a rebrand or theme change touches one mapping layer instead of hundreds of call sites.
- Name semantic tokens by function, never by hue (`color-action-primary`, not `color-blue-primary`) — hue-locked names break the moment the brand color changes.
- Any accent/brand color must be a stated, sourced decision — not defaulted to indigo/violet, and not defaulted to the "safe" alternative either (see Slop Detection above).
- Dark mode, if present or requested, is its own designed system, not an inverted filter: build surface hierarchy (distinct elevation levels, not one dark gray everywhere), re-verify contrast and accent colors independently in dark mode, and consider true near-black surfaces deliberately (real power/legibility benefit on OLED, not just aesthetic). A depth recipe tuned for one polarity must be inverted, not reused, in the other.
- Verify contrast ratios as part of the base design pass, not a separate afterthought bolted on at the end — and re-verify them *through* any gradient or tint you added, since a gradient's lightest and darkest stops both count as adjacent colors.

## Motion & Micro-interaction Elevation

- Every animation must justify itself: feedback (something happened), continuity (spatial relationship preserved), or state change. If you can't state the benefit in one sentence, cut it.
- Timing: ~100–150ms for hover/press micro-feedback, 150–300ms for simple transitions, 300–500ms for more complex/composed transitions. Consistency across the interface matters more than the exact numbers — define motion tokens (e.g. `motion-quick: 150ms ease-out`) rather than hand-picking per component.
- Easing: `ease-out` for things entering/responding to input, `ease-in` for things exiting, `ease-in-out` for continuous motion (toggles). Never linear — it reads as mechanical.
- Hover is the most over-used effect, and where depth treatments turn cheapest. A real object doesn't change shape, glow, grow, or shift color when your eyes pass over it — it does that when you touch it. Treat hover as a cursor-presence hint, not a celebration:
  - Default to the smallest legible change: a shade shift (`bg-zinc-800` → `bg-zinc-750`), a border going from `/60` to `/80` opacity, or nothing at all when the element is already obviously interactive. On a dense screen — table, list, toolbar — no hover state is often the most premium choice.
  - Pick at most one signal. Never combine a shadow change *and* a color change *and* a transform *and* a border change.
  - Never scale, bounce, lift-on-Y, or add/intensify a glow on hover, and don't dramatically enlarge the cast shadow (`0 2px 6px` → `0 3px 8px` is the ceiling) — a button that grows or floats away when the cursor merely passes it reads as a mobile-game asset, not software.
  - Transition duration stays fast and undramatic: 100–150ms, `ease`. Slow (300ms+) or bouncy (overshoot) hover easing is a childish signal on its own, independent of the property animating.
  - Reserve visible hover flourish for the one or two Tier 3 elements on screen. Everything at Tier 0–1 gets a bored, almost invisible hover state, if any.
  - In doubt, ship with no hover effect and add the absolute minimum only if a real user genuinely can't tell the element is interactive.
- Respect `prefers-reduced-motion` always; the interaction's functional meaning (state changed, error shown) must survive with motion removed.
- Hover, focus, active/pressed, and disabled states must each be visually distinguishable from each other and from rest — not one generic browser default left untouched.

## Component State Completeness

For every component being elevated, verify and design: **rest, hover, focus, active/pressed, disabled, loading, empty, error** (and success where relevant) — not just the populated happy-path appearance. A UI that only looks finished in its ideal state reads as unfinished regardless of how good that one state looks. Derive states systematically from the base component styling (e.g., a consistent hover = darken/lighten by a fixed token) rather than choosing each one arbitrarily.

Active/pressed must physically read as pushed in — a subtle downward shift plus the inset sheen swapped for a dark inset shadow (`0 2px 4px inset`) — not merely a color swap.

Focus indicators specifically: never remove with `outline: none` without a compliant replacement. WCAG 2.4.13 requires the focus indicator to have at least 3:1 contrast against adjacent colors and be at least as large as the component's perimeter — treat this as a design requirement, not just a compliance checkbox.

## Consistency & Drift Audit

Real UIs accumulate drift in five distinct ways — check for all five, they're not the same problem:
- **Token drift** — spacing/color/radius/elevation/type values that no longer map to any defined scale (a handful of stray values usually explain most of the drift; fix the top offenders rather than patching every instance individually)
- **Component drift** — the same logical component (e.g. "card," "button") styled differently in different places because someone improvised a one-off
- **State drift** — loading/empty/error/hover/focus states that don't line up with each other across similar components
- **Behavior drift** — interactions that differ across screens even when the components look related
- **Light-source drift** — components whose top-highlight/bottom-shadow pairings disagree in direction or strength, which reads as unconsidered even when each element looks fine alone

Distinguish real drift (accidental, to fix) from intentional variation (a primary vs. secondary button is *supposed* to look different) — don't flatten meaningful distinctions while fixing accidental ones. Where real duplication exists, consolidate into shared tokens/components as part of the elevation, not just visually patched instance-by-instance.

## Accessibility as a Design Quality Signal

Treat accessibility gaps as design maturity gaps, not a separate compliance task run after the "real" design work. Keyboard navigability (logical tab order, no traps), visible high-contrast focus states, sufficient text/UI contrast, and semantic HTML structure are markers of a carefully made product — their absence reads as neglect to anyone who notices, the same way misaligned padding does. Depth work is not exempt: contrast must hold through the gradient, focus states must stay visible against it, and motion must stay subtle enough to respect `prefers-reduced-motion`.

## Before Shipping, Check

1. **One light source.** Every top-light/bottom-dark pairing agrees across every component on screen.
2. **Shadows are neutral**, not color-matched to the fill, unless it's a deliberate Tier-3 glow on an active/live element.
3. **Radius scales with size**, and depth intensity matches importance — walk the screen and confirm nothing low-priority out-competes the primary action.
4. **Active/pressed states exist** and physically read as pushed in.
5. **Hover is minimal or absent** — one signal at most, fast/undramatic, no scale/bounce/growing glow. If you can't justify why an element needs hover at all, remove it.
6. **Still accessible** — focus states visible, contrast holds even through the gradient, motion subtle enough for `prefers-reduced-motion`.

## Final Elevation Report Format

After making changes, report three things:
1. **What was diagnosed as broken or generic**, with specifics (not "it looked basic" — name the exact hierarchy/spacing/state/consistency issue and where).
2. **What specific decisions were made and why** — the actual color source, the type pairing and its rationale, the spacing scale adopted, the depth tier assigned per surface, the motion tokens defined. Never "made it more modern."
3. **What was intentionally left unchanged** because it was already a deliberate, working design choice — not everything that looks unusual is a slop pattern; don't flatten real intent while fixing accidental drift.