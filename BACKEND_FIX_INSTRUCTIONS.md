# Backend Fix Instructions (Google Apps Script)

There are two parts to this fix. You need to update your script to:
1.  **Recalculate Stats:** Fix the "Items" sheet when you delete bids manually.
2.  **Return Bids List:** Make the API actually send the list of recent bids to the website (so the "Recent Bids" section works).

## Part 1: The Maintenance Script

Add this function to the bottom of your script. Run it manually whenever you delete rows from the "bids" sheet.

**Important:** This assumes your column headers in the "bids" sheet are: `timestamp_iso`, `item_id`, `name`, `bid`, `phone`, `note`.

```javascript
function recalculateItemStats() {
  const { ss, items, bids } = getSheets_();

  // 1. Read Items
  const itemsTable = readTable_(items);
  const ic = itemsTable.colIndex;

  // 2. Read Bids
  const bidsByItem = {};
  if (bids.getLastRow() > 1) {
    const bidsTable = readTable_(bids);
    const bc = bidsTable.colIndex;

    bidsTable.rows.forEach(row => {
      const itemId = String(row[bc.item_id] || "").trim();
      const bidVal = Number(row[bc.bid]);
      const name   = String(row[bc.name] || "");
      const phone  = String(row[bc.phone] || "");
      const time   = String(row[bc.timestamp_iso] || "");

      if (!itemId) return;
      if (!bidsByItem[itemId]) bidsByItem[itemId] = [];

      bidsByItem[itemId].push({ name, bid: bidVal, phone, time });
    });
  }

  // 3. Update Items Sheet
  // We need to write back to columns: highest_bid, highest_name, highest_phone, last_update_iso
  // Note: Sheet column index = header index + 1

  itemsTable.rows.forEach((row, i) => {
    const rowNum = i + 2; // header is row 1
    const itemId = String(row[ic.item_id] || "").trim();
    const itemBids = bidsByItem[itemId] || [];

    if (itemBids.length === 0) {
      // Reset to 0
      items.getRange(rowNum, ic.highest_bid + 1).setValue(0);
      items.getRange(rowNum, ic.highest_name + 1).setValue("");
      items.getRange(rowNum, ic.highest_phone + 1).setValue("");
      items.getRange(rowNum, ic.last_update_iso + 1).setValue("");
    } else {
      // Find highest
      itemBids.sort((a, b) => b.bid - a.bid);
      const top = itemBids[0];

      items.getRange(rowNum, ic.highest_bid + 1).setValue(top.bid);
      items.getRange(rowNum, ic.highest_name + 1).setValue(top.name);
      items.getRange(rowNum, ic.highest_phone + 1).setValue(top.phone);
      items.getRange(rowNum, ic.last_update_iso + 1).setValue(top.time);
    }
  });

  Logger.log("Recalculated stats for all items.");
}
```

## Part 2: Update `doGet` to return bids

Your current `doGet` function does not return the list of bids. Update the `action === "get_state"` block in your `doGet` function to look like this:

```javascript
    if (action === "get_state") {
      const itemId = String(e?.parameter?.item_id || "").trim();
      if (!itemId) return jsonResponse({ ok: false, error: "missing_item_id" });

      const table = readTable_(items);
      requireCols_(table.colIndex, ["item_id"], SHEET_ITEMS);

      const idCol = table.colIndex["item_id"];
      let foundRow = null;
      for (const row of table.rows) {
        if (String(row[idCol]).trim() === itemId) {
          foundRow = row;
          break;
        }
      }

      if (!foundRow) return jsonResponse({ ok: false, error: "item_not_found" });

      const itemObj = normalizeItem_(rowToObj_(table.headers, foundRow));

      // --- NEW CODE STARTS HERE ---
      // Fetch bids for this item
      const { bids } = getSheets_();
      const itemBids = [];
      if (bids.getLastRow() > 1) {
        const bidsTable = readTable_(bids);
        const bc = bidsTable.colIndex;
        // Optimization: In a real DB we'd filter by query, here we scan.
        // Small scale is fine.
        bidsTable.rows.forEach(r => {
          if (String(r[bc.item_id]).trim() === itemId) {
            itemBids.push({
              name: r[bc.name],
              bid: r[bc.bid],
              time: r[bc.timestamp_iso]
            });
          }
        });
        // Sort newest first
        itemBids.sort((a, b) => {
            return (new Date(b.time || 0)) - (new Date(a.time || 0));
        });
      }
      itemObj.bids = itemBids;
      // --- NEW CODE ENDS HERE ---

      return jsonResponse({ ok: true, item: itemObj });
    }
```

**After updating the code, save and deploy a new version of your web app.**
