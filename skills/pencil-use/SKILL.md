---
name: pencil-use
description: "Required first step before any Pencil (pen.dev) MCP call. Load this skill before calling `execute`, `get_app_state`, `get_style`, `read_skill` or `browser` from the `pencil` MCP server, or before touching any `.pen` file. It explains how the tools fit together, the rules of the `execute` JavaScript snippets (Insert, Copy, Update, Replace, Get, SetVariables, TakeScreenshot), the layout and text sizing model, variables and themes, components, instances and slots, and the mistakes that make calls fail or produce broken layouts. Triggers: 'in Pencil', 'pen.dev', 'on the canvas', 'open the .pen file', 'edit the design file', 'create a frame', 'add a component in Pencil', 'set variables in Pencil', 'inspect the .pen file', 'why did execute fail'. Load it together with pencil-generate-design for screens and pages, and with pencil-generate-library for tokens, components and themes."
---

# pencil-use: calling the Pencil MCP without errors

Pencil (pen.dev) is a vector design tool whose documents are `.pen` files. Claude Code talks to it through the `pencil` MCP server, which drives the running desktop app (or the IDE extension). This skill is the rulebook for every call. Two companion skills build on it:

- `pencil-generate-design`: build or update a screen, page, modal or panel from code or a description.
- `pencil-generate-library`: build variables, themes and components in the right order.

This is an unofficial community skill. It is not made or endorsed by pen.dev.

## 1. Before the first call

1. **Pencil must be running.** The MCP server connects to the open app. The error `failed to connect to running Pencil app` means the app is closed or the file is not open. Tell the user to open Pencil and the target file. Do not retry in a loop, and do not launch the app yourself unless the user asks.
2. **Never read a `.pen` file with Read, Grep, cat or any file tool.** The files are encrypted. Only the MCP tools can read or write them.
3. **Load deferred tools in one round trip.** If the tools are deferred, load them together:
   `ToolSearch select:mcp__pencil__get_app_state,mcp__pencil__read_skill,mcp__pencil__get_style,mcp__pencil__execute,mcp__pencil__browser`
4. **Call `get_app_state` first.** It reports the open documents and their paths, the current selection, the reusable components, browser nodes, and whether the user is signed in (needed for `Generate`). Copy the file path it returns, exactly as written, into `filePath` for every later call. On Windows it looks like `/C:/Users/name/project/design.pen` (leading slash, forward slashes). A new untitled document lives in Pencil's own folder (`~/.pencil/documents/...`); the MCP cannot "save as", so ask the user to save it from the app where they want it.
5. **Read the built-in docs once per session.** `read_skill()` returns the app's own guide for the installed version. Read at least `read_skill({path: "execute.md"})` and `read_skill({path: "pen-schema.md"})` before your first write. If they contradict this skill, the app's docs win: they match the version the user runs.

## 2. The five tools

| Tool | Use it for |
| --- | --- |
| `get_app_state` | Start of every task, and again when the user may have changed something. Lists files, selection, components, sign-in. |
| `read_skill` | The app's own reference: `SKILL.md`, `execute.md`, `pen-schema.md`, `guide/components.md`, `guide/design-system.md`, `guide/code.md`, `guide/tailwind.md` and others. |
| `get_style` | Ready-made style archetypes (fonts, colors, imagery) when the user has no brand direction. They give reference values only: they do not create variables. Call without arguments to list them, then with `name`, then with `params` if it asks for some. |
| `execute` | Runs a short JavaScript snippet against the document. This is where all reading and writing happens. `filePath` is required. |
| `browser` | Drives the built-in browser: `load-page`, `return-screenshot`, `return-element`, `import-to-canvas`, `screenshot-to-canvas`, `cdp`. Useful to look at a running web app or to bring a live page onto the canvas as editable layers. |

## 3. How `execute` behaves

- **Atomic.** If the snippet throws, every change and every global it created is rolled back. The document is as it was.
- **Fix failures with `edits`, never by resending.** A failed call returns an `editId`. Send `editId` plus `edits` (a list of `find`/`replace` pairs applied to the failed snippet) and no `input`. If the patched snippet fails again, keep patching under the same `editId`; `find` must match the already patched text.
- **Read the warnings.** A successful call can still end with an `issues detected` block (for example "uses fit_content but has no children, so its size collapses to zero"). Fix them in the next call before moving on. A warning about a child you hid with `enabled: false` in an instance can be spurious: confirm with a screenshot.
- **Validation errors are precise.** A bad property fails the whole call with a message such as `/alignItems expected one of: "start", "center", "end", got "stretch"`. Patch exactly that value with `edits`.
- **Scope.** Each call has its own scope. `const` and `let` die with the call. To keep a value for the next call, assign a bare global: `heroId = Insert(...)`. Keep only ids and small values in globals, never large `Get` results.
- **Output.** The response contains what you `Print`, any screenshots you asked for, and a name-to-id map of the nodes created. Name every node so the map is readable.
- **Keep snippets short.** No comments in the snippet. Use arrays, loops, spreads and small helper functions for repeated UI. A good call does one section of work.
- **One call at a time per file.** Do not send several `execute` calls to the same file in parallel: each one changes the document the next one reads.
- **The document is shared.** The user (or another agent) can edit while you work. If a node is missing or different, read it again instead of recreating it, and do not undo the user's changes.

### The API inside a snippet

```ts
document                                   // id of the root
Insert(parentId, node): id                 // one node, appended at the end of the parent
Copy(idOrPath, parentId, overrides?): id   // descendants get new ids
Update(idOrPath, props): void              // cannot change id, type, ref or children
Replace(idOrPath, node): id                // swaps a node (or an instance part) for a new tree
Move(idOrPath, parentId?, index?): void
Delete(idOrPath): void
Get(idOrPath, {depth, resolveVariables, resolveInstances, includePathGeometry})
Get(idOrPath?, (node, ctx) => value)       // visitor, top down; undefined collects nothing
SetVariables(defs, replace = false)
GetVariables(): {variables, themes}
FindEmptySpace({width, height, direction, padding, nodeId}): {x, y, parentId?}
Print(...values)
TakeScreenshot([ids])                      // "document" for the whole canvas
Export([ids], "png" | "jpeg" | "webp" | "pdf" | "html-tailwind" | "html-css", outputPath, options?)
Generate(type, ...)                        // images and SVG artwork, see section 9
```

Paths: an id, or `instanceId/childId` to reach inside a component instance (any depth). Slashes are only for instance nesting, never for ordinary parent and child frames. Always pass id strings, never node objects: use `node.id`.

## 4. Writing nodes

- **Never set `id`.** Pencil generates ids and ignores yours. Capture the returned id.
- **Always set `name`** on every node, in plain words ("Primary Button", "Price Label").
- **`Insert` adds one node.** Add children with further `Insert` calls on the returned id. `Replace` can take a full tree with `children`.
- **To read what you just created**, use `Get(id, {depth: 1}).children`; it reflects the current call.
- **Style objects you spread must include `type`.** `const label = {type: "text", fontFamily: "Inter", fontSize: 14, fill: "$text"}` then `Insert(p, {...label, name: "A", content: "A"})`.
- **Leave defaults out.** Only write a property when it differs from the default (or overrides a component value).
- **Only `frame` and `group` have children.** Text, shapes, icons and instances cannot.
- **Properties do not cascade.** A text inside a dark frame does not inherit a light color. Set every visual property on the node itself, or through a variable.
- **Root stays clean.** Directly under `document`, put only screens, component frames and large containers. Never loose text, buttons or icons. Place components above, screens below, growing right and down.
- **Place root nodes with `FindEmptySpace`**, not guessed coordinates. Chain screens with its `nodeId` option. Never overlap root nodes.
- **Placeholders.** Set `placeholder: true` on a root frame while you work on it, and set it back to `false` as soon as that frame is done. Do not leave it on.
- **Fix, do not rebuild.** When a result looks wrong, update the existing nodes. Do not delete a screen to start again unless the user asks.

## 5. Layout model (it is not CSS)

Pencil uses its own flex-like model. Do not think in HTML or CSS, and if a property is not in the schema it does not exist.

- `layout` is `"none"`, `"horizontal"` or `"vertical"`. **Frames default to `horizontal`** and to `fit_content` width and height.
- `layout`, `padding`, `gap`, `justifyContent`, `alignItems` exist **only on frames**. Never put padding on text: wrap it in a frame.
- `justifyContent`: `start`, `center`, `end`, `space_between`, `space_around`. `alignItems`: `start`, `center`, `end` only. **`stretch` and `baseline` are errors.**
- `padding`: one value, `[vertical, horizontal]`, or `[top, right, bottom, left]`.
- Sizes: a number, a `$variable`, `"fit_content"`, `"fill_container"`, or either with a fallback such as `"fit_content(800)"`. **No percentages, no `vh`, no `calc`, no margin.**
- `x` and `y` are **ignored** for a child inside a layout. Use them only when the parent has `layout: "none"` or the child has `layoutPosition: "absolute"`.
- **Circular sizing collapses to zero.** A `fit_content` parent whose children are all `fill_container` has no size. Give the parent a real width (or a fallback) before children fill it.
- There is no wrapping. Build grids as a vertical frame of horizontal row frames.
- To share width evenly, set each child to `fill_container` and set `gap` on the parent.
- Screen frames: set a real width (1440, 1280, 390...) and `clip: true`. Nothing scrolls on the canvas, so make containers big enough for their content.

## 6. Text

- **Text is invisible until you set `fill`.** Always set it (emoji too).
- `textGrowth` decides sizing:
  - `auto` (default): one line, size from content; any width or height is ignored.
  - `fixed-width`: you set `width` (often `"fill_container"`), height follows the content, lines wrap.
  - `fixed-width-height`: you set both; text can overflow. Use rarely.
- **Never set `width` on text without `textGrowth`.** Never guess text dimensions: let wrapping and layout size it.
- Headings and paragraphs inside a layout: `textGrowth: "fixed-width", width: "fill_container"`. Labels, tags, buttons: leave `auto`.
- `textAlign` only shows with a fixed width; `textAlignVertical` only with `fixed-width-height`. They align text inside its box, not the box itself: move the box with the parent's layout.
- `lineHeight` is a ratio of the font size (`1.4`), not pixels. `fontWeight` is a string (`"600"`).
- All Google Fonts are available by family name. Custom fonts must be installed by the user.

## 7. Variables and themes

- Read before writing: `Print(GetVariables())`.
- `SetVariables` takes objects with a `type` (`color`, `number`, `string` or `boolean`) and a `value`. A bare value such as `"#1A1A1A"` fails. A boolean variable can drive visibility: `enabled: "$show-badge"`.
- Themed values work for every type, not only colors: a number variable with `{value: 6, theme: {size: "sm"}}` and `{value: 14, theme: {size: "lg"}}` lets one component change its padding when an instance sets `theme: {size: "sm"}`.
- Variable names never start with `$`. You add `$` only when you reference one: `fill: "$surface"`, `gap: "$space-4"`, `fontFamily: "$font-body"`.
- `SetVariables(defs)` merges. `SetVariables(defs, true)` **replaces every variable in the file**: never use it on a file you did not create, unless the user asks.
- Themed values are arrays of `{value, theme}`: `{type: "color", value: [{value: "#FFFFFF", theme: {mode: "light"}}, {value: "#111111", theme: {mode: "dark"}}]}`. New axes and values register themselves. The first value of an axis is the default.
- A variable value can reference another variable (`value: "$gray-900"`), which gives you primitive and semantic layers.
- Apply a theme to a subtree with the node's `theme` property: `Update(screenId, {theme: {mode: "dark"}})`. Children inherit it unless they set their own value for the same axis.
- Colors are hex: `#RGB`, `#RRGGBB` or `#RRGGBBAA`. Solid fill opacity goes through the alpha channel.

## 8. Components, instances, slots

- A component is any node with `reusable: true`. An instance is `{type: "ref", ref: componentId}`.
- **Overrides on the instance root** go directly on the ref: `{type: "ref", ref: btnId, width: "fill_container"}`.
- **Overrides inside** go in `descendants`, keyed by child id, `nested/path`, or a unique child name: `descendants: {[labelId]: {content: "Save"}, [iconId]: {icon: "check"}}`. Use flat path keys for deep children; do not nest `descendants` objects.
- A `descendants` entry **with `type`** replaces that child with a new tree. Without `type`, it only changes properties.
- After insertion, change a part with `Update(instId + "/" + childId, {...})` or swap it with `Replace(instId + "/" + childId, {...})`.
- **You cannot delete a child of an instance.** Hide it: `{enabled: false}`.
- **An instance has no `children` of its own.** Do not `Get` it to find its parts. Use the component's child ids, or `Get(instId, {resolveInstances: true})` if you must.
- **`Copy` of a component makes an instance**, not a new component. `Copy` of a normal node gives every descendant a new id, so put descendant changes in the `Copy` call's own `descendants` map, never in a later `Update` on old ids.
- Instances outside a layout need both `x` and `y` overrides, even `0`.
- **Slots**: a frame inside a component with `slot: [recommendedComponentIds]`. Fill it on an instance with `Insert(instId + "/" + slotId, {type: "ref", ref: itemId, ...})`. Hide an unused slot with `enabled: false`. **An empty slot collapses to zero height** (a frame with no children sized by `fit_content`): give the slot frame a height in the component, for example `height: 40`.
- **Icons**: `{type: "icon", library: "lucide", icon: "search", width: 20, height: 20, fill: "$text"}`. Libraries: `lucide`, `feather`, `Material Symbols Outlined`, `Material Symbols Rounded`, `Material Symbols Sharp`, `phosphor`. Swap an icon inside an instance with a `descendants` override on `icon`.
- Components cannot be referenced across files, except through an imported design library (`.lib.pen`). Otherwise copy them into the file.

## 9. Images and artwork

- There is no `image` node. An image is a fill on a frame or shape: `fill: {type: "image", url: "./images/photo.jpg", mode: "cover"}`, with the path relative to the `.pen` file.
- Prefer real assets from the project (logos, photos, SVG exported by a designer). Do not draw logos, mascots or illustrations out of paths: they look bad.
- `Generate` (AI images, stock images, SVG artwork, background removal) needs the user signed in to pen.dev and runs after the call returns. Ask the user before using AI generation; many teams do not want generated images. If it is not wanted or not available, leave a clearly named placeholder frame and say so.
- For an SVG path, always set `viewBox` and explicit `width` and `height`.

## 10. Reading and checking

- Read one node: `Print(Get(id, {depth: 2}))`. List components: `Get(n => n.reusable && Print(n.id, n.name))`.
- There is no whole-document dump without a visitor. Print one compact row per node instead of JSON objects.
- **Clipping check** after each section: `Get(sectionId, (n, c) => c.problems && Print(n.name, "|", c.parentCtx?.node.name, "|", c.problems))`. **Treat it as a hint, not a verdict.** In tests (November 2026), `ctx.bounds` came back shifted (about 50 px down for children of layout frames, wrong x with `space_between`), so `ctx.problems` reported "fully clipped" on nodes that render fine. Confirm any reported problem with `TakeScreenshot` before changing the design, and rely on the `issues detected` block for collapsed sizes.
- **Hardcoded color check**: `Get(screenId, n => typeof n.fill === "string" && n.fill.startsWith("#") && Print(n.id, n.name, n.fill))`.
- `TakeScreenshot([sectionId])` at the end of the call that finishes a section. Screenshots cost tokens: one per finished section, on the smallest meaningful node. Use visitors with `ctx.bounds` for size checks.
- Look at each screenshot for: collapsed layout, clipped text, overlaps, poor contrast, leftover placeholder text, misaligned edges.
- `Export` is for delivering files (PNG, PDF, HTML), not for checking your work. It accepts an absolute output folder (`"C:/project/exports"`) and names each image `<nodeId>.png`.

## 11. Pre-flight checklist

Before each `execute`:

- [ ] `filePath` is the path `get_app_state` returned.
- [ ] No `id` set on new nodes; every node has a `name`.
- [ ] No comments; repeated UI uses loops or helpers.
- [ ] Values needed later are bare globals, not `const`.
- [ ] Every text has `fill`; every wrapped text has `textGrowth` plus a width.
- [ ] No `%`, `vh`, `calc`, `margin`, `alignItems: "stretch"` or `"baseline"`; padding and layout only on frames.
- [ ] No `x`/`y` on children of a layout frame; no circular `fit_content` and `fill_container`.
- [ ] Colors and spacing come from `$variables` when the file has them.
- [ ] Instance parts changed through `descendants`, `Update(inst + "/" + child)` or `Replace`, never `Delete`.
- [ ] Root frames use `FindEmptySpace`, `placeholder: true` while working, `clip: true` for screens.
- [ ] `SetVariables(..., true)` only on a file you own and with the user's consent.

## 12. Common errors

| Symptom | Cause | Fix |
| --- | --- | --- |
| `failed to connect to running Pencil app` | App closed or file not open | Ask the user to open Pencil and the file; do not loop |
| Text does not show | No `fill` on the text | Add `fill` |
| Element has zero width | Circular `fit_content` and `fill_container` | Give the parent a width or a fallback |
| `x`/`y` change nothing | Parent has a layout | Use layout, or `layoutPosition: "absolute"` |
| Update on a copied child fails | Copy gave new ids | Use the `descendants` map in the `Copy` call |
| Cannot delete a part of an instance | Not allowed | `enabled: false` on that part |
| Variable fails to set | Bare value or name starting with `$` | `{type, value}`, name without `$` |
| Values vanish after the call | Declared with `const` or `let` | Assign a bare global |
| `Generate` fails | User not signed in | Placeholder, tell the user |
| Width `"100%"` rejected | Percentages unsupported | `fill_container` or a number |
| Empty frame or slot invisible | `fit_content` with no children | Give it a height, or a fallback `fit_content(40)` |
| `ctx.problems` says "clipped" but it looks fine | Shifted `ctx.bounds` | Trust the screenshot |
