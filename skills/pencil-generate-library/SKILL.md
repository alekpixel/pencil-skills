---
name: pencil-generate-library
description: "Workflow for building or updating a real design system in Pencil (pen.dev), usually from a codebase: variables (colors, spacing, radius, type), light and dark themes, reusable components, their variants and states, slots, a foundations board, and an optional .lib.pen design library. Use it with pencil-use whenever the user wants tokens or components in a .pen file, even a single component, because a component needs token foundations and variants to be usable. Triggers: 'create a design system in Pencil', 'import my tokens into pen.dev', 'sync the CSS variables / Tailwind theme to Pencil', 'add dark mode to the .pen file', 'build a button component in Pencil', 'create the component library', 'make variants for this component', 'turn this file into a library', 'reconcile Pencil with the code'. It says what to build and in what order (discovery, tokens, file layout, components one by one, QA), with a user checkpoint between phases. pencil-use says how to call the tools."
---

# pencil-generate-library: tokens, themes and components in order

Build a design system in a `.pen` file that matches the code. **Load `pencil-use` first**: it holds the `execute` rules. This skill holds the order of work and the design system decisions.

## 1. The rule that matters most

**This is never one call.** A small system takes 15 to 40 `execute` calls, a larger one many more. Work in the smallest useful unit, check it, show it, then go on. A system built in one long snippet comes out broken and is hard to repair.

## 2. How Pencil differs from other tools

Plan with Pencil's real model, not one borrowed from elsewhere:

| Need | In Pencil |
| --- | --- |
| Tokens | Document `variables` of type `color`, `number`, `string`, `boolean`, referenced as `$name` |
| Modes (light, dark) | Theme axes. A variable value is a list of `{value, theme: {mode: "dark"}}`; a frame picks a value with its `theme` property |
| Alias (semantic to primitive) | A variable whose value is another variable: `value: "$gray-900"` |
| Text styles | None. Use font variables (family, size, line height, weight as strings or numbers) and, if useful, small text components |
| Shadow styles | None. Shadow effects accept variables for color, blur and offset |
| Component | Any node with `reusable: true` |
| Variants | No variant sets. One component per variant, named with slashes: `Button/Primary`, `Button/Outline` |
| Boolean option (show icon) | Instance override `enabled: false` on that part |
| Swappable icon | Instance override on the icon node: `{icon: "plus"}` |
| Swappable content | A `slot` frame inside the component, with recommended component ids |
| Shared library | A file converted to `.lib.pen` in the app (one way, cannot be undone), imported by screen files |

## 3. Workflow

```
Phase 0  DISCOVERY        read code and file, no writes          -> checkpoint: plan approved
Phase 1  FOUNDATIONS      variables and themes                   -> checkpoint: token summary approved
Phase 2  FILE LAYOUT      areas on the canvas, foundations board -> checkpoint: board screenshot approved
Phase 3  COMPONENTS       one at a time, atoms before molecules  -> checkpoint after each component
Phase 4  QA AND LIBRARY   audits, fixes, optional .lib.pen       -> checkpoint: sign-off
```

Name the next phase at each checkpoint. A "looks good" about the tokens is not approval to start components.

### Phase 0. Discovery (read only)

**0a. Find the tokens in the code**, in this order:
1. CSS custom properties (`:root { --background: ... }`, `.dark { ... }` or `[data-theme="dark"]`).
2. Tailwind config or Tailwind v4 `@theme` blocks.
3. Token files (DTCG `.tokens.json`, Style Dictionary).
4. Theme objects in CSS-in-JS, or native theme files (iOS asset catalogs, Android `colors.xml`).

Note the dark mode mechanism, the spacing and radius scales, fonts and the type scale, and shadows.

**0b. Find the components in the code**: the list of UI primitives (Button, Input, Checkbox, Badge, Avatar, Card...), their props, which props are variants (`variant`, `size`, `state`), which are booleans, which take children.

**0c. Read the file**:

```js
Print(GetVariables())
Get(n => n.reusable && Print(n.id, "|", n.name))
Get(document, (n, c) => { if (c.depth === 1) { Print(n.id, "|", n.type, "|", n.name); c.skipChildren() } })
```

Also check `get_app_state` for imported libraries. Match any naming already in the file.

**0d. Lock the scope** with the user: the exact token list and the exact component list for this round. A first round of six to ten components is a good size.

**0e. Map code to Pencil and settle conflicts.** Present a table:

```
Code token            Pencil variable       Light      Dark       Status
--background          --background          #FFFFFF    #0A0A0A    new
--primary             --primary             #1D4ED8    #3B82F6    exists, value differs (ask)
Component             Pencil components                           Status
Button variant x size Button/Primary, Button/Secondary, Button/Outline, Button/Ghost (size via theme axis)   new
```

When code and file disagree, ask which wins. Do not decide alone.

**Checkpoint**: show the plan, wait for approval.

### Phase 1. Foundations

**Naming.** Keep the code's names so design and code round-trip: a CSS variable `--muted-foreground` becomes the Pencil variable `--muted-foreground`, referenced as `$--muted-foreground`. Never start a name with `$`. Avoid `:` in names. If the code has no names, use short kebab names by role: `surface`, `text`, `text-muted`, `border`, `accent`, `space-4`, `radius-md`.

**Layers.**
- **Primitives**: raw values, no theme (`gray-50` ... `gray-950`, `blue-500`, `space-1` ... `space-16`).
- **Semantics**: role names, themed, pointing at primitives (`--background`: light `$white`, dark `$gray-950`).
- Components use **semantics only**. A component that references `$gray-900` directly will not switch with the theme.

**Size of the system.**
- Under about 50 tokens: one flat set, one `mode` axis with `light` and `dark`.
- Larger: primitives plus semantics as above, and extra theme axes only when the code really has them (`density: comfortable, compact`; `brand: a, b`).

**Write in small batches**, one family per call (primitive colors, then semantic colors, then spacing and radius, then type), and read back:

```js
SetVariables({
  "gray-50": {type: "color", value: "#FAFAFA"},
  "gray-900": {type: "color", value: "#171717"},
  "gray-950": {type: "color", value: "#0A0A0A"},
  "white": {type: "color", value: "#FFFFFF"}
})
SetVariables({
  "--background": {type: "color", value: [{value: "$white", theme: {mode: "light"}}, {value: "$gray-950", theme: {mode: "dark"}}]},
  "--foreground": {type: "color", value: [{value: "$gray-900", theme: {mode: "light"}}, {value: "$gray-50", theme: {mode: "dark"}}]}
})
const v = GetVariables()
Print(Object.keys(v.variables).length, "variables", "|", JSON.stringify(v.themes))
```

- Check that the first value of the `mode` axis is the default the product uses (usually `light`).
- Give every themed variable a value for **every** theme value you use. A missing value shows the wrong color silently.
- Keep values identical across themes when they should not change (spacing, radius).
- Typography: `--font-sans` (string), sizes (`text-sm` 14, `text-base` 16...), line heights as ratios (`leading-normal` 1.5), weights as strings (`"400"`, `"600"`).
- Shadows: a themed color variable for the shadow and number variables for blur and offset if the code has a shadow scale.
- **Never** `SetVariables(..., true)` on an existing file without explicit consent: it wipes every variable.

Exit criteria: every token in the plan exists with the right type, every themed token has every theme value, aliases point at existing variables.

**Checkpoint**: show counts per family, theme axes, and any value you had to guess.

### Phase 2. File layout and foundations board

Keep the canvas readable:

- A **Foundations** frame at the top left: color swatches, type specimens, spacing bars, radius samples.
- A **Components** area to its right or below, one frame per component family (`Buttons`, `Inputs`, `Cards`), holding the components with a title and a short usage note.
- **Screens** further down and to the right, never mixed with components.
- Use `FindEmptySpace` for every root frame.

Swatches and specimens must be **bound to variables**, so the board updates when a token changes. Build swatches with a loop:

```js
boardId = Insert(document, {type: "frame", name: "Foundations / Color", layout: "vertical", gap: 16, padding: 32, fill: "$--background", x: 0, y: 0})
rowId = Insert(boardId, {type: "frame", name: "Semantic colors", gap: 16})
for (const name of ["--background", "--foreground", "--primary", "--muted", "--border"]) {
  const s = Insert(rowId, {type: "frame", name: "Swatch " + name, layout: "vertical", gap: 8})
  Insert(s, {type: "rectangle", name: "Chip", width: 96, height: 64, cornerRadius: 8, fill: "$" + name, stroke: "$--border", strokeWidth: 1})
  Insert(s, {type: "text", name: "Label", content: name, fontSize: 12, fill: "$--foreground"})
}
TakeScreenshot([boardId])
```

Then copy the board with `theme: {mode: "dark"}` next to it: if anything stays light, a token is missing its dark value.

**Checkpoint**: screenshot of the boards in both themes.

### Phase 3. Components, one at a time

Order by dependency: **atoms first** (Icon wrappers, Badge, Avatar, Button, Input, Checkbox, Toggle), then **molecules** (Input group with label and hint, Menu item, Tab), then **organisms** (Card, Dialog, Sidebar, Table row). A component that contains another must be built after it, and must contain an **instance** of it, not a redrawn copy.

For each component:

**3a. Design the API before drawing.** From the code props:
- Variant axes that change color or structure become **separate components**: `Button/Primary`, `Button/Secondary`, `Button/Outline`, `Button/Ghost`, `Button/Destructive`.
- Axes that only change token values (size, density) can be a **theme axis** instead of more components: themed number variables such as `button-padding-y` and `button-padding-x` with `size: md, sm, lg` (first value is the default). One component then serves every size through `theme: {size: "sm"}` on the instance. Tested and working. Use it when the code works the same way; otherwise make size components too.
- A global switch (show or hide every badge in mockups) can be a **boolean variable** bound to `enabled: "$show-badge"`. It changes every instance at once, so keep it for demo toggles, not for per-instance options.
- Keep the count sane. If variants times sizes times states goes past about 30 components, split the family or move an axis to a theme.
- States: a `Disabled` or `Focus` component only when the user wants to show them; hover and pressed are often only documented.
- Booleans (with icon, with hint) are **not** variants: build the full component and hide parts per instance with `enabled: false`.
- Icons are **not** variants: one icon node, swapped with an `icon` override.
- Free content areas are **slots**: a frame with `slot: [recommendedComponentIds]`, `width: "fill_container"` and a **fixed height** (for example `height: 40`). An empty slot sized by `fit_content` collapses to zero and disappears.

**3b. Build the base component** in one call: a frame with `reusable: true`, layout, padding, gap, radius and fills **all from variables**, named children (`Label`, `Icon`, `Content`). Capture its id and its child ids in globals.

```js
const pos = FindEmptySpace({width: 600, height: 200, direction: "right", padding: 80})
buttonsId = Insert(document, {type: "frame", name: "Buttons", layout: "vertical", gap: 24, padding: 32, x: pos.x, y: pos.y})
btnPrimaryId = Insert(buttonsId, {type: "frame", name: "Button/Primary", reusable: true, gap: 8, padding: ["$button-padding-y", "$button-padding-x"], cornerRadius: "$radius-md", fill: "$--primary", alignItems: "center", justifyContent: "center"})
btnIconId = Insert(btnPrimaryId, {type: "icon", name: "Icon", library: "lucide", icon: "plus", width: 16, height: 16, fill: "$--primary-foreground"})
btnLabelId = Insert(btnPrimaryId, {type: "text", name: "Label", content: "Button", fontFamily: "$--font-sans", fontSize: "$text-sm", fontWeight: "500", fill: "$--primary-foreground"})
TakeScreenshot([btnPrimaryId])
```

**3c. Build the other variants.** `Copy` of a component makes an **instance**, not a new component. To get independent variant components, rebuild each one with a small helper function inside one call, looping over a variant table:

```js
const make = (name, fill, fg, stroke) => {
  const id = Insert(buttonsId, {type: "frame", name, reusable: true, gap: 8, padding: ["$button-padding-y", "$button-padding-x"], cornerRadius: "$radius-md", fill, alignItems: "center", justifyContent: "center", ...(stroke ? {stroke, strokeWidth: 1} : {})})
  Insert(id, {type: "icon", name: "Icon", library: "lucide", icon: "plus", width: 16, height: 16, fill: fg})
  Insert(id, {type: "text", name: "Label", content: "Button", fontFamily: "$--font-sans", fontSize: "$text-sm", fontWeight: "500", fill: fg})
  return id
}
btnSecondaryId = make("Button/Secondary", "$--secondary", "$--secondary-foreground")
btnOutlineId = make("Button/Outline", "$--background", "$--foreground", "$--border")
btnGhostId = make("Button/Ghost", "#00000000", "$--foreground")
```

Keep child names identical across variants (`Label`, `Icon`) so instances can be overridden by name and a variant swap keeps working.

**3d. Lay out the family** in a readable grid inside its family frame: one row per variant, one column per size or state, with small text headers. Use row frames with `gap`, never absolute positions.

**3e. Document it**: a title and a two or three line usage note in the family frame (when to use which variant, the one primary action rule).

**3f. Check it**:
- Hardcoded check on the family frame: no hex fills except deliberate transparent ones (`#00000000` works as a transparent fill for ghost variants).
- The `issues detected` block of the call (collapsed frames, empty slots). The `ctx.problems` clipping check gives false alarms in current versions: confirm with a screenshot.
- Instance test: insert one instance of each variant in a scratch frame, override the label with a long text and hide the icon, then screenshot it in light and dark (`theme` on the scratch frame). Delete the scratch frame afterwards.

**Checkpoint per component**: screenshot, variant list, and the ids. Do not start the next component before approval.

### Phase 4. QA and library

1. **Token audit**: every variable used by a component exists; every themed variable has every theme value.
2. **Binding audit**: no hex colors or raw spacing inside components, except values listed as deliberate.
3. **Contrast**: text against its background in both themes (4.5:1 for body text, 3:1 for large text and UI borders). Read resolved values with `Get(id, {resolveVariables: true})`.
4. **Naming audit**: every node named, no duplicates among components, consistent casing (`Family/Variant`).
5. **Leftovers**: no `placeholder: true`, no scratch frames, no default labels on the boards.
6. **Screenshots** of every family frame in both themes.
7. **Library (optional)**: to share the system with other files, the user converts the file with **Libraries > Turn this file into a library** in the app. It becomes `.lib.pen`, the change cannot be undone, and the file must not import other libraries. Recommend keeping screens in separate files that import it. Ask before suggesting the conversion; the user does it in the app.

**Checkpoint**: audit report and sign-off.

## 4. Keeping state across a long build

Conversation context gets compacted on long builds. Keep a small ledger **as a JSON file next to the project** (not inside the `.pen` file), for example `design/pencil-ledger.json`:

```json
{
  "file": "design/app.pen",
  "phase": "phase3",
  "done": ["phase0", "phase1", "phase2", "Button", "Badge"],
  "variables": 64,
  "themes": {"mode": ["light", "dark"]},
  "components": {"Button/Primary": "id", "Button/Outline": "id", "Badge/Default": "id"},
  "children": {"Button/*": {"Label": "id", "Icon": "id"}},
  "next": "Input"
}
```

Update it after each checkpoint. To resume, read the ledger, then rebuild the id map from the file itself (`Get(n => n.reusable && Print(n.id, "|", n.name))` and `GetVariables()`), because ids in the ledger can be stale if the user edited the file. **Never guess an id from memory.**

## 5. Anti-patterns

- Starting components before the tokens are agreed and written.
- Components that reference primitives or hex values instead of semantic variables.
- One component per icon, or one per boolean option.
- Variant names that differ in child names, which breaks overrides.
- A variant matrix that grows past about 30 components in one family.
- Redrawing a nested part instead of using an instance of its component.
- `SetVariables(..., true)` on someone else's file.
- Deleting by name prefix to clean up: delete only ids you created and recorded.
- Skipping a checkpoint to save time, or building on work the user rejected.
- Several `execute` calls sent in parallel to the same file.
