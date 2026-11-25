# סורק פאות — Wig Scanner

A single-page mobile web app (`index.html`, Hebrew / RTL) for the Wigbox team. Look up a wig by number or by scanning its barcode with the phone camera. The app shows where the wig is, whether it needs to move, and lets you confirm the move, including which drawer it was really put in.

No backend and no build step. It is one HTML file that reads a Google Sheet live.

**Repo:** https://github.com/wigbox/wig-scanner
**Live app:** https://wigbox.github.io/wig-scanner/

## What it does

1. **Look up a wig** by typing its number, scanning the barcode (📷 button), or opening `…/?id=12345`.
2. **Shows the result** as a colour-coded card:

| Card | Meaning |
|---|---|
| 🟢 No change | In stock on display, nothing pending |
| 🔵 In storage | Currently in a drawer, nothing pending. Shows which drawer if it was placed with this app |
| ⚪ In wash | Currently in חפיפה |
| 🟠 Action needed | Wig is on the "to drawer" or "to display" list |
| 🔴 Left inventory | Sold, personal use, barter, returned, missing, damaged, or **any other status** (Other, Cancelled, Never Made, empty) |
| 🟣 On order | Ordered, reserved, wish list or fix-return |
| ❌ Not found | Wig number is not in Full Inventory |

3. **Confirm a move.** Tapping a confirm button sends the move to an automation in Boost.space (Integrator) via a webhook. Everything after that is done by the automation, not by this app.

### Moving a wig to a drawer

The list says which drawer (e.g. `A2`). The card shows **"הנחתי במגירה A2"**. One tap if it went there.

If it was put somewhere else, tap **"הונחה במגירה אחרת"**, type the drawer (a letter A–O and a number 1–7, for example `A3`, upper or lower case) and confirm. A code that isn't a real drawer is rejected and the confirm button stays disabled. The webhook then carries both drawers:

| Webhook field | Example | Meaning |
|---|---|---|
| `planned_drawer` | `A2` | The drawer the list asked for |
| `actual_drawer` | `A3` | The drawer it was really put in (same as planned when it went where asked) |
| `drawer_changed` | `true` | `true` only when they differ |

**Coming out of wash.** The card says what is expected. A wig with a drawer in `Bin List` shows "אחרי החפיפה: להעביר למגירה X" (one tap, or type the drawer it really went to). A wig with no drawer in `Bin List` shows "אחרי החפיפה: להעביר לתצוגה", with a second option to type a drawer if it goes into one. (The rule "no drawer in `Bin List` means display" matches how the sheet is built today: the 70 wigs meant for display are all absent from `Bin List`.)

**Correcting a drawer.** On the "in storage, no change" card, **"הפאה במגירה אחרת — עדכני מגירה"** lets you type the drawer the wig is really in (for example after it was put in the wrong one). It sends `action` = `עדכון מגירה` with `current_location` = `באחסון`, and the Boost scenario's "update row" route accepts that word and rewrites column C. If the wig has no row in `סטטוס פאות`, nothing is updated.

Other moves into storage (a wig on display, or coming out of wash, with no drawer on the list) also ask which drawer. The box is required, `planned_drawer` is empty and `actual_drawer` holds the choice. Moves to display do not ask for a drawer.

The existing fields (`wig_id`, `action`, `zoho_status`, `description`, `location`, `current_location`, `timestamp`) are unchanged.

The accepted drawer codes are defined by the `isValidDrawer` function near the top of the script (letters A–O, numbers 1–7).

On "in storage, no change" cards the app also shows the drawer the wig **should** be in (from `Bin List`) and, when known, the drawer it was placed in. If they differ, the card says so.

## Data source

Google Sheet **"Storage Management"** (ID `1cexgD9BIzu-zX51FFQCuyJ-IlY0O7DlKSVjD3gXXSYw`). The app only reads it. Each tab is fetched as CSV through Google's `gviz/tq` endpoint on every lookup, with no caching.

| Tab | gid | Columns used | Used for |
|---|---|---|---|
| `Full Inventory` | 210643961 | A = S/N, B = Status, C = Location (`DISPLAY` / `STORAGE` / `Not In Store`) | Does the wig exist, its status, and whether it is on display or in storage |
| `סטטוס פאות` | 2057793530 | A = S/N, B = `מגירה` / `חפיפה`, C = drawer it was placed in | Wash status, and the drawer a wig was placed in |
| `למגירה` | 0 | A = S/N, B = action, C = drawer to move to, D = wig code, E = drawer group | Wigs pending a move **to a drawer** |
| `לתצוגה` | 1 | A = S/N, B = action, C = drawer it is in now, D = wig code, E = drawer group | Wigs pending a move **to display** |
| `Bin List` | 759975009 | A = drawer (bin), B = S/N | The drawer each wig should be in |

Status mapping (not case-sensitive):

- **In stock:** `In Stock`, `Consignment`
- **On order:** `Ordered`, `Order Reserved`, `Inventory Reserved`, `Wish List`, `Fix-Return`
- **Left inventory:** `Sold`, `Personal Use`, `Barter`, `Returned`, `Missing`, `Damaged`, and everything else

### Lookup order

1. Find the wig in `Full Inventory`. If it isn't there, show "not found".
2. If its status is not in stock, show that status and stop. If it is also in חפיפה, offer to clear the wash status.
3. Current location: wash (from `סטטוס פאות`) wins, otherwise `Full Inventory` (`STORAGE` = in a drawer, otherwise display).
4. Check `לתצוגה`, then `למגירה`. A match means "action needed" with a confirm button.
5. If neither list has it, show the current location with optional move buttons.

## What the Boost automation does with a confirmed move

(Handled in Boost, described here so it is written down in one place.)

- **Moved to a drawer:** the wig's row in `סטטוס פאות` becomes `מגירה`, with `actual_drawer` (the drawer it was really put in, even when it matches the plan) in column C. The planned drawer is not stored: the app reads it live from `Bin List`, and Boost's run history keeps `planned_drawer` and `drawer_changed`.
- **Moved to wash:** the row becomes `חפיפה`, with column C emptied.
- **Moved to display:** the wig's row is removed from `סטטוס פאות`, including any drawer information.

## Setup requirements

- **The sheet must be shared as "Anyone with the link → Viewer."** The app fetches it anonymously from the browser.
- **Camera scanning needs HTTPS.** GitHub Pages works. Plain `http://` will not get camera permission.
- Barcode scanning uses [html5-qrcode](https://github.com/mebjas/html5-qrcode), loaded from jsDelivr. The scanned value is reduced to digits, so the barcode must encode the wig number.
- The webhook URL is the `WEBHOOK` constant at the top of the `<script>` block.

## Keeping it working

The app depends on the sheet's exact structure, and **it breaks silently if the sheet changes**:

- Columns are read **by position, not by header name.** Inserting or deleting a column shifts the results.
- Tabs are referenced **by gid, not by name.** Deleting and recreating a tab gives it a new gid. The gid is the number in the URL after `gid=`.
- If a tab can't be loaded, the error card names the failing tab and links to its raw CSV for debugging.

## Files

| File | Purpose |
|---|---|
| `index.html` | The whole app: markup, styles and logic |
