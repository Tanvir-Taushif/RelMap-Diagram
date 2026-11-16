# Relmap

**Turn a spreadsheet of tables and columns into an interactive ER diagram, right in your browser.**

Relmap reads an Excel or CSV file that lists your database tables (or views), their columns and data types. It works out how the tables join, then draws an entity-relationship diagram you can explore, edit and export.

It is a single HTML file with no backend, no build step and no database. Open it in a browser and it works.

---

## Table of contents

- [Why Relmap](#why-relmap)
- [Use cases](#use-cases)
- [Features](#features)
- [Quick start](#quick-start)
- [Input file format](#input-file-format)
- [How joins are detected](#how-joins-are-detected)
- [User manual](#user-manual)
  - [Moving around the canvas](#moving-around-the-canvas)
  - [Exploring a table’s connections](#exploring-a-tables-connections)
  - [Working with joins](#working-with-joins)
  - [Relationship types](#relationship-types)
  - [Editing tables and columns](#editing-tables-and-columns)
  - [Searching tables and columns](#searching-tables-and-columns)
  - [View options](#view-options)
  - [Importing and exporting](#importing-and-exporting)
  - [Undo, redo and saving](#undo-redo-and-saving)
- [Keyboard shortcuts](#keyboard-shortcuts)
- [Deployment](#deployment)
- [Tech stack](#tech-stack)
- [Browser support](#browser-support)
- [Limitations](#limitations)
- [License](#license)

---

## Why Relmap

Many databases, especially reporting and analytics databases built on **views**, have no foreign keys defined. Tools that draw ER diagrams from database constraints show those tables as unconnected boxes.

Relmap works from **metadata alone**: table name, column name and data type. A query against `information_schema` can produce that list in seconds. Relmap then infers the joins from naming conventions such as `CustomerId → Customers.Id` and lets you correct, add or remove joins by hand.

## Use cases

- **Documenting a view-based or reporting database** where foreign keys don’t exist.
- **Onboarding new developers or analysts**, so they can see how tables connect before writing queries.
- **Planning SQL joins**: click a table and see every table it joins to, and through which columns.
- **Data warehouse modelling**: lay out fact and dimension tables (star or snowflake schemas).
- **Schema reviews and audits**: spot unjoined tables, duplicate or versioned views (`_v1`, `_v2`, `_etl`) and inconsistent naming.
- **Designing a new schema**: start from a blank canvas and add tables, columns and joins by hand.
- **Sharing a diagram**: export it as an image, or as Excel or CSV that someone else can re-import.

## Features

- Import **Excel (.xlsx, .xls)** or **CSV/TSV** files, or drag and drop them onto the page
- **Automatic join detection** from column names, with weak matches marked for review
- **Three relationship types**: one to many, one to one, many to many, shown with crow’s-foot notation
- **Animated joins**: lines draw in, and moving dots show the direction of each join
- **Click a table** to highlight every table it connects to, up to 3 steps away
- **Add, remove and change joins** by dragging between columns or using a form
- **Add, rename and delete tables and columns**, change data types, set primary keys
- **Pattern search** across tables and columns: plain text, loose letters, wildcards and regular expressions
- **Zoom, pan, fit to screen** and an overview map
- **Auto layout** that groups connected tables together
- **Export** to Excel, CSV, an SVG image or a JSON project file
- **Undo and redo**, plus automatic saving in your browser
- Light and dark themes

---

## Quick start

### Option 1: open the file

1. Download `relmap.html` from this repository.
2. Double-click it to open it in Chrome, Edge, Firefox or Safari.
3. Go to **File → Import Excel or CSV…** and choose your metadata file.

> An internet connection is needed the first time for the Excel reader (SheetJS) and the font. Everything else runs locally.

### Option 2: GitHub Pages

1. Go to **Settings → Pages** in this repository.
2. Set the source to the main branch and root folder.
3. Open `https://<your-username>.github.io/<repo-name>/relmap.html`.

To serve it at the site root, rename `relmap.html` to `index.html`.

### Getting the metadata from your database

**PostgreSQL** (views and tables):

```sql
SELECT table_name  AS "View Name",
       column_name AS "Column Name",
       data_type   AS "Data Type"
FROM information_schema.columns
WHERE table_schema = 'public'
ORDER BY table_name, ordinal_position;
```

**SQL Server:**

```sql
SELECT TABLE_NAME  AS [View Name],
       COLUMN_NAME AS [Column Name],
       DATA_TYPE   AS [Data Type]
FROM INFORMATION_SCHEMA.COLUMNS
ORDER BY TABLE_NAME, ORDINAL_POSITION;
```

**MySQL:**

```sql
SELECT TABLE_NAME AS `View Name`,
       COLUMN_NAME AS `Column Name`,
       DATA_TYPE AS `Data Type`
FROM INFORMATION_SCHEMA.COLUMNS
WHERE TABLE_SCHEMA = 'your_database'
ORDER BY TABLE_NAME, ORDINAL_POSITION;
```

Export the result to Excel or CSV and import it into Relmap.

---

## Input file format

One row per column. The first row must be a header.

| View Name    | Column Name | Data Type |
|--------------|-------------|-----------|
| FactOrders   | order_id    | bigint    |
| FactOrders   | customer_id | bigint    |
| FactOrders   | order_date  | date      |
| DimCustomer  | customer_id | bigint    |
| DimCustomer  | full_name   | text      |

Relmap finds columns by their header names, so the order and exact wording don’t matter much:

| Purpose | Recognised headers (case-insensitive) | Required |
|---|---|---|
| Table name | contains `table`, `view`, `entity` or `object` | Yes |
| Column name | contains `column`, `field` or `attribute` | Yes |
| Data type | `Data Type`, `Type`, or contains `type` | Recommended |
| Primary key | `Is Primary Key`, `PK`, `Key` (values like `Yes`, `True`, `1`, `PK`) | Optional |
| Foreign key target table | `References Table`, `FK Table`, `Foreign Table` | Optional |
| Foreign key target column | `References Column`, `FK Column` | Optional |
| Relationship type | `Relationship` or `Cardinality` (`One to many`, `1:1`, `N:M`…) | Optional |

If no headers are recognised, the first three columns are read as table, column and type.

### Optional extra sheets (Excel only)

These are what Relmap writes when you export to Excel, so an exported file re-imports with all joins and positions kept.

- **Relationships**: `From Table`, `From Column`, `To Table`, `To Column`, `Relationship`, `Source`
- **Layout**: `Table`, `X`, `Y`

If the file contains joins (a Relationships sheet or References columns), those are used instead of automatic detection.

---

## How joins are detected

Relmap guesses joins from column names. It is right most of the time, but **review the results**, especially the dashed lines.

1. **Primary key of each table.** Relmap uses the first match from this list:
   - a column you have marked as the primary key
   - a column named `Id`
   - a column named after the table, such as `customer_id` in `DimCustomer` or `OrderId` in `Orders`
2. **Foreign key columns.** Any column ending in `Id` or `_id`, such as `ProductId`, `product_id` or `ProductVariantId`.
3. **Matching to a table.** The column name, minus `Id`, is compared with each table’s base name. To get the base name, Relmap:
   - removes common prefixes: `vw_`, `v_`, `tbl_`, `dim_`, `fact_`, `Dim`, `Fact`, `inventory_vw_inventory_`, `admin_vw_`, `logistics_`, `stg_`…
   - removes version suffixes: `_v1`, `_v2`, `_etl`, `_old`, `_new`, `_tmp`…
   - ignores plurals, so `Categories` matches `Category`
4. **Partial matches.** If the full name doesn’t match, Relmap tries dropping leading words, so `FromStoreId` matches `Stores`. These joins are drawn as **dashed lines** and marked as guesses.
5. **Choosing between versions.** When several tables share a base name (`vw_Orders`, `vw_Orders_v1`, `vw_Orders_v2`), Relmap prefers the one with the same prefix and version as the source table.
6. **Relationship type.** Detected joins are one to many. If a table’s primary key is itself a foreign key to another table, the join is one to one. Many to many is never detected automatically.

Columns like `CreatedBy`, `UpdatedBy` or `ApprovedBy` are **not** joined automatically. Add those joins by hand if you want them.

Click **Detect joins** at any time to run detection again. Joins you added by hand, or that came from the file, are kept.

---

## User manual

### Moving around the canvas

| Action | How |
|---|---|
| Pan | Drag an empty part of the canvas (or Alt+drag, or middle-click drag) |
| Zoom | Mouse wheel, trackpad pinch, or the **−** / **+** buttons at the bottom left |
| Fit everything | The fit button at the bottom left, or press **F** |
| Reset to 100% | Click the zoom percentage |
| Jump to an area | Click or drag on the overview map at the bottom right |
| Move a table | Drag it by its header or body |
| Rearrange everything | **Auto layout** in the toolbar |

On touch screens, drag with one finger to pan and pinch with two to zoom.

### Exploring a table’s connections

**Click any table.** Relmap then:

- highlights the table and every table joined to it, with a ripple animation
- dims everything else
- tints the columns used in each join
- adds moving dots along the highlighted joins to show direction
- lists all connected tables in the side panel, split into **This table points to** and **Points to this table**

In the side panel you can also:

- **Zoom to connections**: fit the table and its neighbours on screen.
- **Hide unrelated**: remove everything not connected.
- **Direct joins / Up to 2 steps / Up to 3 steps**: widen the highlight to tables further away.

Click an empty part of the canvas, or press **Esc**, to clear the selection. Hovering over a table without clicking also highlights its joins briefly.

### Working with joins

**Add a join by dragging**

1. Hover over a column. Small dots appear on its left and right edges.
2. Drag from a dot onto a column in another table.
3. Release. The join is created and its details open in the side panel.

If you drag from a primary key to a non-key column, Relmap flips the join so the key is on the “one” side.

**Add a join with the form**

1. Select a table.
2. In the side panel under **Add a join**, choose:
   - a column in this table
   - **points to** or **is pointed to by**
   - the other table and its column (its key is preselected)
   - the relationship type
3. Click **Add join**.

**Inspect, change or remove a join**

- Click a join line to select it. The side panel shows both ends, where the join came from (detected, guessed, imported or added by hand) and what it means.
- **Reverse direction** swaps which side is which.
- **Remove join** deletes it. You can also select it and press **Delete**, or click the **×** next to it in a table’s connected list.

### Relationship types

| Type | Line ends | Meaning |
|---|---|---|
| **One to many** (1 : N) | Crow’s foot on one end, double bar on the other | Many rows on the crow’s-foot side point to one row on the other side. This is the usual foreign key. |
| **One to one** (1 : 1) | Double bar on both ends | Each row matches at most one row on the other side. |
| **Many to many** (N : M) | Crow’s foot on both ends | Rows on both sides can relate to many rows on the other. Usually stored through a link table. |

To change a type, click the join line and choose one of the three options in the side panel. Highlighted joins show a small **1 : N**, **1 : 1** or **N : M** label at their midpoint. You can show labels on every join, or turn them off, under **View → Relationship labels**.

### Editing tables and columns

| Task | How |
|---|---|
| Add a table | **Add table** in the toolbar. It appears in an empty spot near the centre of the screen with an `id` column. |
| Rename a table | Select it and edit the name at the top of the side panel, or double-click its header. |
| Delete a table | Select it and click **Delete table**, or press **Delete**. Its joins are removed too. |
| Add a column | Select the table and click **Add column**, then type the name. |
| Rename a column or change its type | Edit the name or type fields in the **Columns** list. The type field suggests common types. |
| Remove a column | Click the **×** next to it. Joins using it are removed. |
| Set the primary key | Click the key icon next to a column. Click it again to unset it. |

Table and column names must be unique: two tables can’t share a name, and neither can two columns in one table.

### Searching tables and columns

The search box in the toolbar (**Ctrl+F**) searches every table name and every column name at once, and suggests results as you type.

Results appear in two groups, **Tables** and **Columns**, with tabs to show only one group. Matching letters are highlighted, and each column result shows which table it belongs to and its data type.

**Search patterns**

| You type | It finds | Example match |
|---|---|---|
| `order` | names containing the text; exact and starts-with matches rank first | `OrderId`, `vw_Orders`, `CustomerOrderItems` |
| `ordid` | loose letters, where each letter continues a word or starts a new one | `OrderId` |
| `*_id` | wildcard: `*` is any text, `?` is any one character; must match the whole name | `order_id`, `customer_id` |
| `vw_Order?` | wildcard | `vw_Orders` |
| `/^is[A-Z]/` | a regular expression (case-insensitive) | `IsAdmin`, `IsManager` |

**Using the results**

- **Click a table** (or press **Enter** on it) to select it and zoom to it.
- **Click a column** to zoom to that exact row. It flashes so you can spot it, and the table expands if the column was hidden.
- **Highlight on canvas** (or **Ctrl+Enter**) dims everything except the tables with matches and shows the matched columns. Click **Clear** at the top of the canvas, or press **Esc**, to return to normal.
- **↑ / ↓** move through suggestions and **Esc** closes them.

The **Filter tables** box in the side panel accepts the same patterns.

### View options

Open the **View** menu in the toolbar.

| Setting | Options |
|---|---|
| Columns shown | **All columns**, **Keys and joined columns** (the default for large schemas), **Table names only**. In the shorter modes, click “N more columns” at the bottom of a table to expand just that one. |
| Relationship labels | On highlighted joins, on every join, off |
| Lines | Right-angled or curved |
| Flow animation | On every join, only on highlighted joins, or off |
| Flow direction | Foreign key to key, or key to foreign key |
| Theme | Match system, light, dark |

When zoomed far out, column rows are hidden automatically to keep the diagram fast.

### Importing and exporting

Everything is in the **File** menu.

**Open**

- **Import Excel or CSV…** (Ctrl+O). You can also drag a file onto the page. Importing replaces the current diagram, and you can undo it.
- **Load star schema example** opens a small demo diagram.
- **Start with a blank canvas** clears the diagram so you can design from scratch.

**Export**

| Format | Contains | Re-importable |
|---|---|---|
| **Excel** | `Columns`, `Relationships` and `Layout` sheets | Yes, with joins, types and positions |
| **CSV** | One row per column, with `Is Primary Key`, `References Table`, `References Column` and `Relationship` | Yes, with joins and types |
| **SVG image** | The whole diagram as a vector image, for documents, wikis and slides | No |
| **Relmap project (JSON)** | Everything, including positions | Yes |

### Undo, redo and saving

- Every change can be undone with **Ctrl+Z** and redone with **Ctrl+Shift+Z** or **Ctrl+Y**, or with the arrow buttons in the toolbar.
- Deleting a table, column or join also shows an **Undo** link in the message at the bottom of the screen.
- Your diagram, layout and view settings **save automatically in your browser** (localStorage). They stay on that computer and in that browser only.
- To move a diagram to another computer or share it, export it (Excel or JSON) and import it there.

---

## Keyboard shortcuts

| Shortcut | Action |
|---|---|
| `Ctrl/⌘ + F` | Search tables and columns |
| `Ctrl/⌘ + O` | Import a file |
| `Ctrl/⌘ + Z` | Undo |
| `Ctrl/⌘ + Shift + Z` or `Ctrl/⌘ + Y` | Redo |
| `F` | Fit the whole diagram on screen |
| `+` / `-` | Zoom in or out |
| `Delete` / `Backspace` | Delete the selected join or table |
| `Esc` | Clear the selection, search highlight or menu |
| `Enter` in search | Open the highlighted result |
| `Ctrl/⌘ + Enter` in search | Highlight all matches on the canvas |
| `Alt + drag` or middle-click drag | Pan, even over a table |
| Double-click a table header | Rename the table |

---

## Deployment

Relmap is one static file, so it can be hosted anywhere that serves HTML:

- GitHub Pages, GitLab Pages, Netlify, Vercel
- IIS, Nginx, Apache
- SharePoint document libraries, Amazon S3 or Azure Blob static websites
- Or open it straight from disk

No server-side code, database or API keys are needed. Your schema data never leaves the browser.

### Running fully offline

The page loads two resources from the internet:

- [SheetJS](https://sheetjs.com/) `xlsx.full.min.js` from cdnjs, for reading and writing Excel files
- the *Instrument Sans* font from Google Fonts, which falls back to your system font if unavailable

To run with no internet access:

1. Download `xlsx.full.min.js` (version 0.18.5) and place it next to `relmap.html`.
2. In `relmap.html`, change the script tag to:

```html
<script src="xlsx.full.min.js"></script>
```

### Embedded sample data

The file includes a built-in dataset (`USER_DATA` near the top of the `<script>` tag) that loads on first open. Before publishing to a public repository, consider replacing it with `null` if the schema is confidential:

```js
const USER_DATA = null;
```

When it’s `null`, the app opens on an empty canvas with import buttons.

---

## Tech stack

- **HTML, CSS and vanilla JavaScript**, in a single file with no framework and no build step
- **SVG** for joins, markers and animations
- **SheetJS** for Excel and CSV parsing and export
- **localStorage** for automatic saving

### Project structure

```
.
├── relmap.html   # the whole app: markup, styles and script
└── README.md
```

### Main parts of the script

| Area | What it does |
|---|---|
| Naming and join inference | `tableInfo`, `findPK`, `inferRelationships` |
| Rendering | `render`, `renderTables`, `renderLinks`, `applyClasses` |
| Link routing | `relPath`, `pathBetween`, `rounded` |
| Layout | `autoLayout`, a force-directed layout per connected group with overlap removal and packing |
| Search | `makeMatcher`, `runSearch`, `focusColumn`, `highlightMatches` |
| Import and export | `parseWorkbook`, `exportXLSX`, `exportCSV`, `exportSVG`, `exportJSON` |
| History and saving | `snapshot`, `undo`, `redo`, `saveLater` |

---

## Browser support

Recent versions of Chrome, Edge, Firefox and Safari on desktop. Tablets and phones work, with the side panel shown as a bottom sheet. Animations are turned off automatically if your system has **reduce motion** enabled.

## Limitations

- Joins are guessed from naming conventions. Tables with unconventional column names need joins added by hand.
- Many-to-many joins and audit columns (`CreatedBy`, `UpdatedBy`) are not detected automatically.
- Saved work lives in one browser on one computer. Export to keep a copy or share it.
- Very large schemas (thousands of tables) will be slower. Use **Keys and joined columns** or **Table names only** mode.
- Composite (multi-column) keys are not modelled. Each join links one column to one column.

## License

Add your license here, for example MIT.
