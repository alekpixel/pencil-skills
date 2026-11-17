---
name: pencil-generate-design
description: "Workflow for building or updating a full screen in Pencil (pen.dev) from code or from a description, by reusing the file's design system (components, variables, themes, slots) instead of hardcoded colors and sizes. Use it with pencil-use whenever the user wants a page, screen, view, landing page, dashboard, settings page, modal, dialog, drawer, sidebar, panel or any multi-section layout drawn in a .pen file. Triggers: 'design this page in Pencil', 'put this screen on the canvas', 'turn this component/page/route into a Pencil design', 'mock up the dashboard in pen.dev', 'build a landing page in Pencil', 'update the Pencil screen to match the code', 'make the dark version of this screen', 'add the mobile layout', 'draw the empty and error states'. It inventories what the file already has, maps code tokens to Pencil variables, then builds the view section by section with a check after each one. For creating new components or tokens, use pencil-generate-library."
---

# pencil-generate-design: screens from code or a description

Build or update **composed views** in a `.pen` file: full pages, app screens, modals, drawers, sidebars, panels. The rule of this skill: **find what the file already has and use it.** Most files that matter have variables and components that mirror the codebase. A screen built from instances and `$variables` stays linked to the system; a screen of hex colors and loose frames does not.

**Load `pencil-use` first.** It holds the rules every `execute` call depends on (layout model, text sizing, ids, failure handling). This skill only adds the workflow.

## Scope

- Use this skill when the deliverable is a **screen or view** made of existing parts.
- To create or change **reusable components, variables or themes**, use `pencil-generate-library`. If the screen needs a component that does not exist, stop and agree with the user whether to build it properly there or draw it locally in this screen.
- To turn a design **into code**, read the app's own guide: `read_skill({path: "guide/code.md"})`.

## Prerequisites

- Pencil is open with the target file. `get_app_state` answers. If it does not, ask the user to open Pencil; do not retry in a loop.
- You know the source: code files, a route, a running URL, a screenshot, or a written brief.

## Workflow

Follow the steps in order. **No canvas write before step 3 is complete.**

### Step 1. Understand the deliverable

1. From code: read the page or view and the components it renders. Note each component's **default props**, not only the ones passed (a `<Button>` with no variant may default to `primary`).
2. List the sections from top to bottom (for a page: header, hero, content, footer; for a modal: title bar, body, actions).
3. For each section, list the UI parts: buttons, inputs, cards, tabs, avatars, badges, icons.
4. List the **states** the user cares about: default, loading, empty, error, signed out. Ask which to draw if unclear; each state is its own frame.
5. Note the **target sizes**: desktop 1440 (or the app's max width), tablet 768, mobile 390, or the real width of the modal or panel.
6. Note the **images**: logos, photos, avatars. Find the real files in the project (they will be applied as image fills by relative path).

### Step 2. Inventory the file

Run read-only calls first.

```js
Get(n => n.reusable && Print(n.id, "|", n.name))
Print(GetVariables())
Get(document, (n, c) => { if (c.depth === 1) { Print(n.id, "|", n.type, "|", n.name, "|", c.bounds.width, "x", c.bounds.height); c.skipChildren() } })
```

From the results, build three maps:

- **Components**: the part you need, the component id, its child ids for text, icons and slots. Read each needed component once with `Print(Get(componentId, {depth: 3}))`; note label ids, icon ids, slot ids and their recommended components.
- **Variables**: each code token and the Pencil variable that matches it. Match by name first (`--background` in CSS is often `--background` or `background` in Pencil), then by value. Note theme axes and their values (`mode: light, dark`).
- **Conventions**: the names, widths and spacing of existing screens. Match them.

If the file **imports a library**, its components appear in `get_app_state`; use them like local ones.

If the file has **no design system**:
- Say so to the user and propose `pencil-generate-library` first for anything beyond a quick sketch.
- For a quick sketch, define a minimal token set with `SetVariables` (background, surface, text, muted text, border, accent, a spacing scale, one or two font families) so the screen still uses `$variables`.
- With no brand direction at all, offer `get_style` archetypes as a starting point and let the user pick.

### Step 3. Gate: write the plan

Before any write, show the user a short plan:

```
View: Settings page, 1440 wide, light and dark
Sections: Header | Sidebar | Profile form | Danger zone
Components: Button/Primary (id), Button/Ghost (id), Input (id), Sidebar (id, slot nav), Avatar (id)
Missing: Toggle (no component; draw locally or build in library?)
Tokens: --background, --foreground, --muted-foreground, --border, --primary, space 16/24/32
Images: ./assets/avatar.jpg
```

Wait for the user when something is missing or ambiguous. Otherwise continue.

### Step 4. Create the screen frame

One call, alone. Find space, create the root frame, mark it as a placeholder, return its id in a global.

```js
const pos = FindEmptySpace({width: 1440, height: 1024, padding: 120})
screenId = Insert(document, {type: "frame", name: "Settings", x: pos.x, y: pos.y, width: 1440, height: "fit_content(1024)", layout: "vertical", fill: "$--background", clip: true, placeholder: true})
```

- Build every section **inside** this frame from the start. Do not build sections at the root and move them in later.
- Several screens: chain them with `FindEmptySpace({..., nodeId: previousScreenId})` so they line up.

### Step 5. Build one section per call

For each section, one `execute`:

1. Create the section frame inside the screen, with layout, gap and padding from variables.
2. Insert **instances** of components, with text and icons set through `descendants`.
3. Fill **slots** with `Insert(instanceId + "/" + slotId, {...})`; hide unused slots with `enabled: false`.
4. Draw locally only what no component covers: layout frames, row and column wrappers, one-off decoration.
5. Take one screenshot of the section, in the same call. A clipping check (`ctx.problems`) can be added, but it gives false alarms in current versions: the screenshot and the tool's `issues detected` block decide.
6. Hide parts you do not need (an icon, an empty slot) with `enabled: false` in `descendants`, keyed by the child's name when it is unique in the component (`"Icon"`, `"Label"`) or by its id.

```js
const t = {type: "text", fontFamily: "$--font-primary", fill: "$--foreground"}
headerId = Insert(screenId, {type: "frame", name: "Header", width: "fill_container", padding: [16, 32], alignItems: "center", justifyContent: "space_between", stroke: "$--border", strokeWidth: {bottom: 1}})
Insert(headerId, {...t, name: "Title", content: "Settings", fontSize: 20, fontWeight: "600"})
actionsId = Insert(headerId, {type: "frame", name: "Actions", gap: 12})
for (const [ref, label] of [[ghostBtnId, "Cancel"], [primaryBtnId, "Save changes"]]) Insert(actionsId, {type: "ref", ref, name: label, descendants: {[btnLabelId]: {content: label}}})
TakeScreenshot([headerId])
```

Rules while building:

- **Tokens, not values.** Colors, spacing, radii and fonts come from `$variables` whenever one exists. Hex and raw numbers only for values the system truly lacks, and list them in your report.
- **Instances, not copies of their insides.** Never redraw a button as a frame and a text when a Button component exists.
- **Real content.** Use the labels, counts and copy from the code or the brief, not "Lorem ipsum" or "Title". Check that no component placeholder text is left ("Button", "Label").
- **The right variant.** In Pencil, variants are usually separate components (`Button/Primary`, `Button/Outline`). Pick the one the code's default props imply.
- **Fix in place.** If a section looks wrong, correct it with `Update` or `Replace`. Do not delete and redo the screen.

When the last section is done: `Update(screenId, {placeholder: false})`.

### Step 6. From a running web app

When the source runs in a browser (local dev server or URL), the `browser` tool gives you the truth about spacing and images:

1. `browser({action: "load-page", url})`, then `browser({action: "return-screenshot"})` to look at it.
2. For a section you must match closely, `browser({action: "return-element", target: "query", querySelector: ".pricing"})` returns its DOM and computed styles. Query one section at a time; the whole body can be huge.
3. `import-to-canvas` reproduces a page or element as editable layers. Use it as a **reference layer** next to your screen, not as the deliverable: imported layers are not linked to the design system. Rebuild with components and variables, compare, then ask the user before deleting the reference.

### Step 7. Validate the whole view

1. Read the `issues detected` block of your last calls and fix real issues (collapsed frames).
2. Hardcoded color check: list fills that are hex strings instead of `$variables`. Run it on the whole document (`Get(n => ...)` without a root) so it also catches values inside components; a transparent `#00000000` on a ghost button is fine.
3. One screenshot per section, not only the full screen: a full-screen capture at low resolution hides clipped text and leftover labels.
4. Look for: clipped or overlapping text, placeholder copy, wrong variant, poor contrast, broken alignment, empty image frames.
5. Compare with the source (code, screenshot or brief) and fix differences with targeted calls.

### Step 8. Variants of the same screen

- **Dark mode**: copy the finished screen with `Copy(screenId, document, {name: "Settings / Dark", x, y, theme: {mode: "dark"}})` (position from `FindEmptySpace`). Only works if colors use themed variables; that is the test that the screen is token-clean. Instances inside the copy follow the theme too.
- **Export for review**: `Export([screenId, darkId], "png", "C:/project/design/exports", {scale: 1})` writes `<nodeId>.png` files the user can open without Pencil.
- **Breakpoints**: one frame per width (`Settings / Desktop`, `Settings / Mobile`), each rebuilt with the right structure (one column on mobile, sheets instead of side panels), not a scaled copy.
- **States**: one frame per state (`Settings / Empty`, `Settings / Error`), copied from the default and changed through the `Copy` call's `descendants`.

### Step 9. Updating an existing screen

1. Read its structure: `Get(screenId, (n, c) => c.depth <= 2 && Print(n.id, "|", n.type, "|", n.name))`.
2. Decide which sections change; leave the rest alone.
3. Change content with `Update`, swap a part with `Replace`, swap a variant by replacing the instance with a ref to the other component, add sections with `Insert` and `Move` to place them.
4. Re-check and screenshot only what you touched.
5. Never undo edits the user made since your last read.

## Report

End with: what was built (screen names and ids), which components and variables were used, any hardcoded values left and why, missing components, and what the user should review.
