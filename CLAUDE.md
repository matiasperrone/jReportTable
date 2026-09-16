# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

jReportTable is a single-file jQuery plugin (`src/jquery.jReportTable.js`) that renders a data table with pagination, search filtering, sorting, and ajax loading. It is data-agnostic: it does not require column definitions and derives table headers from the shape of the first row of data it receives (from a preload array or from the ajax response).

There is no build system, package manager, or test suite in this repo — it's a plain jQuery plugin distributed as one `.js` file plus a `readme.md` and usage examples. Development is done by editing `src/jquery.jReportTable.js` directly and verifying behavior via the files in `examples/`.

## Running the example

`examples/example1.html` loads the plugin from `../src/` and wires it up against `examples/example1.php`, which returns hardcoded JSON pages (simulating an ajax backend). To try it, serve the `examples/` directory with a PHP-capable server (e.g. `php -S localhost:8000` from `examples/`) and open `example1.html` in a browser. There's no bundler step — edits to `src/jquery.jReportTable.js` are picked up on page reload.

## Architecture

The whole plugin lives in `src/jquery.jReportTable.js`, structured as a classic jQuery-boilerplate plugin:

- **Constructor (`jReportTable`)** — merges user `options` over `_defaults`, deep-merging the `selectors`, `pagination`, and `render` sub-objects specifically (a plain `$.extend` would clobber them). Builds `this.pagedata`, the mutable request/paging state (search term, current page `start`, `pagelength`, `total`, `sorting`/`sortorder`) that is sent as the ajax payload on every page request.
- **`init`** — wires the search input's `keyup` handler, builds the page-length `<select>`, and does the first data load: if `settings.data` was preloaded, it's rendered directly via `requestHandler`; otherwise it calls `requestPage(1)` to fetch via ajax.
- **`rtevents`** — DOM event handlers (search debounce, pagination link clicks, column header clicks for sorting, page-length `<select>` change). These update `this.pagedata` and then call `requestPage`.
- **`requestPage`** — issues the `$.ajax` call (or is bypassed on first load when static `data` is preloaded) using `this.pagedata` as the POST/GET payload, with `requestHandler` as the success callback.
- **`requestHandler`** — the core renderer. On the very first response it builds the `<th>` header row from the keys of the first data row (filtered by `hideColumn`/`showColumn`), then builds `<tr>`/`<td>` rows for every record, updates the "showing X to Y of Z" text, and calls `setPages`. Each `<td>` value passes through `transform`.
- **`transform`** — per-cell formatting hook: applies `$`-money or `%`-percent formatting for columns listed in `settings.types.money`/`types.percent`, otherwise delegates to the user-supplied `settings.render.value(key, value, row, tr)` callback.
- **`setPages` / `redrawPages` / `setPage`** — rebuild the pagination widget (prev/next + numbered links with `...` truncation) based on `pagedata.total` and `pagedata.pagelength`, then rebind click handlers.
- Registered as `$.fn.jReportTable` at the bottom, following the jQuery Boilerplate pattern (supports both `$(...).jReportTable(options)` to instantiate and `$(...).jReportTable('methodName', ...args)` to call an instance method, e.g. `.jReportTable('refresh', true)`).

### Key behavioral notes

- All server communication (initial ajax load, search, pagination, sorting, page-length changes) round-trips through `requestPage` → `$.ajax` → `requestHandler`, always sending the full `pagedata` object (search, start, pagelength, total, sorting, sortorder, refresh, plus `query`/`custom` if configured) as the request payload. The `examples/example1.php` backend shows the expected response shape: `{ data: [...], records: <total count> }`.
- Column visibility is controlled by `hideColumn` (exclude list) and `showColumn` (explicit include list, and also defines column order); these are evaluated on both the header-building and row-building code paths and must stay consistent between them.
- The plugin instance is stored on the element via `$.data(this, 'plugin_jReportTable')`, and only instantiates once per element — calling `.jReportTable(options)` again on an already-initialized element is a no-op for the constructor (only string-method calls act on the existing instance).
- Selector/pagination/render config objects are optional and merge on top of `_defaults` rather than replacing them, so partial overrides (e.g. only overriding `pagination.previousHTML`) work without breaking the rest.
