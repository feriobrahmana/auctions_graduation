# Graduation Auction (Static Site)

This folder contains a static auction website designed for GitHub Pages. It polls a Google Apps Script API for live updates and supports multiple items.

## GitHub Pages setup

1. Push this repo to GitHub.
2. Go to **Settings → Pages**.
3. Under **Build and deployment**, choose **Deploy from a branch**.
4. Select **main** and **/ (root)**.

The site will be available at `https://<your-username>.github.io/<repo-name>/auction/`.

## Google Sheets data

The Google Sheet must include an `items` tab with these columns:

```
item_id, title, description, min_increment, end_time_iso, status, highest_bid, highest_name, highest_phone, last_update_iso
```

- `status` is `OPEN` or `CLOSED`.
- When `status` is `CLOSED`, the site shows **SOLD** if `highest_bid > 0`, otherwise **CLOSED (No sale)**.

## Images per item

Place images in `auction/assets/<item_id>/` with file names `1.jpg` through `6.jpg`.

Example:

```
auction/assets/item1/1.jpg
auction/assets/item1/2.jpg
```

Only images that exist will be displayed on the item detail page.

## API endpoints

The site expects these API actions to be available:

- `action=list_items` (GET)
- `action=get_state` (GET)
- `action=place_bid` (POST)

If `list_items` does not exist yet, the site is still built to call it and will be ready once the endpoint is deployed.
