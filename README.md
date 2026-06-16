# Inventory Tracker

A lightweight internal web app for managing team inventory — reservations, borrowing, returns, and condition tracking. No backend server, no database, no monthly bills. Everything runs in the browser and stores data directly in this GitHub repository.

**Live app → [https://pollats.github.io/inventory_tracker/](https://pollats.github.io/inventory_tracker/)**

---

## What it does

- **Reserve items** — book inventory for a date range, with real-time availability checking based on total unit counts
- **Borrow / Return** — check items out against an active reservation, return them with an optional condition photo
- **Returns history** — full archive of every return, including who borrowed it, who returned it, days held, and condition photo
- **Manage inventory** — add, edit, or remove items directly from the app without touching any code

---

## How it works

The app is a single HTML file (`index.html`) hosted on GitHub Pages. It reads and writes data directly to CSV files in this repository using the GitHub API. There is no server, no database, and no external service beyond GitHub itself.

```
pollats/inventory_tracker/
├── index.html          # The entire app — HTML, CSS, and JavaScript
├── data/
│   ├── items.csv       # Inventory list (categories, items, unit counts)
│   ├── reservations.csv  # Active and upcoming reservations
│   ├── checkouts.csv   # Items currently borrowed
│   └── archive.csv     # Completed returns with condition photos
└── README.md
```

---

## Data files

### `data/items.csv`
Defines your inventory. Edit this directly or use the **Manage inventory** section of the app.

| Column | Description |
|---|---|
| Category | Groups items in the app (e.g. Audio Visual, IT Equipment) |
| Item Name | Display name of the item |
| ID / Model Number | Optional asset tag or model number |
| Total Units | How many of this item exist in total |
| Units Available | How many are currently on the shelf (decremented on borrow, incremented on return) |

### `data/reservations.csv`
Written by the app when someone makes a reservation. Each row is one booking.

| Column | Description |
|---|---|
| Category | Copied from items at booking time |
| Item Name | The reserved item |
| ID / Model Number | Copied from items |
| Reserved By | Name entered by the user |
| Start Date | First day of reservation (YYYY-MM-DD) |
| End Date | Last day of reservation (YYYY-MM-DD) |
| Timestamp | Unique ID for the reservation (ISO timestamp) |

### `data/checkouts.csv`
Written by the app when someone takes an item. One row per active or completed borrow.

| Column | Description |
|---|---|
| Item Name | The borrowed item |
| ID / Model Number | Copied from reservation |
| Reservation ID | Links back to the reservation (matches Timestamp in reservations.csv) |
| Taken By | Name of the person who borrowed it |
| Taken Timestamp | When they took it |
| Returned By | Name of person who returned it (filled on return) |
| On Behalf Of | Optional — if someone returned it for someone else |
| Return Timestamp | When it was returned |
| Photo Path | Unused (photos are stored inline in archive.csv) |
| Status | `taken` or `returned` |

### `data/archive.csv`
A permanent log of every completed return. Never deleted — grows over time.

| Column | Description |
|---|---|
| Item Name | The returned item |
| ID / Model Number | Asset tag if applicable |
| Category | Item category |
| Reserved By | Who originally borrowed it |
| Reservation Start / End | Original booking dates |
| Taken Timestamp | When they actually took it |
| Returned By | Who physically returned it |
| On Behalf Of | If returned by someone else |
| Return Timestamp | When it was returned |
| Days Held | Calculated from taken → returned timestamps |
| Photo | Base64-encoded JPEG condition photo (500px wide, 50% quality), if one was taken |

---

## Access token

Writing to the CSV files requires a GitHub Personal Access Token. Reading (checking availability, browsing the app) is public and requires no token.

The token is only requested when someone submits a change (reservation, borrow, or return). It is never stored in the source code. On first write, the app prompts for the token and optionally saves it to `localStorage` on that device so users are not prompted again.

**To create a token:**
1. Go to GitHub → Settings → Developer settings → Personal access tokens → Fine-grained tokens
2. Click **Generate new token**
3. Set **Repository access** to only this repo (`pollats/inventory_tracker`)
4. Under **Permissions → Repository permissions → Contents**, set to **Read and write**
5. Generate and share the token with your team (treat it like a shared password)

If a token is invalid or expired, the app will clear it and prompt again on the next write attempt.

---

## Availability logic

When someone tries to reserve an item, the app calculates:

```
Available units = Units Available − active reservations overlapping the requested dates
```

`Units Available` already reflects how many are physically on the shelf (it decrements when someone takes an item and increments when they return it). Reservations are checked on top of that. The booking is blocked only when the result hits zero.

Two reservations overlap if:

```
existing_start ≤ requested_end  AND  existing_end ≥ requested_start
```

---

## Condition photos

When returning an item, users can optionally take or upload a photo. The app:
1. Compresses the image to a maximum of 500px wide at 50% JPEG quality (~20–50 KB)
2. Converts it to a base64 string
3. Stores the string directly in the `Photo` column of `archive.csv`

To view a photo, open **Returns history** in the app and click **View condition photo** on any archive row. The browser decodes the base64 string back into an image on the fly — no image file is needed.

---

## Hosting

The app is hosted on **GitHub Pages** from the `main` branch root. Any push to `main` automatically redeploys the live app within 1–2 minutes.

To update the app, edit `index.html` and push to `main`. To update inventory without using the app, edit `data/items.csv` directly on GitHub.

---

## Tech stack

| Layer | Technology | Cost |
|---|---|---|
| Frontend | HTML + CSS + vanilla JavaScript | Free |
| Hosting | GitHub Pages | Free |
| Data storage | CSV files in this repo via GitHub API | Free |
| Write authentication | GitHub fine-grained Personal Access Token | Free |

No frameworks, no build step, no CI/CD configuration, no paid services.

---

## Known limitations

- **Race condition** — if two people submit a change at the exact same millisecond, one write may conflict. The app retries up to 4 times with backoff. Extremely unlikely in practice for an internal tool at this scale.
- **CSV size** — condition photos stored as base64 will grow `archive.csv` over time (~30–60 KB per photo). GitHub handles files up to 100 MB, so this is not a concern for years of normal use.
- **No authentication** — anyone with the app URL can read availability and browse reservations. Write access requires the shared token.
- **No cancellation from the borrow flow** — a taken item can only be cleared by returning it. Admins can manually edit `checkouts.csv` on GitHub if needed.
