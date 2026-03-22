---
name: superme
description: "Online supermarket automation. Login, search products, add to wishlists, magicorder from past orders. Currently supports Shufersal. Usage: /superme login, /superme search <product>, /superme add <#>, /superme magicorder, /superme lists, /superme help"
risk: medium
source: local
date_added: "2026-03-21"
---

# /superme — Online Supermarket Automation

Automate Israeli online supermarkets using browser-use CLI. Currently supports **Shufersal** (`shufersal.co.il`).

---

## Prerequisites (auto-install)

**IMPORTANT:** Before executing ANY command below, always run this first to ensure browser-use is installed:
```bash
which browser-use 2>/dev/null || uv tool install browser-use --python 3.12
```
This is a no-op if already installed, and auto-installs if missing. Run it silently at the start of every `/superme` invocation.

---

## Session State

The skill tracks state across commands within a conversation using temp files:
- `/tmp/superme_cookies.json` — exported browser cookies
- `/tmp/superme_cookie_str.txt` — cookie string for curl API calls
- `/tmp/superme_session_list_id.txt` — current session's wishlist ID (created on first add)
- `/tmp/superme_last_search.json` — last search results with product codes for `/superme add`

---

## Commands

### `/superme help`

Show available commands:
```
/superme login <email> <password>    Log in to Shufersal Online
/superme search <product>            Search for a product (Hebrew or English)
/superme add <#>                     Add search result # to your session wishlist
/superme magicorder                  Consolidate last 5 orders into one wishlist
/superme lists                       View your Superme wishlists
/superme close                       Close the browser session
```

---

### `/superme login`

Log in to Shufersal Online.

**Steps:**

1. Ask the user for credentials if not provided inline. Parse from args if given as `/superme login user@email.com password123`.

2. Open login page:
```bash
browser-use --headed --session superme open "https://www.shufersal.co.il/online/he/login"
```

3. Wait for load, then fill credentials via JavaScript (Shadow DOM inputs):
```bash
browser-use --session superme eval "document.getElementById('j_username').value = 'EMAIL'; document.getElementById('j_username').dispatchEvent(new Event('input', {bubbles: true}));"
browser-use --session superme eval "document.getElementById('j_password').value = 'PASS'; document.getElementById('j_password').dispatchEvent(new Event('input', {bubbles: true}));"
```

4. Get state, find submit button (`<button type=submit />` inside `<form id=loginForm />`), click it.

5. Wait 5 seconds, verify: `window.location.href` should NOT contain `/login`.

6. After login, export cookies for API calls:
```bash
browser-use --session superme cookies export /tmp/superme_cookies.json
```
Then build the cookie string:
```python
import json
with open('/tmp/superme_cookies.json') as f:
    cookies = json.load(f)
shufersal = [c for c in cookies if 'shufersal' in c.get('domain','')]
cookie_str = '; '.join(f"{c['name']}={c['value']}" for c in shufersal)
with open('/tmp/superme_cookie_str.txt', 'w') as f:
    f.write(cookie_str)
```

**IMPORTANT:** Never store or log the user's password. Only use it in the eval command.

---

### `/superme search <product>`

Search for a product on Shufersal. Must be logged in first.

**Steps:**

1. Check session is active and on shufersal.co.il. If not, navigate to homepage first.

2. Use JavaScript to fill search input and submit (handles Hebrew correctly):
```bash
browser-use --session superme eval "var input = document.getElementById('js-site-search-input'); input.value = 'PRODUCT_HERE'; input.dispatchEvent(new Event('input', {bubbles: true}));"
browser-use --session superme eval "document.querySelector('button[type=submit][aria-label]').click();"
```

3. Wait 5 seconds, get page state.

4. Parse products from state. Each product appears as `<li level=1>` with:
   - `<strong>` — product name
   - Text after — size/weight
   - Next line — brand
   - `<span>` with price
   - Sale prices have "מחיר קודם" (previous price)
   - Product codes appear in `<span id=P_XXXXX>` elements

5. **Save search results** to `/tmp/superme_last_search.json` with product codes, names, prices for use by `/superme add`. Format:
```json
[{"index": 1, "name": "Product Name", "code": "P_XXXXX", "price": "19.90", "size": "400g"}, ...]
```

6. Present results using **English only** (terminal can't render Hebrew/RTL). Use transliterated Hebrew names. Columns: #, Product Name, Size, Price, Per 100g, Notes.

7. Filter and prioritize exact matches over loosely related items.

8. Recommend the best value option and tell the user: "Use `/superme add #` to add to your list."

---

### `/superme add <#>`

Add a product from the last search results to the session wishlist. If no wishlist exists for this session, create one first.

**Steps:**

1. Load `/tmp/superme_last_search.json` to get the product by index number.

2. Check if a session wishlist exists by reading `/tmp/superme_session_list_id.txt`:
   - **If file exists**: use the stored list ID
   - **If not**: create a new wishlist (see "Creating a session wishlist" below)

3. Navigate to the wishlist edit page:
```bash
browser-use --session superme open "https://www.shufersal.co.il/online/he/my-account/personal-area/wish-list-2-cart/LIST_ID"
```

4. Wait for load, then add the product via the Vue component:
```javascript
var comp = document.querySelector('#personalAreaContainer').__vue__;
function findComp(vm) {
  if(vm.submitWishList) return vm;
  for(var c of (vm.$children || [])) { var f = findComp(c); if(f) return f; }
  return null;
}
var wl = findComp(comp);

wl.createNewProductSearch({
  code: 'P_XXXXX',
  name: 'Product Name',
  price: {value: 1},
  stock: {stockLevelStatus: {code: 'inStock'}},
  sellingMethod: 'byUnit'
}, 1, 'byUnit');

wl.submitWishList();
```

5. Wait 5 seconds, then verify by checking entries count:
```javascript
store.dispatch('loadWishList', 'LIST_ID').then(function() {
  var entries = store.state.wishList.selectedWishList.entries;
  // entries.length should have increased
});
```

6. If a WAF error dialog appears ("תווים לא חוקיים"), close it — the initial save usually succeeds despite the dialog.

7. Confirm to user: "Added [product name] to your Superme List. [N] items total."

#### Creating a session wishlist

When no session list exists yet:

```javascript
// Must be on a Shufersal page. Use $.ajax (jQuery is loaded on the site):
$.ajax({
  url: ACC.config.encodedContextPath + '/wishlist/createWishlist',
  type: 'POST',
  data: JSON.stringify({name: 'SupermeListDDMMYYYY'}),
  contentType: 'application/json',
  success: function(d) { /* d.shoppingListId is the new list ID */ }
});
```

**IMPORTANT:**
- List name must NOT contain `/` or special characters (causes 500 error). Use format: "Superme List DD-MM-YYYY"
- Save the returned `shoppingListId` to `/tmp/superme_session_list_id.txt`
- The `$.ajax` call must be made from a browser eval on a Shufersal page (CSRF protection)

---

### `/superme magicorder`

Consolidate the last 5 orders into a single wishlist with all unique products.

**Steps:**

1. Export cookies (if not already done):
```bash
browser-use --session superme cookies export /tmp/superme_cookies.json
```
Build cookie string (same as login step 6).

2. Fetch orders list via API:
```bash
curl -s 'https://www.shufersal.co.il/online/he/my-account/orders' \
  -H 'accept: */*' -H 'content-type: application/json' \
  -H 'x-requested-with: XMLHttpRequest' \
  -b "$(cat /tmp/superme_cookie_str.txt)"
```
Response: `closedOrders[]` array. Take first 5 order codes.

3. Fetch each order's details:
```bash
curl -s "https://www.shufersal.co.il/online/he/my-account/orders/ORDER_CODE" \
  -H 'accept: */*' -H 'content-type: application/json' \
  -H 'x-requested-with: XMLHttpRequest' \
  -b "$(cat /tmp/superme_cookie_str.txt)"
```
Response: `entries[]` with `product.name`, `product.code`, `product.price.formattedValue`.

4. Consolidate products, deduplicate by code. Exclude delivery items ("משלוח", "דמי").

5. Present to user sorted by frequency (most ordered first), English transliteration.

6. Create empty wishlist via `$.ajax` (see "Creating a session wishlist" above). Name: "Superme Magicorder DD-MM-YYYY".

7. Navigate to wishlist edit page, add all products via Vue component (same as `/superme add` step 4, but loop over all products and call `submitWishList()` once at the end).

8. Verify server-side count matches. Close any WAF error dialogs.

9. Give user the link: `https://www.shufersal.co.il/online/he/wish-lists/main`

---

### `/superme lists`

View all Superme wishlists.

**Steps:**

1. Get wishlists via browser API:
```javascript
fetch(ACC.config.encodedContextPath + '/wishlist/getUserWishlists', {
  credentials: 'include',
  headers: {'X-Requested-With': 'XMLHttpRequest'}
}).then(r => r.json())
```

2. Filter for lists whose name starts with "Superme".

3. Present in a table: List Name, Items Count, Date.

4. If no Superme lists found, suggest `/superme search` to start.

5. Link: `https://www.shufersal.co.il/online/he/wish-lists/main`

---

### `/superme close`

```bash
browser-use --session superme close
```
Also clean up temp files: `/tmp/superme_session_list_id.txt`, `/tmp/superme_last_search.json`

---

## Key Technical Details

### API Endpoints (require session cookies)

| Endpoint | Method | Notes |
|----------|--------|-------|
| `/online/he/my-account/orders` | GET | List all orders. Returns `{closedOrders: [...]}` |
| `/online/he/my-account/orders/CODE` | GET | Order details. Returns `{entries: [...]}` |
| `/wishlist/getUserWishlists` | GET | All user wishlists. Via browser fetch only. |
| `/wishlist/createWishlist` | POST | Create empty list. Body: `{name: "..."}`. Via `$.ajax`. |
| `/wishlist/editWishlist` | POST | Edit list (add/remove products). Via Vue store dispatch. |
| `/wishlist/getWishlist?listId=ID` | GET | Get wishlist details with entries. |

### Cookie Management

- **For curl API calls** (orders): Export cookies via `browser-use cookies export`, build cookie string including httpOnly cookies (JSESSIONID, XSRF-TOKEN).
- **For browser-side API calls** (wishlists): Use `$.ajax` or `fetch` from browser eval — cookies are automatically included.
- Cookies from `document.cookie` are incomplete (missing httpOnly). Always use `cookies export`.

### Vue Component Access

The wishlist edit page uses Vue.js. Access the component tree:
```javascript
var comp = document.querySelector('#personalAreaContainer').__vue__;
function findComp(vm) {
  if(vm.submitWishList) return vm;
  for(var c of (vm.$children || [])) { var f = findComp(c); if(f) return f; }
  return null;
}
var wl = findComp(comp);
```

Key methods:
- `wl.createNewProductSearch(product, qty, sellingMethod)` — add product to local state
- `wl.submitWishList()` — save all changes to server
- `wl.changesArr` / `wl.productArr` / `wl.entries` — state arrays

### Known Issues

- **WAF blocks special characters**: Product names with `'` (e.g., `תפוצ'יפס`) trigger "תווים לא חוקיים" (426) on the auto-save retry. The initial save usually succeeds. Close the error dialog and verify server-side.
- **List names**: Cannot contain `/`. Use hyphens for dates: DD-MM-YYYY.
- **Hebrew in CLI args**: `browser-use type` doesn't handle Hebrew. Use `eval` with `document.getElementById().value = '...'` instead.
- **Cart doesn't persist**: Items added to cart via browser automation are lost when the session closes. Always use wishlists.

---

## Session Management

- All commands use `--session superme` for persistent browser session
- Always use `--headed` on initial `open` so user can see the browser
- Login state persists within the session but is lost on `close`

## Supported Vendors

| Vendor | Domain | Status |
|--------|--------|--------|
| Shufersal | shufersal.co.il | **Supported** |
| Rami Levy | rami-levy.co.il | Planned |
| Yochananof | yochananof.co.il | Planned |
| Victory | victory-online.co.il | Planned |

## Error Handling

- Login fails → check credentials, try again
- Session lost → run `/superme login` again
- WAF error on save → close dialog, verify server-side count
- 500 on create wishlist → check name for special characters
- browser-use not installed → `uv tool install browser-use --python 3.12`
