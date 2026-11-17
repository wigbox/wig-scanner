# סורק פאות — Wig Scanner

A single-page mobile web app (`index.html`, Hebrew / RTL) for the Wigbox team. Look up a wig by number or by scanning its barcode with the phone camera. The app shows where the wig is, whether it needs to move, and lets you confirm the move.

No backend and no build step. It is one HTML file that reads Google Sheets live.

**Repo:** https://github.com/wigbox/wig-scanner
**Live app:** https://wigbox.github.io/wig-scanner/

## What it does

1. **Look up a wig** by typing its number, scanning the barcode (📷 button), or opening `…/?id=12345`.
2. **Shows the result** as a colour-coded card:

| Card | Meaning |
|---|---|
| 🟢 No change | In stock, nothing pending |
| 🔵 In storage | Currently in a drawer (מגירה), nothing pending |
| ⚪ In wash | Currently in חפיפה |
| 🟠 Action needed | Wig is on a to-display or to-storage list |
| 🔴 Left inventory | Sold, personal use, barter, returned, missing or damaged |
| 🟣 On order | Ordered, reserved or wish list |
| ❌ Not found | Wig number is not in MASTER_DATA |

3. **Confirm a move.** Tapping a confirm button sends the wig id, the action, the new status, the description, the box/category, the current location and a timestamp to an automation in Boost.space (Integrator) via a webhook. Everything that happens after that is handled inside the automation, not in this app.

## Data source

Google Sheet **"Extra Inventory Managment"** (ID `1GaCqafFxWIAVYZgZ-pKSPX3LmDvgkGnUuZZeqDVk6eo`). The app is read-only on the sheet. Each tab is fetched as CSV through Google's `gviz/tq` endpoint, on every lookup, with no caching.

| Tab | gid | Columns used | Used for |
|---|---|---|---|
| `MASTER_DATA` | 1059386396 | C = S/N (wig id), D = Status | Does the wig exist, and is it in stock, sold, on order, etc. |
| `STORAGE_DATA` | 450172942 | A = WIG ID, B = LOCATION (`מגירה` / `חפיפה`) | Current location. A wig not listed here is assumed to be on display. |
| `לתצוגה` | 731318979 | A = description, B = wig id, C = ארגז | Wigs pending a move **to display** |
| `לאחסון` | 1556036128 | A = description, B = wig id, C = ארגז, D = בוצע | Wigs pending a move **to storage**. Rows with בוצע = TRUE are ignored. |

Master status mapping: `Sold` / `Personal Use` / `Barter` / `Returned` / `Missing` / `Damaged` mean the wig has left inventory. `Ordered` / `Order Reserved` / `Inventory Reserved` / `Wish List` / `Fix-Return` mean on order. `In Stock` / `Consignment` (and anything unrecognised) mean in stock.

### Lookup order

1. Find the wig in `MASTER_DATA`. If it isn't there, show "not found".
2. If its status is not in stock, show that status and stop. If it is also in חפיפה, offer to clear the wash status.
3. Otherwise read the current location from `STORAGE_DATA`.
4. Check `לתצוגה`, then `לאחסון`. A match means "action needed" with a confirm button.
5. If neither list has it, show the current location with optional move buttons.

## Setup requirements

- **The sheet must be shared as "Anyone with the link → Viewer."** The app fetches it anonymously from the browser.
- **Camera scanning needs HTTPS.** GitHub Pages works. Plain `http://` will not get camera permission.
- Barcode scanning uses [html5-qrcode](https://github.com/mebjas/html5-qrcode), loaded from jsDelivr. The scanned value is reduced to digits, so the barcode must encode the wig number.
- The webhook URL is the `WEBHOOK` constant at the top of the `<script>` block.

## Keeping it working

The app depends on the sheet's exact structure, and **it breaks silently if the sheet changes**:

- Columns are read **by position, not by header name.** Inserting or deleting a column shifts the results.
- Tabs are referenced **by gid, not by name.** Deleting and recreating a tab gives it a new gid. The gid is the number in the URL after `gid=`.
- This has already happened once: `HELPER` was split into `לתצוגה` and `לאחסון`, and `STORAGE_DATA` lost a column.
- If a tab can't be loaded, the error card names the failing tab and links to its raw CSV for debugging.

## Known data issue

A wig that is on a pending list but missing from `MASTER_DATA` shows "not found." As of the last check, 53 of the `לאחסון` entries (for example 26515) were not in `MASTER_DATA`. Fix this in the sheet, not in the app.

## Files

| File | Purpose |
|---|---|
| `index.html` | The whole app: markup, styles and logic |
