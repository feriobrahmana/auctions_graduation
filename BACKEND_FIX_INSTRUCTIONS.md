# Backend Fix Instructions (Google Apps Script)

The issue you are seeing (stale data on the Landing Page) happens because the **"Items"** tab in your Google Sheet acts as a summary/cache. When a new bid comes in, the script updates that row. However, when you **manually delete** a bid from the "Bids" tab, the "Items" tab doesn't know about it and keeps showing the old numbers.

To fix this permanently, you can add a "Maintenance" script to your Google Apps Script project. You can run this function whenever you manually delete bids or edit data, and it will re-sync everything.

## Step 1: Add this function to your script

Open your Google Apps Script project (Extensions > Apps Script) and paste this code at the bottom:

```javascript
// Run this function manually to fix "Items" sheet when you delete bids
function recalculateItemStats() {
  const ss = SpreadsheetApp.getActiveSpreadsheet();
  const itemsSheet = ss.getSheetByName("items");
  const bidsSheet = ss.getSheetByName("bids");

  // 1. Read all Items
  // Assumes headers are in row 1. Data starts row 2.
  // Columns (0-indexed based on README):
  // 0: item_id, 6: highest_bid, 7: highest_name, 8: highest_phone, 9: last_update_iso
  const itemsRange = itemsSheet.getRange(2, 1, itemsSheet.getLastRow() - 1, itemsSheet.getLastColumn());
  const itemsValues = itemsRange.getValues();

  // 2. Read all Bids
  // Assumes columns: item_id, name, amount, phone, timestamp
  const bidsByItem = {};

  // Check if there are any bids (more than just the header row)
  if (bidsSheet.getLastRow() > 1) {
    const bidsRange = bidsSheet.getRange(2, 1, bidsSheet.getLastRow() - 1, bidsSheet.getLastColumn());
    const bidsValues = bidsRange.getValues();

    bidsValues.forEach(row => {
      const itemId = row[0];
      const name = row[1];
      const amount = row[2];
      const phone = row[3];
      const time = row[4];

      if (!itemId) return;

      if (!bidsByItem[itemId]) {
        bidsByItem[itemId] = [];
      }
      bidsByItem[itemId].push({ name, amount, phone, time });
    });
  }

  // 4. Update Items
  const updatedItems = itemsValues.map(itemRow => {
    const itemId = itemRow[0];
    const bids = bidsByItem[itemId] || [];

    if (bids.length === 0) {
      // No bids? Reset columns 6, 7, 8, 9
      itemRow[6] = 0;    // highest_bid
      itemRow[7] = "";   // highest_name
      itemRow[8] = "";   // highest_phone
      itemRow[9] = "";   // last_update_iso
    } else {
      // Find highest bid
      // Sort descending by amount
      bids.sort((a, b) => b.amount - a.amount);
      const topBid = bids[0];

      itemRow[6] = topBid.amount;
      itemRow[7] = topBid.name;
      itemRow[8] = topBid.phone;
      itemRow[9] = topBid.time; // Ensure this is ISO string if your frontend expects it
    }
    return itemRow;
  });

  // 5. Save back to sheet
  itemsRange.setValues(updatedItems);
  Logger.log("Recalculated stats for " + updatedItems.length + " items.");
}
```

## Step 2: How to use it
1.  Save the script.
2.  Select `recalculateItemStats` from the function dropdown in the toolbar.
3.  Click **Run**.
4.  Grant permissions if asked.

After running this, your `index.html` (Landing Page) will correctly show $0 and "—" for items with no bids, because the backend database (the Items sheet) is now clean.
