# Inventory Management DApp – Full Project Explanation

This DApp is a simple inventory tracker deployed on Ethereum-compatible networks. It lets an owner add, read, update, delete products, and view/export transaction history derived from smart contract events.

- Frontend stack: Vanilla HTML/CSS/JS + Web3.js (MetaMask provider)
- Smart contract: Solidity (0.8.x)
- Event-driven UI: Reads past events to render history and compute stats


## Project structure

- `index.html` – UI layout and wiring to JS
- `styles.css` – Minimal, high-contrast theme
- `abi.js` – Contract ABI used by Web3.js
- `app.js` – All frontend logic (wallet connect, CRUD, CSV import/export, history)
- `InventoryMgmt.sol` – Solidity contract implementing CRUD and events
- `README.md` – Repo placeholder


## How it works at a glance

1. Connect wallet via MetaMask, which injects `window.ethereum`.
2. Paste your deployed contract address and “Initialize Contract” to create a Web3 contract instance using `contractABI` from `abi.js`.
3. Perform CRUD operations (owner-only) that send transactions.
4. UI reads contract state via `view` methods and scans on-chain events (`ProductAdded`, `ProductUpdated`, `ProductDeleted`) to build history and compute stats.
5. Import products from a CSV or export products/history/summary to CSV files.


## UI: `index.html`

Key elements and IDs the JS relies on:
- MetaMask connect: button `connectWallet()` and account label `#account`
- Contract config: input `#contractAddress`, button `initContract()`, status `#configStatus`
- Import CSV: file input `#csvFile`, buttons `importProductsFromCSV() / confirmImportProducts() / cancelImport()`, preview `#importPreview`, status `#importStatus`
- Add product: inputs `#addName/#addQuantity/#addPrice`, button `addProduct()`, status `#addStatus`
- Read product: input `#readId`, button `getProduct()`, result area `#productDetails`, status `#readStatus`
- Update product: dropdown `#updateProductSelect` (populated via `populateProductDropdown()`), form with `#updateId/#updateName/#updateQuantity/#updatePrice`, button `updateProduct()`, status `#updateStatus`
- Delete product: input `#deleteId`, button `deleteProduct()`, status `#deleteStatus`
- All products: button `getAllProducts()`, list container `#allProductsList`, status `#allProductsStatus`
- History + stats: container `#historyList`, stats `#currentStockValue` and `#totalTransactions`, controls to refresh and export

Scripts loaded in order:
- Web3.js CDN
- `abi.js` to expose `window.contractABI`
- `app.js` containing all UI logic


## Styling: `styles.css`

- Clean monochrome UI with uppercase headers
- `.status` blocks for success/error feedback (toggles `.error`)
- `.product-item` for products, `.history-item` with color-coded left borders by action type (add/update/delete)
- Responsive grid for history stats


## ABI: `abi.js`

- Exposes `window.contractABI = [ ... ]` – auto-generated JSON ABI matching `InventoryManagement`:
  - Constructor
  - Events: `ProductAdded(id,name,quantity,price)`, `ProductUpdated(id,name,quantity,price)`, `ProductDeleted(id)`
  - Functions:
    - `addProduct(string,uint256,uint256)` (nonpayable)
    - `getProduct(uint256)` (view) returns `(uint256 id, string name, uint256 quantity, uint256 price, bool exists)`
    - `updateProduct(uint256,string,uint256,uint256)` (nonpayable)
    - `deleteProduct(uint256)` (nonpayable)
    - `getTotalProducts()` (view) returns `uint256`
    - Public getters for `owner`, `productCount`, and `products(uint256)`


## Frontend logic: `app.js` – function-by-function

Top-level state:
- `web3, contract, account` – Web3 provider, contract instance, current EOA
- `transactionHistory = []` – in-memory cache of parsed event history
- On DOMContentLoaded: restore `contractAddress` from `localStorage`

### Wallet & contract wiring

- `async function connectWallet()`
  - Ensures MetaMask is installed
  - Requests accounts via `eth_requestAccounts`
  - Saves `account` and constructs `web3 = new Web3(window.ethereum)`
  - Updates UI with short account string

- `function initContract()`
  - Reads address from `#contractAddress`
  - Requires wallet already connected
  - Instantiates `contract = new web3.eth.Contract(contractABI, address)`
  - Persists address in `localStorage`
  - Kicks off history fetch

- Helpers
  - `getValues(...ids)` – Reads `.value` from multiple inputs and returns array
  - `clearFields(...ids)` – Clears given inputs’ `.value`

### CSV import flow

- `async function importProductsFromCSV()`
  - Validates file chosen and contract initialized
  - Reads file using `FileReader`
  - Auto-detects delimiter (comma or tab)
  - Validates header contains Product Name, Quantity, Price (case-insensitive)
  - Parses rows, coercing `quantity`/`price` to positive integers; skips invalid lines
  - Shows a preview via `showImportPreview(products)` (no chain calls yet)

- `function showImportPreview(products)`
  - Renders a scrollable list with count and two buttons: confirm/cancel

- `async function confirmImportProducts()`
  - Re-parses the same CSV to reconstruct products array (keeps implementation independent)
  - Sends a transaction per product: `contract.methods.addProduct(...).send({ from: account })`
  - Throttles slightly between txs (`500ms` delay)
  - Reports success/failure counts, clears preview, refreshes on-chain history

- `function cancelImport()`
  - Clears file input + preview; emits status message

### CRUD actions

- `async function addProduct()`
  - Validates all fields present
  - Sends `addProduct(name, quantity, price)` from current account
  - Clears form and refreshes history after a small delay

- `async function getProduct()`
  - Calls `getProduct(id)` and renders a single product card (includes `exists` flag)

- `async function populateProductDropdown()`
  - Loads `getTotalProducts()` and iterates from `1..count`
  - Calls `getProduct(i)` and adds active items to `<select>` as options

- `async function loadProductForUpdate()`
  - Loads a product selected in the dropdown and prefills update form

- `async function updateProduct()`
  - Sends `updateProduct(id, name, quantity, price)` and refreshes history

- `async function deleteProduct()`
  - Sends `deleteProduct(id)` and refreshes history

- `async function getAllProducts()`
  - Iterates `1..getTotalProducts()` and collects active items
  - Renders them as a list of product cards

### Status helper

- `function showStatus(elementId, message, isError)`
  - Writes text, toggles `.error` class if needed, auto-hides after 5s

### Event-driven history and stats

- `async function fetchHistoryFromContract()`
  - Requires `contract` initialized
  - Fetches all past events from genesis: `ProductAdded`, `ProductUpdated`, `ProductDeleted`
  - Normalizes events to objects with: `type, productId, productName, quantity, price, totalValue, blockNumber, transactionHash`
  - Looks up the block for each event to resolve `timestamp`
  - Sorts by `blockNumber` descending, stores in `transactionHistory`
  - Calls `updateHistoryDisplay()` and `updateHistoryStats()`

- `function updateHistoryDisplay()`
  - Renders `transactionHistory` as stylized cards
  - Add: shows name/qty/price/total value
  - Update: shows ID/name/new qty/price/new value
  - Delete: shows ID and labels removed value as “Unknown” (no state at delete time in event)

- `async function updateHistoryStats()`
  - Sets total transaction count and invokes `calculateCurrentStockValue()`

- `async function calculateCurrentStockValue()`
  - Iterates active products and sums `quantity * price`, updates `#currentStockValue`

- `function clearHistory()`
  - Informs the user on-chain history is immutable

### CSV export utilities

- `function downloadCSV(csv, filename)`
  - Creates a Blob with UTF-8 BOM for correct Excel opening, triggers download

- `function escapeCSV(field)`
  - Properly quotes commas/quotes/newlines per CSV rules

- `async function exportProductsToCSV()`
  - Exports all active products with computed `Total Stock Value`

- `async function exportTransactionHistoryToCSV()`
  - Exports the in-memory `transactionHistory` array (requires `fetchHistoryFromContract()` run first)

- `async function exportAllDataToCSV()`
  - Creates a single CSV including a summary section, full product list, and full history


## Smart contract: `InventoryMgmt.sol` – line-by-line explanation

```solidity
// SPDX-License-Identifier: MIT
```
- License identifier, required by many tooling ecosystems.

```solidity
pragma solidity ^0.8.0;
```
- Compiler version pragma. Uses Solidity 0.8.x, which has built-in overflow checks.

```solidity
contract InventoryManagement {
```
- Defines the contract. All state and functions live within.

```solidity
    struct Product {
        uint id;
        string name;
        uint quantity;
        uint price;
        bool exists;
    }
```
- Structure to represent a product on-chain. Includes a boolean `exists` to mark “soft delete”.

```solidity
    mapping(uint => Product) public products;
```
- State mapping of product ID to Product. Public gives you an auto-generated getter `products(id)`.

```solidity
    uint public productCount;
```
- Tracks the last assigned product ID. Also acts as upper bound for iteration on the frontend.

```solidity
    address public owner;
```
- Stores deployer’s address as the contract owner.

```solidity
    event ProductAdded(uint id, string name, uint quantity, uint price);
    event ProductUpdated(uint id, string name, uint quantity, uint price);
    event ProductDeleted(uint id);
```
- Events emitted for off-chain indexing/UX. The frontend reads these to build history.

```solidity
    constructor() {
        owner = msg.sender;
        productCount = 0;
    }
```
- Constructor runs once at deployment. Sets the `owner` and initializes the counter.

```solidity
    modifier onlyOwner() {
        require(msg.sender == owner, "Only owner can perform this action");
        _;
    }
```
- Access control modifier gating state-changing methods to the owner only.

```solidity
    function addProduct(string memory _name, uint _quantity, uint _price) public onlyOwner {
        productCount++;
        products[productCount] = Product(productCount, _name, _quantity, _price, true);
        emit ProductAdded(productCount, _name, _quantity, _price);
    }
```
- CREATE: increments `productCount` to get a new ID, stores a `Product`, marks it as existing, and emits an event.

```solidity
    function getProduct(uint _id) public view returns (uint, string memory, uint, uint, bool) {
        require(products[_id].exists, "Product does not exist");
        Product memory p = products[_id];
        return (p.id, p.name, p.quantity, p.price, p.exists);
    }
```
- READ: reverts if product was never created or has been deleted; returns a tuple matching the ABI.

```solidity
    function updateProduct(uint _id, string memory _name, uint _quantity, uint _price) public onlyOwner {
        require(products[_id].exists, "Product does not exist");
        products[_id].name = _name;
        products[_id].quantity = _quantity;
        products[_id].price = _price;
        emit ProductUpdated(_id, _name, _quantity, _price);
    }
```
- UPDATE: requires existence; updates fields and emits an event capturing new values.

```solidity
    function deleteProduct(uint _id) public onlyOwner {
        require(products[_id].exists, "Product does not exist");
        products[_id].exists = false;
        emit ProductDeleted(_id);
    }
```
- DELETE: soft-deletes by toggling `exists` to false. Data remains stored, which helps preserve history but prevents re-use in `getProduct`.

```solidity
    function getTotalProducts() public view returns (uint) {
        return productCount;
    }
}
```
- Utility getter to return how many IDs have been used. Note this is the max ID, not the number of active products.


## Data contracts and shapes

- Product (Solidity tuple / JS array via `getProduct`): `[id, name, quantity, price, exists]`
- Event payloads (JS):
  - `ProductAdded`: `{ id, name, quantity, price }`
  - `ProductUpdated`: `{ id, name, quantity, price }`
  - `ProductDeleted`: `{ id }`
- Normalized history entry (JS): `{ type: 'add'|'update'|'delete', productId, productName?, quantity?, price?, totalValue?, blockNumber, transactionHash, timestamp }`


## Edge cases and notes

- Only the owner can add/update/delete. Use the deployer account in MetaMask.
- `getTotalProducts()` is not the count of active products; deleted IDs are skipped client-side.
- Deleting a product doesn’t remove `name/quantity/price`; it sets `exists=false`. The front-end still can’t call `getProduct` for deleted items (it would revert), so it ignores them.
- History is built from events. If events are pruned or node doesn’t serve `getPastEvents` deep history, history may be incomplete.
- CSV import sends one tx per product. This is simple but can be slow and costs gas per product. Batch functions could improve this.


## How to run locally

1. Deploy the contract (examples):
   - Hardhat/Anvil/Ganache local chain, or any testnet.
2. In MetaMask, select the same network as the deployment.
3. Open `index.html` in a browser with MetaMask (no dev server required).
4. Click “Connect MetaMask”.
5. Paste deployed contract address, click “Initialize Contract”.
6. Use the UI to add/update/delete/read products and view/export history.


## CSV formats

- Import header (case-insensitive, order flexible):
  - `Product Name`, `Quantity`, `Price per Unit`
- Export files created:
  - `inventory_products_YYYY-MM-DD.csv` – all active products
  - `inventory_history_YYYY-MM-DD.csv` – event history
  - `inventory_complete_YYYY-MM-DD.csv` – summary + products + history


## Possible improvements

- Add ownership transfer and pausing (OpenZeppelin Ownable/Pausable)
- Add input validation on-chain (e.g., non-empty name)
- Add batch `addProducts` to reduce gas/latency for imports
- Index events with `indexed` fields for better filtering/analytics
- Paginate `getPastEvents` for very large histories
- Store timestamps in events or maintain created/updated timestamps in storage
- Support decimals for prices (use `uint` with fixed scale, e.g., 2 decimals)


## Quick reference – JS API usage

- Send tx: `contract.methods.addProduct(name, qty, price).send({ from: account })`
- Read state: `contract.methods.getProduct(id).call()`
- Events: `contract.getPastEvents('ProductAdded', { fromBlock: 0, toBlock: 'latest' })`
- Get block timestamp: `web3.eth.getBlock(blockNumber)`


---

If you want, I can add diagrams, inline code comments, or a Hardhat script to deploy and verify the contract automatically.