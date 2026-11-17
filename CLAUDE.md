# Parapet – window-sill furniture designer

A single-page 3D viewer (`index.html`, three.js) for designing a DIY built-in unit that covers a
window sill, adds storage and hides a radiator. Rooms and furniture designs are both data, inline
in `index.html`: rooms in the `<script type="application/json" id="pokoje">` block, designs
(versions) in the `<script type="application/json" id="wersje">` block; every version belongs to
one room. **Adding a room or a design means adding an entry to one of those blocks – not touching
the viewer code.**

The owner speaks Polish. All UI text, element names and assembly notes are in Polish. Reply in
the language the owner writes in.

## Run and view

```bash
cd /home/kamil/Desktop/claude-parapet
python3 -m http.server 48765 --bind 127.0.0.1     # port 8765 is taken by something else
```

Open `http://127.0.0.1:48765/index.html`. three.js 0.160 loads from the jsDelivr CDN, so internet
is needed. The browser caches aggressively: reload with Ctrl+Shift+R, or add `?v=N` to the URL.

The **Pokój** section of the left panel has one button per room, and the **Wersja** section below
it shows only the versions of the selected room. Switching a room rebuilds the room, the room
dimensions and the camera; switching a version rebuilds the furniture. Both happen in place. The
page opens on the room last selected (`localStorage` key `parapet.pokoj`) and, within a room, on
the version last selected there (`parapet.wybrana.<room name>`), or on the first one. The page is
read-only: rooms and versions are added, renamed or removed only by editing the blocks in
`index.html` when the owner asks for it.

## Files

| Path | What it is |
|---|---|
| `index.html` | Viewer: room renderer, panel renderer, dimensions, parts list, export, room and version switch. The `pokoje` block holds the rooms, the `wersje` block all designs; `v1` (room `Sypialnia`) is the reference design (17 parts, passes the overlap check) |
| `pokoj.jpeg`, `rzut-rur.jpeg` | Owner's photo of the room `Sypialnia` and top-down sketch of its pipes |
| `.playwright-mcp/` | Scratch screenshots from testing; safe to ignore |

## Coordinate system

Units are **centimetres** in the room data, the version data and the code. The parts list and exports show millimetres.
Each room has its own coordinates (values below are for `Sypialnia`):

- **x** – from the left wall to the right: `0 … 210` (room `size[0]`)
- **y** – from the floor up: `0 … 250` (room `size[1]`)
- **z** – from the face of the wall under the window, into the room. The window is recessed, so
  its plane is at `z = -2` (`-window.recess`).

A panel is an axis-aligned box: `pos` is its corner with the smallest x, y and z (left, bottom,
nearest the window wall); `size` is `[x width, y height, z depth]`.

## Room format

The `pokoje` block is a JSON array; its order is the button order. A room is shown only if it
passes `validRoom()`; otherwise it is skipped with a console error. Never change a room's
measurements unless the owner gives new ones. Fields (cm):

| Field | Required | Meaning |
|---|---|---|
| `name` | yes | Unique, non-empty; shown on the button and referenced by versions' `room` |
| `size` | yes | `[width x, height y]` of the room |
| `depth` | yes | How deep into the room the floor, ceiling and side walls are drawn |
| `wallT` | yes | Thickness of the window wall (drawn only) |
| `window` | yes | `{x, w, h, recess, gap, split?}`: left edge, width and height of the opening; how far the frame is set back behind the wall face; gap between sill top and frame. `split` = list of mullion positions as fractions of `w` (`[0.43]` = two sashes; omit for one sash). `hinges` = optional list with one entry per sash, `"left"`, `"right"` or `null` (fixed sash): the hinge side, the handle is drawn on the opposite edge. Without `hinges` a handle is drawn left of each mullion (or near the right edge of a single sash) |
| `sill` | yes | `{top, t, out, over, apron?}`: height of the sill's top face, thickness, projection in front of the wall, overhang beyond the opening on each side. `apron: {h, t}` = trim strip on the wall right under the sill, along its whole length: height and thickness (how far it sticks out of the wall) |
| `radiators` | no | List of `{x, y, z, w, h, d, ribs, collectors?, valve?}`: min corner and size of the ribs, number of ribs, heights of the horizontal headers, and `valve: {side: "left"/"right", w}` – only widens the radiator's dimension span; draw the valve itself in `pipes` |
| `pipes` | no | List of `{name?, r, points, mat?}`. 2 points = straight cylinder, more = smooth curve through the points. `mat`: `pipe` (default), `metal`, `white`. Use for risers, branches, valves, knobs, brackets |
| `dims` | no | Extra room dimensions in the drawing: `{a, b, off?}` points in cm (the room, sill and radiator dimensions are generated automatically) |
| `info` | no | Extra rows `[label, value]` for **Tabela wymiarów pokoju** |
| `furniture` | no | Room furnishings for visualisation only (sofa, bed, wardrobe…), toggled with **Widoczność → Meble**; not in the parts list, cost or export. List of `{name, color?, parts}`; `parts` = list of boxes `{pos, size, color?, r?}` (min corner and size in cm like panels, `color` as `"#rrggbb"` – part colour overrides the item colour, `r` = edge rounding radius in cm, e.g. 3–6 for upholstery) |
| `notes` | no | Markdown lines added to `## Pomieszczenie` in the `.md` export |

### Room `Sypialnia`

Measured by the owner unless marked as an estimate.

| Thing | Occupies (cm) |
|---|---|
| Room | x 0–210, y 0–250 |
| Window sill (parapet) | x 1–209, y 100.5–104.5, z -2–7 |
| Window frame | starts at y 106.2 (1.7 gap above the sill), plane z = -2 |
| Radiator ribs | x 29–77, y 12–69, z 5–19 |
| Radiator valve + knob | x 19–29, y about 62–67, z up to about 19 (knob depth not measured) |
| Riser pipe A | Ø3, x 3.5–6.5, z 48–51, floor to ceiling |
| Riser pipe B | Ø3, x 3.5–6.5, z 53–56, floor to ceiling |
| Upper branch pipe | axis y 64.5, about Ø2.2; runs from riser A along the left wall toward the window at x 4.8–7, turns right at z 12–19 to the valve |
| Lower branch pipe | axis y 14, about Ø2.2; runs from riser B, bulges out to x 12 near z 46–50, continues at x 8.8–11, turns right at z 12–19 to the radiator |

Both branch pipes cross any front plane placed between z 19 and z 48 near the left wall. A front
panel in the left bay needs cut-outs for them, or must stay clear of x < 12 at those heights.

## Version format

The `wersje` block is a JSON array; its order is the button order. One entry:

```json
{
  "name": "v1",
  "room": "Sypialnia",
  "saved": "2026-11-12T11:02:28.000Z",
  "notes": "## Okucia i dodatki\n\n- ...\n\n## Kolejność składania\n\n1. ...\n",
  "panels": [
    {"name": "Blat na parapecie", "size": [210, 1.8, 25], "pos": [0, 104.5, -2]},
    {"name": "Klapa (półka prawa 72)", "size": [126, 1.8, 19.2], "pos": [82, 70.2, 1.8], "lid": true, "note": "klapa na zawiasach, luz 2 mm"}
  ]
}
```

`name` must be non-empty and unique within its room; the same name may be used in different rooms (each room numbers its versions from `v1`). Versions are selected per room, and the last selection is remembered per room. `room` must be the `name` of a room in the
`pokoje` block; a version with an unknown or missing `room` is skipped with a console error. Keep one panel per line. The text is inside a `<script>` tag, so write `<` as `\u003c`. A version is shown only
if `panels` is a non-empty array and every panel has a string `name`, a 3-number `size` with all
values > 0, and a 3-number `pos`; otherwise it is skipped with a console error.

### Panel fields

| Field | Required | Meaning |
|---|---|---|
| `name` | yes | Polish name shown in the parts list |
| `size` | yes | `[x, y, z]` in cm. The smallest value is the thickness; the other two are the cut format |
| `pos` | yes | `[x, y, z]` of the min corner in cm |
| `mesh` | no | `true` = 1 mm perforated sheet metal instead of board. Use thickness `0.1` |
| `hdf` | no | `true` = 3 mm HDF back panel (like IKEA backs) instead of 18 mm board. Use thickness `0.3`; not with `mesh`, `lid`, `door` or `notch`. Drawn in a lighter grey, listed in its own section of the `.md` export and priced per m² (`hdf`) instead of in the board sheets |
| `tilt` | no | Leaning panel, thin in **z** (e.g. a sloped back): `pos` is its bottom rear corner, `size[1]` its true length along the slope, and the top edge is shifted `tilt` cm toward the room (negative = toward the wall). Not with `mesh`, `lid`, `door` or `notch`. The overlap script checks it as an upright box, so check it by eye |
| `lid` | no | `true` = hinged flap. Must be horizontal (thin in y). Pivots about its rear top edge (min z, max y) and swings up toward the wall when **Klapy i drzwiczki otwarte** is ticked. Three hinges are drawn automatically |
| `door` | no | Side door on a vertical panel thin in **x** (not with `lid`, `mesh` or `notch`). Value `"front-right"`, `"front-left"`, `"back-right"` or `"back-left"`: hinge edge (`front` = max z, `back` = min z) and the side it swings to (`right` = +x). Two hinges are drawn; swings 85° with **Klapy i drzwiczki otwarte** |
| `note` | no | Short Polish remark. Shown as a ⚠ tooltip in the parts list and in the `.md` export |
| `notch` | no | `[z depth, y height]` rectangular cut-out at the top rear corner. Only for vertical side panels (thin in x) |
| `step` | no | Integer > 0: number of the step in `## Kolejność składania` in which this panel is mounted. Drives the **Składanie** slider. A panel without `step` appears only at the end. Every panel in the current versions has one |

Renderer limits to respect:

- Furniture board is always `1.8` thick. Sheet metal is `0.1`. HDF backs (`hdf`) are `0.3`.
- A vertical `mesh` panel must be thin in **z** (facing the room) or in **x** (facing sideways).
  A horizontal `mesh` panel only renders correctly when it is also a `lid`.
- Panels are boxes. Anything else (cut-outs for pipes, rounded corners) cannot be drawn – describe
  it in `note` and in `notes`.
- Round numbers to 2 decimals.

### `notes`

A Markdown string appended to the `.md` export after the parts tables. Write it in Polish, with
dimensions in **millimetres**, using these three sections:

- `## Okucia i dodatki` – hinges, fixings, edge banding, glue
- `## Kolejność składania` – numbered assembly steps that match the panels in this file
- `## Do sprawdzenia przed zamówieniem` – measurements and assumptions the owner must verify

Rewrite the notes whenever the panels change; stale assembly steps are worse than none. Keep each
step as `N. **Tytuł.** …` – the slider reads the titles from that pattern – and keep the panels'
`step` numbers in sync when steps are added, removed or renumbered. Keep the `panels` array sorted
by `step` (within a step, in the order the step's text mounts them): the parts list and the `.md`
export follow the array order.

## Adding a new design

1. Copy the closest existing entry of the same room in the `wersje` block and append it as a new
   entry (or after the one it is based on); keep its `room`. Never change an existing version unless the owner asks for that.
2. Set a `name` that is new in that room (the next `vN`, optionally `vN-<short-slug>`) and `saved`, and edit `panels`.
3. Check every panel against its room (the obstacle table for `Sypialnia` above) and against the
   other panels. Panels may touch but must not overlap. This script reads the obstacles from the
   version's room – the sill, the radiators and every straight (2-point) pipe – and prints overlaps:

   ```bash
   python3 - '<version name>' <<'EOF'
   import json, re, sys, itertools
   html = open('index.html', encoding='utf-8').read()
   block = lambda id: json.loads(re.search(r'<script type="application/json" id="%s">\n(.*?)\n</script>' % id, html, re.S).group(1))
   d = next(v for v in block('wersje') if v['name'] == sys.argv[1])
   r = next(x for x in block('pokoje') if x['name'] == d['room'])
   (W, H), win, sill = r['size'], r['window'], r['sill']
   sx = max(0, win['x'] - sill['over'])
   obstacles = {'parapet': ([sx, sill['top'] - sill['t'], -win['recess']],
                            [min(win['w'] + 2 * sill['over'], W - sx), sill['t'], win['recess'] + sill['out']])}
   if 'apron' in sill:
       ap, sl = sill['apron'], min(win['w'] + 2 * sill['over'], W - sx)
       obstacles['listwa pod parapetem'] = ([sx, sill['top'] - sill['t'] - ap['h'], 0], [sl, ap['h'], ap['t']])
   for i, k in enumerate(r.get('radiators', [])):
       obstacles['kaloryfer %d' % (i + 1)] = ([k['x'], k['y'], k['z']], [k['w'], k['h'], k['d']])
   for i, p in enumerate(r.get('pipes', [])):   # tylko proste odcinki (2 punkty); łuki sprawdzić na oko
       if len(p['points']) != 2: continue
       a, b, rr = *p['points'], p['r']
       lo = [min(a[k], b[k]) - (rr if a[k] == b[k] else 0) for k in range(3)]
       hi = [max(a[k], b[k]) + (rr if a[k] == b[k] else 0) for k in range(3)]
       obstacles[p.get('name', 'rura %d' % (i + 1))] = (lo, [hi[k] - lo[k] for k in range(3)])
   boxes = [(p['name'], p['pos'], p['size']) for p in d['panels']]
   def hit(a, b, eps=0.01):
       return all(a[1][k] + a[2][k] > b[1][k] + eps and b[1][k] + b[2][k] > a[1][k] + eps for k in range(3))
   for a, b in itertools.combinations(boxes, 2):
       if hit(a, b): print('OVERLAP', a[0], '<->', b[0])
   for a in boxes:
       for n, (pos, size) in obstacles.items():
           if hit(a, (n, pos, size)): print('HITS', a[0], '->', n)
       lo, hi = a[1], [a[1][k] + a[2][k] for k in range(3)]
       if lo[0] < 0 or hi[0] > W or lo[1] < 0 or hi[1] > H or lo[2] < -win['recess']:
           print('OUTSIDE ROOM', a[0])
   EOF
   ```

   Curved pipes (3+ points, e.g. the branch pipes) are not in the script – check them by eye in the viewer.
   A panel that wraps the sill on purpose (like a notched side) will be reported; that is fine if
   it has a matching `notch`.
4. Rewrite `notes` for the new design.
5. Select the version in the viewer and look at it from the front, the side and with **Pokój: szkielet** (there are no preset views – rotate the camera by dragging, e.g. Playwright `mouse.move`/`down`/`move`/`up` on the canvas).
   Tick **Klapy i drzwiczki otwarte** if there are lids. Check the browser console for errors.
6. Tell the owner the version name and list the cut sizes in mm. State every assumption you made.

## Construction rules agreed with the owner

- Material: 18 mm laminated furniture board, cut to order. The deliverable is the list of board
  sizes in mm.
- The unit must cover the sill, give storage, and mask the radiator. Two cats (2 kg and 5 kg) sit
  on it and plant pots stand on it.
- Board strips narrower than about 50–60 mm are too weak to carry load or take screws in the edge.
- A horizontal board spanning more than about 80 cm needs a stiffening rail, a support, or to be
  fixed to a front panel.
- A flap needs something to rest on at the front (a rail, or the top of a frame) and 2 mm clearance
  at the sides.
- Keep the radiator bay ventilated: perforated metal (≥ 40 % open area), air in at the bottom, out
  at the top. Do not box the radiator in with solid board.
- Board sizes in the data are nominal, with no allowance for uneven walls. Say so in `notes`.

Assumptions about `Sypialnia` not yet confirmed by the owner – keep them visible in `notes`:

- The 1.7 cm gap between sill and window is vertical (the window frame starts above the sill).
- The radiator width of 58 cm includes the valve and knob.
- The sill ends 1 cm short of each side wall.
- No skirting boards.

## Changing the viewer (`index.html`)

Only when the owner asks for a viewer feature. Things to know:

- `DEFAULT_PANELS` stays an empty array – used only when there is no valid version.
- The room is data, not code: room geometry belongs in the `pokoje` block. `ROOMS` is the parsed
  block (only rooms passing `validRoom()`), `ROOM` the selected room. `selectRoom(r)` sets `ROOM`,
  calls `buildRoom()` (clears and rebuilds the `room` group, sun, **Tabela wymiarów pokoju** and,
  via `buildRoomDims()`, the room dimensions; keeps the room visibility mode), resets the camera
  (`resetView()`, start view from `START_VIEW()` – same distance in every room, centred on its width), filters the version buttons to that room and selects a version.
- `VERSIONS` is the parsed `wersje` block; `current` is the selected version (`null` only when the room has none).
  `PANELS` is `current.panels`, or the empty default only when there is no valid version. `select(v)` rebuilds the panels, parts list
  and furniture dimensions (`buildPanels`, `buildCut`, `buildProjDims`) without a reload.
- 3D meshes are kept in `panelMeshes[i]`, never on the panel objects.
- The **Widoczność** rows remember their last mode (`pełny/szkielet/ukryty` per row: `room`, `panels`, `furn`) in `localStorage` under `parapet.widocznosc`; restored before the first draw. Room furniture defaults to hidden.
- `validPanels()` is the single place that defines what a showable design is.
- Avoid `alert`/`confirm`/`prompt`; show messages in the page.
- Under the parts list, **Szacowany koszt** (`costHtml`, `fillCost`) estimates the price from the
  panels: board in whole 2800×2070 sheets with 25 % waste, cutting and edge banding as half the
  sum of perimeters, perforated sheet by area + 20 %, plus a lump sum for hardware. Unit prices
  (`PRICE_DEF`) are rough guesses; the owner can edit them in the page, and they are remembered
  in `localStorage` under `parapet.ceny`. The `.md` export includes the same table (`costMd`),
  with the prices currently entered, between the sheet-metal table and `notes`.
- **Składanie** slider (`buildSteps`, `applyStep`): position 0 = empty room, then one position
  per distinct panel `step`, last = everything. Panels of later steps are hidden (with their
  hinges – `panelRoots[i]` holds all scene objects of panel i), panels of the current step get
  orange edges, and their dimensions follow. `select(v)` resets it to the full unit.
- **Mobile** (`max-width: 760px` or `max-height: 500px`, `mobileUI()`): both cards are hidden and
  open one at a time as sheets from the bottom bar `#mbar` (class `open` on `#views` / `#cut`;
  sideways on a landscape phone). Sheet + bar take at most half the screen and the sections keep their desktop
  order; `updateViewOffset()` shifts the
  render centre (`camera.setViewOffset`) into the free half while a sheet is open; it only sets the target (`vo`), and
  `stepViewOffset()` in the render loop eases toward it. On touch screens (`touchUI()`, `hover: none`), tapping a parts-list
  row toggles the highlight and tapping ⚠ shows the note. In portrait, `resetView` moves the camera
  back (`viewFit`) so the whole wall fits, and `resize` rescales it when the phone is rotated.

## Verifying with Playwright

```
navigate  http://127.0.0.1:48765/index.html?v=<n>
click     #roomList button:has-text("<room name>")
click     #verList button:has-text("<version name>")
screenshot                    (must be saved under the project folder)
console messages (level error) – expect none
```

Useful selectors: visibility
`.seg[data-g="room|panels|furn"] button[data-m="on|ghost|off"]` (rows **Pokój**, **Projekt** = the designed unit, **Meble** = room `furniture`), `#dimsToggle` (room), `#projToggle` (project), `#mebleToggle` (room furniture) – all drawing dimensions are in cm,
`#lidToggle`, assembly slider `#stepRange` (+ `#stepPrev`, `#stepNext`, label `#stepLabel`),
parts list `#cut summary`, mobile bar `#mbar button[data-sheet="views|cut"]` (use a 390×844 viewport), row `#cut tr[data-i="N"]`, export `#exportMd`.
