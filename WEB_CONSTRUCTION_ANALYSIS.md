# Web Construction Analysis

## 1. Architecture Overview
This is a **Static Web Application** hosted (presumably) on GitHub Pages. It relies entirely on client-side JavaScript to interact with a backend.
- **Frontend:** Plain HTML, CSS, and Vanilla JavaScript (no frameworks like React or Vue).
- **Backend:** A Google Apps Script deployment (`script.google.com`) acting as a serverless API and database interface (likely connecting to a Google Sheet).
- **Data Persistence:** Google Sheets (via the Apps Script API) and LocalStorage (for client-side caching).

## 2. User Journey & Implementation Details

### A. Landing Page (`index.html`)
**What the user sees first:**
1.  **Header:** A dark header with "Graduation Auction".
2.  **Loading/Cached State:**
    -   *If visited before:* The user immediately sees the list of items from their last visit (loaded from `localStorage`). A blue note appears: *"Showing cached items..."*.
    -   *First visit:* They see a "Loading items..." message.
3.  **Live Update:** After a brief moment (fetching data), the list updates with the latest bids/status. The note changes to *"Items are up to date"*.

**Technical Construction:**
-   **Structure:** A grid layout (`.items-grid`) using CSS Grid (`grid-template-columns: repeat(auto-fit, minmax(260px, 1fr))`) to be responsive on mobile and desktop.
-   **Data Fetching:**
    -   It calls `fetch(API_URL + "?action=list_items")`.
    -   It polls this endpoint every 3 seconds (`setInterval(fetchItems, 3000)`).
-   **Caching Strategy:**
    -   On successful fetch, data is saved to `localStorage` (`graduation_auction_items`).
    -   On load, it checks `localStorage` first to render content immediately (Optimistic UI).
-   **Status Indication:**
    -   Items have color-coded pills: Green (OPEN), Red (SOLD), Purple (CLOSED).
    -   Logic: `buildStatus` function checks `status` ("CLOSED" vs "OPEN") and `highest_bid` (>0 means SOLD).

### B. Item Detail Page (`item.html`)
**What the user sees:**
1.  **Navigation:** A "Back to items" link.
2.  **Item Details:** Title, Description, Status Pill, and Meta info (Current bid, End time).
3.  **Gallery:** A grid of photos.
4.  **Bidding Section (If OPEN):**
    -   A form to enter Name, Bid Amount, Phone, and Note.
    -   Helper text showing the *minimum required bid* (Current Bid + Increment).
    -   "Recent bids" list at the bottom.
5.  **Closed State (If CLOSED):**
    -   The bid form disappears.
    -   A notice appears showing the Winner and Winning Bid (if sold) or "No sale".

**Technical Construction:**
-   **Routing:** No real client-side router. It uses a query parameter: `item.html?item_id=123`.
-   **Image Loading:**
    -   It blindly attempts to load images `1.jpg` through `6.jpg` from `assets/<item_id>/`.
    -   It uses `img.onload` to count successful loads and `img.onerror` to remove broken links (since it doesn't know how many images exist beforehand).
-   **Real-time Updates:**
    -   Polls `action=get_state` every 3 seconds to update the bid history and highest bid live.
    -   Updates the `<title>` tag dynamically.
-   **Bidding Logic:**
    -   **Validation:** The input field dynamically updates `min` and `step` attributes based on the current highest bid.
    -   **Submission:** Sends a POST request to the API (`action=place_bid`).
    -   **Feedback:** Shows success ("Bid accepted") or specific error messages ("Bid too low", "Auction closed").

## 3. Key Findings regarding "Relevant Pages"
-   **Root vs. `auction/` folder:** The root directory contains the *production* code. The `auction/` folder appears to be an older or simpler version lacking features like caching, bid history lists, and dynamic form validation. **The root files are the ones that matter.**
-   **No Hidden Admin:** There are no admin pages found in the source. Admin actions (like closing auctions) are likely done directly in the Google Sheet.
