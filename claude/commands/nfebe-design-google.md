---
description: Move a UI toward a Google-style design language: clean, colorful, creative, illustration- and vector-rich. Brand-adaptive, works on any project.
argument-hint: [page, surface, or component to focus]
allowed-tools: Read, Edit, Write, Grep, Glob, Bash
---

Bring the UI toward a Google-style design language: creative, clean, colorful, and rich with illustrations and vector icons. Keep the project's own brand and soften it forward; never rebrand. $ARGUMENTS

## The look

- **Google creativity.** Friendly, modern, a little playful. Confident use of shape, color, and motion. Approachable over corporate or editorial.
- **Cleanliness.** Generous whitespace, clear hierarchy, uncluttered surfaces, Material-style elevation and soft radius. Let content breathe.
- **Color.** Use color generously and on purpose: colorful tiles, accent fills, gradient hero cards. Stay within one ramp; semantic colors carry status (success, warning, danger).
- **Illustrations.** Spot illustrations for empty states, onboarding, success and error screens, and section headers. Vector and on-brand, never stocky.
- **Vectors and icons.** One simple line/vector icon set, consistent weight and size, used creatively for status, accents, and affordances. No novelty or clashing glyphs.

## Operating rules

- **Discover, do not invent.** Read the project's real brand first (logo, primary and secondary colors, fonts, Tailwind or theme config, CSS variables). Match any reference the user names.
- **Tokens before components, components before pages.** Change the smallest shared thing first and let it cascade.
- **One style per role.** A button, a card, a chip each look like one thing everywhere. Remove orphan and legacy styles.
- **Consistency across surfaces.** Marketing, app, and admin share the same system.
- **Restyle, do not restructure** unless asked. Match surrounding code conventions.

## Process

1. **Brand and tokens.** Establish the source of truth: neutral ramp, brand colors, semantic colors, spacing scale, radius scale, elevation and shadow set, type scale, motion. Apply these first.
2. **Base components.** Surfaces, buttons, inputs, chips (pill), cards.
3. **Chrome.** Nav and sidebar (translucent or glassy where it fits; active state is a filled pill, never a faint underline); page background (subtle gradient or brand wash).
4. **Content.** Hero cards (balance, welcome, summary) get a richer treatment such as a gradient or accent fill; pick one and replicate it. Add illustrations to empty, loading, error, and success states.
5. **Sweep.** Grep for hard-coded colors, ad-hoc radii and shadows, and leftover old-system components; replace with tokens.

## Interactions and states

- No native `alert`, `confirm`, or `prompt`. Use custom modals; destructive or admin actions get prefilled templates and state the consequence clearly.
- Design every state: empty, loading, error, success. Motion stays subtle and consistent and respects reduced motion.

## Accessibility

- Contrast at least AA for text and essential UI. Visible focus states, adequate hit targets, keyboard reachable.

## Done

- Each touched surface eyeballed; no leftover editorial or old-system look.
- Tokens are the source of truth; no one-off colors, shadows, or radii remain in touched files.
- Responsive holds; dark mode holds if present; no functional regressions.
