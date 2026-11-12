# Parapet – window-sill furniture designer

A single-page 3D viewer (`index.html`, three.js) for designing a DIY built-in unit that covers a
window sill, adds storage and hides a radiator. The room is fixed. Furniture designs are data:
one JSON file per design in `wersje/`. **Adding a new design means writing a new JSON file – not
editing `index.html`.**

The owner speaks Polish. All UI text, element names and assembly notes are in Polish. Reply in
the language the owner writes in.

## Run and view

```bash
cd /home/kamil/Desktop/claude-parapet
python3 -m http.server 48765 --bind 127.0.0.1     # port 8765 is taken by something else
```

Open `http://127.0.0.1:48765/index.html`. three.js 0.160 loads from the jsDelivr CDN, so internet
is needed. The browser caches aggressively: reload with Ctrl+Shift+R, or add `?v=N` to the URL.

The page starts with an empty room. A design is loaded with **Wczytaj z pliku** in the left panel;
it is then kept in `localStorage` (key `parapet.wersja.v1`) and survives a refresh. **Wyczyść
wczytany plik** returns to the empty room.

## Files

| Path | What it is |
|---|---|
| `index.html` | Viewer: room geometry, panel renderer, dimensions, parts list, export, load/save |
| `wersje/mebel-*.json` | Furniture designs, one per file. `mebel-v1.json` is the reference design (17 parts, passes the overlap check) |
| `pokoj.jpeg`, `rzut-rur.jpeg` | Owner's photo of the room and top-down sketch of the pipes |
| `.playwright-mcp/` | Scratch screenshots from testing; safe to ignore |

## Coordinate system

Units are **centimetres** in the JSON and in the code. The parts list and exports show millimetres.

- **x** – from the left wall to the right: `0 … 210`
- **y** – from the floor up: `0 … 250`
- **z** – from the face of the wall under the window, into the room. The window is recessed, so
  its plane is at `z = -2`.

A panel is an axis-aligned box: `pos` is its corner with the smallest x, y and z (left, bottom,
nearest the window wall); `size` is `[x width, y height, z depth]`.

## The room (fixed – do not change)

Defined in the `R` object at the top of the script in `index.html`. Measured by the owner unless
marked as an estimate.

| Thing | Occupies (cm) |
|---|---|
| Room | x 0–210, y 0–250 |
| Window sill (parapet) | x 1–209, y 100.5–104.5, z -2–7 |
| Window frame | starts at y 106.2 (1.7 gap above the sill), plane z = -2 |
| Radiator ribs | x 29–77, y 10–69, z 5–19 (bottom y 10 is an estimate) |
| Radiator valve + knob | x 19–29, y about 62–67, z up to about 19 (knob depth not measured) |
| Riser pipe A | Ø3, x 3.5–6.5, z 48–51, floor to ceiling |
| Riser pipe B | Ø3, x 3.5–6.5, z 53–56, floor to ceiling |
| Upper branch pipe | axis y 64.5, about Ø2.2; runs from riser A along the left wall toward the window at x 4.8–7, turns right at z 12–19 to the valve |
| Lower branch pipe | axis y 14, about Ø2.2; runs from riser B, bulges out to x 12 near z 46–50, continues at x 8.8–11, turns right at z 12–19 to the radiator |

Both branch pipes cross any front plane placed between z 19 and z 48 near the left wall. A front
panel in the left bay needs cut-outs for them, or must stay clear of x < 12 at those heights.

## Design file format

```json
{
  "format": "parapet-mebel",
  "formatVersion": 1,
  "name": "v1",
  "saved": "2026-11-12T11:02:28.000Z",
  "units": "cm; size = [X szer., Y wys., Z głęb.], pos = lewy dolny tylny narożnik",
  "notes": "## Okucia i dodatki\n\n- ...\n\n## Kolejność składania\n\n1. ...\n",
  "panels": [
    { "name": "Blat na parapecie", "size": [210, 1.8, 25], "pos": [0, 104.5, -2] },
    { "name": "Klapa (półka prawa 72)", "size": [126, 1.8, 19.2], "pos": [82, 70.2, 1.8],
      "lid": true, "note": "klapa na zawiasach, luz 2 mm" }
  ]
}
```

The loader rejects a file unless `format` is exactly `"parapet-mebel"`, `panels` is a non-empty
array, and every panel has a string `name`, a 3-number `size` with all values > 0, and a
3-number `pos`.

### Panel fields

| Field | Required | Meaning |
|---|---|---|
| `name` | yes | Polish name shown in the parts list |
| `size` | yes | `[x, y, z]` in cm. The smallest value is the thickness; the other two are the cut format |
| `pos` | yes | `[x, y, z]` of the min corner in cm |
| `mesh` | no | `true` = 1 mm perforated sheet metal instead of board. Use thickness `0.1` |
| `lid` | no | `true` = hinged flap. Must be horizontal (thin in y). Pivots about its rear top edge (min z, max y) and swings up toward the wall when **Klapy i drzwiczki otwarte** is ticked. Three hinges are drawn automatically |
| `door` | no | Side door on a vertical panel thin in **x** (not with `lid`, `mesh` or `notch`). Value `"front-right"`, `"front-left"`, `"back-right"` or `"back-left"`: hinge edge (`front` = max z, `back` = min z) and the side it swings to (`right` = +x). Two hinges are drawn; swings 85° with **Klapy i drzwiczki otwarte** |
| `note` | no | Short Polish remark. Shown as a ⚠ tooltip in the parts list and in the `.md` export |
| `notch` | no | `[z depth, y height]` rectangular cut-out at the top rear corner. Only for vertical side panels (thin in x) |

Renderer limits to respect:

- Furniture board is always `1.8` thick. Sheet metal is `0.1`.
- A vertical `mesh` panel must be thin in **z** (facing the room). A horizontal `mesh` panel only
  renders correctly when it is also a `lid`.
- Panels are boxes. Anything else (cut-outs for pipes, rounded corners) cannot be drawn – describe
  it in `note` and in `notes`.
- Round numbers to 2 decimals.

### `notes`

A Markdown string appended to the `.md` export after the parts tables. Write it in Polish, with
dimensions in **millimetres**, using these three sections:

- `## Okucia i dodatki` – hinges, fixings, edge banding, glue
- `## Kolejność składania` – numbered assembly steps that match the panels in this file
- `## Do sprawdzenia przed zamówieniem` – measurements and assumptions the owner must verify

Rewrite the notes whenever the panels change; stale assembly steps are worse than none.

## Adding a new design

1. Copy the closest existing file in `wersje/` to `wersje/mebel-<short-slug>.json`. Never
   overwrite an existing version unless the owner asks for that.
2. Set a new `name` and edit `panels`.
3. Check every panel against the obstacle table above and against the other panels. Panels may
   touch but must not overlap. This script prints overlaps:

   ```bash
   python3 - wersje/mebel-<short-slug>.json <<'EOF'
   import json, sys, itertools
   d = json.load(open(sys.argv[1]))
   obstacles = {
       'parapet': ([1, 100.5, -2], [208, 4, 9]),
       'kaloryfer': ([29, 10, 5], [48, 59, 14]),
       'zawór': ([19, 62, 9], [10, 5, 10]),
       'pion A': ([3.5, 0, 48], [3, 250, 3]),
       'pion B': ([3.5, 0, 53], [3, 250, 3]),
   }
   boxes = [(p['name'], p['pos'], p['size']) for p in d['panels']]
   def hit(a, b, eps=0.01):
       return all(a[1][k] + a[2][k] > b[1][k] + eps and b[1][k] + b[2][k] > a[1][k] + eps for k in range(3))
   for a, b in itertools.combinations(boxes, 2):
       if hit(a, b): print('OVERLAP', a[0], '<->', b[0])
   for a in boxes:
       for n, (pos, size) in obstacles.items():
           if hit(a, (n, pos, size)): print('HITS', a[0], '->', n)
       lo, hi = a[1], [a[1][k] + a[2][k] for k in range(3)]
       if lo[0] < 0 or hi[0] > 210 or lo[1] < 0 or hi[1] > 250 or lo[2] < -2:
           print('OUTSIDE ROOM', a[0])
   EOF
   ```

   The branch pipes are curved and are not in the script – check them by eye in the viewer.
   A panel that wraps the sill on purpose (like a notched side) will be reported; that is fine if
   it has a matching `notch`.
4. Rewrite `notes` for the new design.
5. Load the file in the viewer and look at it from the front, the side and with **Pokój: szkielet**.
   Tick **Klapy i drzwiczki otwarte** if there are lids. Check the browser console for errors.
6. Tell the owner the file path and list the cut sizes in mm. State every assumption you made.

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
- Board sizes in the file are nominal, with no allowance for uneven walls. Say so in `notes`.

Assumptions not yet confirmed by the owner – keep them visible in `notes`:

- The 1.7 cm gap between sill and window is vertical (the window frame starts above the sill).
- The radiator width of 58 cm includes the valve and knob.
- The radiator bottom is 10 cm above the floor.
- The sill ends 1 cm short of each side wall.
- No skirting boards.

## Changing the viewer (`index.html`)

Only when the owner asks for a viewer feature. Things to know:

- `DEFAULT_PANELS` stays an empty array – the default view is the empty room.
- Do not change the values in `R` unless the owner gives new measurements.
- `PANELS` is either the stored design or the empty default. The parts list, dimension labels,
  hover highlight and `.md` export are all generated from it.
- `validPanels()` is the single place that defines what a loadable design is.
- Saving is disabled when there are no panels.
- Avoid `alert`/`confirm`/`prompt`; show messages in the page.

## Verifying with Playwright

```
navigate  http://127.0.0.1:48765/index.html?v=<n>
click     #verLoad            → then upload the JSON file
screenshot                    (must be saved under the project folder)
console messages (level error) – expect none
click     #verReset           → back to the empty room when done
```

Useful selectors: view buttons `button[data-v="persp|front|top|side"]`, visibility
`.seg[data-g="room|panels"] button[data-m="on|ghost|off"]`, `#dimsToggle`, `#furnToggle`,
`#lidToggle`, parts list `#cut summary`, row `#cut tr[data-i="N"]`, export `#exportMd`.
