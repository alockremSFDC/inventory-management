---
name: saas-sidebar-redesign
description: Redesign a Vue 3 application's UI into a modern SaaS-style interface with a left vertical sidebar instead of a top nav, design tokens, consistent spacing, and a polished professional look. Use this skill when asked to restyle, modernize, or redesign a Vue 3 UI, replace a top navigation bar with a sidebar, or add a consistent design system to an existing app.
---

# SaaS Sidebar Redesign

Turns an existing Vue 3 app into a clean, SaaS-style interface: fixed left sidebar, tokenized colors and spacing, consistent cards and tables. It changes presentation and layout only. Routes, data flow, and component behavior stay as they are.

Supporting files in this folder:
- `reference/tokens.css`: the design tokens to paste into the global stylesheet.
- `reference/app-shell.md`: sidebar template, shell CSS, mobile drawer, shared class styles, and a table of knock-on fixes.

Not for: Vue 2 apps, non-Vue frontends, or requests that change behavior or data.

## Working rules

- Read the repo's own guidance first (CLAUDE.md, design notes). Where it conflicts with this skill, the repo wins. Typical rules to respect: color palette, no emojis in the UI, chart approach, i18n.
- If the repo requires a subagent for `.vue` edits (for example a `vue-expert` agent), delegate those edits to it, then run a code-review pass.
- Do not add a UI library, Tailwind, or an icon package unless the user asks. Match the app's existing icon approach (inline SVG with `currentColor` is the default).
- Do not mass-rewrite per-view scoped styles. Change what breaks or clashes with the new shell, and leave the rest.
- No emojis or text glyph icons in the UI.

## Phase 1: Audit (read-only)

Find and note, with file and line:
1. **The shell:** the root component that holds the top nav and `<router-view>` (usually `App.vue`).
2. **Navigation:** every `router-link`, how the active state is computed, and which labels are hard-coded rather than translated.
3. **Global styles:** the non-scoped stylesheet and the shared class names views depend on (`.card`, `.badge`, `.page-header`, tables, stat cards). These names must keep working.
4. **Hard-coded offsets:** anything tied to the old header height: `position: sticky; top: <n>px`, `padding-top`, `calc(100vh - <n>px)`.
5. **Floating UI:** dropdowns that open downward (profile menu, language switcher), and every modal's `position: fixed` and `z-index`.
6. **Existing constraints:** palette, fonts actually loaded, breakpoints (often none), i18n setup and locale files, icon approach.

Finish the audit with a short list of what will change and what will be left alone. Show it to the user before editing if the change set is large.

## Phase 2: Tokens

1. Copy `reference/tokens.css` into the app's global stylesheet. Adjust the accent and sidebar colors to fit the repo's palette; keep the spacing, radius, type, and z-index scales.
2. Point the shared classes at the tokens (`.card`, `.badge`, `.page-header`, buttons, tables). Keep every existing class name and variant.
3. If the font is Inter and nothing loads it, either load it or drop it from the stack. Do not leave a font that silently falls back.

## Phase 3: Shell

Replace the top nav with the sidebar using `reference/app-shell.md`:
- Brand at the top; nav list in the middle; language and profile controls in the footer.
- Drive the nav from an array of `{ to, labelKey, icon }` so every label goes through i18n.
- Active state comes from the router (`active-class` / `exact-active-class`), not manual path comparison. Make sure `/` does not match every route.
- The content column has one max-width and one padding. Remove duplicate `max-width` / `margin: 0 auto` containers elsewhere (filter bars often have their own).
- Page-level filters and toolbars render inside the content column, not above the whole app.

## Phase 4: Knock-on fixes

Work through this checklist. Each item is a bug the shell change usually introduces:
- [ ] Sticky elements that used the old header height now use `top: 0` or move into the page header.
- [ ] Dropdowns in the sidebar footer open upward or sideways and are not clipped by `overflow`.
- [ ] Modals and overlays use a z-index above the sidebar and the drawer scrim.
- [ ] Any hard-coded nav label now has a key in every locale file.
- [ ] The active link has `aria-current="page"` and a visible `:focus-visible` outline.
- [ ] Animations respect `prefers-reduced-motion` (the token file handles the shared transition).
- [ ] Views that defined their own page header, container width, or card style still look consistent next to the shared ones. Fix outliers by deleting the override, not by adding another.

## Phase 5: Responsive

Add the first breakpoint (about 900px): the sidebar becomes an off-canvas drawer, opened from a slim top bar, closed on link click and on scrim click. Many apps have no media queries at all, so expect to add this from scratch.

## Design rules

- Spacing comes from the 4px token scale only. No one-off pixel values for gaps and padding.
- One radius scale, one shadow scale. Cards use `--radius-lg` and `--shadow-sm`.
- Neutral slate surfaces plus a single accent. Status colors only for status.
- At most two font weights per view; a clear size step between page title, section title, and body.
- Consistent card padding and consistent gaps between cards across every view.
- Text contrast at WCAG AA or better, including sidebar text on the sidebar background.

## Verification

1. The production build passes.
2. Every route renders, with no console errors.
3. The sidebar highlights the right item on each route, including `/`.
4. Check at 1440, 1024, and 390 pixel widths: no horizontal scroll, drawer works at 390.
5. Open each dropdown and each modal; confirm none is clipped or hidden behind the sidebar.
6. Switch language (if the app has one) and confirm nav labels change and nothing overflows.
7. Take screenshots at the three widths and show them to the user.

For browser checks, use the browser tool the project and the user's environment permit. If the repo's instructions and the user's own policy disagree about which tool to use, stop and ask rather than choosing.
