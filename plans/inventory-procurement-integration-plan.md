# Inventory, Assets & Procurement: integration and implementation plan (v1)

## Context
This plan combines two inputs:

1. **`procurement-assets-integration.md` (rev 4).** The reviewed design for the seam between the `procurement` app (PR → PO → GRN → QC) and the existing Assets module. Decisions D1–D19.
2. **Inventory & Asset Management Report (Oct 2026), Part 4: gap analysis of `assets-erd_1`.** It lists 17 missing entities. Six are P1 ("build first"): Location/Bin, StockBalance, StockTransfer, Party, StockCount/Audit and Lot/Batch. It also proposes field changes to existing entities.

The two overlap in one place: **the stock ledger**. Procurement writes `receive` movements into `StockMovement`, and every P1 gap in the report is solved by making that same ledger location- and lot-aware. So this plan settles the order of work and the rules that keep the GRN path, the new inventory documents and the Assets module consistent.

Rev 4 is not reopened. D1–D19 stand as written. New decisions start at **D20**.

> **Not checked against code.** The `esysflow` / `django-backend` / `esysflow_ui` repositories are not available in this session. File and line references are carried over from rev 4 and have not been re-verified. Check each phase against `feature/procurement-module` before starting it.

## Target architecture

```
        PROCUREMENT (procurement app)          INVENTORY DOCUMENTS (inventory app)
   PR → PO → GRN → QC ──► Post to Assets         StockTransfer        StockCount
                │      (inventory.receive)       (inventory.transfer) (inventory.count)
                │                                       │                    │
                └──────── assets_bridge ────────┬───────┴────────────────────┘
                                                ▼
                          ASSETS (assets app): the only writer of stock
          ┌──────────────────────────────────────────────────────────────┐
          │ Asset / Asset Part · AssetType · Lot · Location (tree)       │
          │                                                              │
          │ apply_movement() ──► StockMovement (append-only ledger:      │
          │                      location, lot, source_kind/source_id)  │
          │                  └─► StockBalance (asset × location × lot)   │
          │                      Asset.quantity_on_hand = cached sum     │
          └──────────────────────────────────────────────────────────────┘
                                                ▲
              Party: existing BusinessPartner (vendor/manufacturer/customer)
```

**Layering stays `procurement → inventory → assets`.** Assets never imports `inventory` or `procurement`:
- `Location`, `Lot` and `StockBalance` live in **assets**, because `StockMovement` (assets) has FKs to them and `apply_movement` (assets) maintains the balance.
- `StockTransfer` and `StockCount` are documents. They live in **inventory** and write only through `apply_movement`, tagged with `source_kind`/`source_id`, the same pattern rev 4 uses for GRN lines.
- `BusinessPartner` must sit in an app **below** assets (see D20).

## Phase plan

| Phase | Scope | Depends on | Size |
|---|---|---|---|
| **0** | Doc housekeeping from rev 4 (v4 → v5, decision log, snapshot) | — | S |
| **1** | Procurement ↔ Assets seam, exactly as rev 4 (§1–§8) + Party consolidation (D20) | 0 | L |
| **2** | Location-aware ledger: Location tree, StockMovement.location/lot, StockBalance, backfill | 1 | L |
| **3** | Lot/Batch tracking and bin-level receipt in GRN/QC | 2 | M |
| **4** | Inventory documents: StockTransfer, StockCount/Audit | 2 (3 for lots) | L |
| **5** | P2 entities, in procurement-impact order | 1–4 | XL, split |
| **6** | P3 (Meter, Licence) and "Raise PR" loop-backs | 5 | M |

Why the seam ships before locations: rev 4 is fully reviewed and doesn't need locations. Every GRN movement it writes carries `source_kind="grn_line"`, so Phase 2 can backfill those rows **exactly** onto the GRN's site (D24). Building locations first would delay procurement and give no better data.

---

## Phase 0: housekeeping
Exactly as rev 4 "Doc housekeeping":
- save rev 4 to `esysflow/.claude/plans/procurement-assets-integration.md`;
- edit v4 in place into v5 (§2, §3, §4, §5, §6, history), add the D1–D19 decision log, save a `-v5.md` snapshot.

Add to this repo/plan folder: this document as the umbrella plan. Its D20+ log moves into v5 once Phases 1–2 are approved.

## Phase 1: the procurement seam (rev 4) + Party
**Build rev 4 as written.** Order of commits:
1. Assets prep: `StockMovement.source_kind`/`source_id` migration + index; visibility fixes for `CheckoutDialog`, the procurement line picker and the MRS rollups.
2. Separate commit (D15): `preview_build` warns with `component_inactive`, `execute_build` refuses. Called out in the commit message.
3. `access/registry.py`: public `controller_grant_*` adapters (D16). `procurement.reverse` in `VALID_CAPABILITIES`. `procurement/apps.py::ready()` registration, idempotent (D12).
4. Models: `POLine.tracking_mode_snapshot`, `GRNLine.posted_purchase_value`, `GRNLine.posted_asset_type`, the close-line fields (`cancelled_qty`, reason, user, time, `reprocure`).
5. `procurement/assets_bridge.py`: `resolve_line_type`, `prevalidate`, `post_line_single`, `post_line_quantity`, `reverse_line`.
6. Services and API: `reresolve-line-type`, `close-line`, `reprocure` on `close_po`.
7. Read-back: `GET procurement/asset-history/`.
8. UI: Procurement panel, "From GRN-…" movement links, drawer link, QC tag warning, `visible`-aware post result, close dialog with "Re-procure?" (default No, hidden on direct POs).

**Party consolidation (D20), added to Phase 1:**
- The report's gap #4 ("Party") is already half-built: procurement uses `BusinessPartner` as the vendor. Do **not** add a second Party table.
- Confirm where `BusinessPartner` lives. If it is in `procurement`, move it to a shared app below assets (e.g. `partners` or `core`) **before** assets references it. The move is a model-state-only migration (`SeparateDatabaseAndState`) so the table is not rebuilt.
- Add a `kind` set to it if missing: `vendor`, `manufacturer`, `customer`, `service_provider` (one partner can hold several).
- New `Asset.vendor` FK (nullable). The GRN post sets it from the PO vendor, as one more line in the rev 4 §3 field mapping.
- `AssetInvoice.vendor` free text → new `vendor` FK + keep the text as `vendor_text` (legacy). The UI's "manual (legacy)" label from rev 4 §6 then keys off `vendor IS NULL AND vendor_text != ''`.
- Data migration: match existing `vendor_text` to partners by normalised name (case/whitespace/GSTIN if present). Report unmatched rows. Never auto-create partners.

**Exit criteria:** every rev 4 verification item passes (`makemigrations --check`, `manage.py test procurement assets inventory access`, `ruff check`, `tsc --noEmit`, `vitest run`, the manual end-to-end flow), plus: GRN-created assets carry `vendor` = PO vendor, and assets still does not import procurement.

## Phase 2: location-aware ledger (P1 gaps 1, 2, 6-prep)
### Models (assets app)
- **`Location`** (D21): `org`, `site` (FK, required), `parent` (self FK), `name`, `code`, `path` (materialised, e.g. `WH1/Rack-A/Bin-3`), `usage` (`internal` | `transit` | `scrap` | `adjustment`), `is_active`. Unique `(site, path)`.
  - Each active Site gets a root internal location called **Default**, created by a data migration and by a `post_save` on new sites.
  - One org-level `transit` location (used by transfers in Phase 4).
- **`StockMovement`** gains `location` (FK, nullable in this migration) and `lot` (FK to `Lot`, nullable). `kind` gains `transfer_out`, `transfer_in`, `return` (D22).
- **`StockBalance`** (D23): `org`, `asset`, `location`, `lot` (nullable), `quantity`. Unique `(asset, location, lot)` with NULLs not distinct (PostgreSQL 15+; otherwise a partial unique index pair).
- **`Lot`** is created here (empty until Phase 3) so the FK exists in one migration.
- `Asset.location` (free-text bin) → new `Asset.location_ref` FK (single-tracked assets only). The string is kept as `location_note` until the backfill is accepted, then dropped in a later release.

### Service changes
- `apply_movement` takes `location` (required for quantity items once backfill is done) and `lot`. In the same transaction it does `select_for_update` on the `StockBalance` row, applies the signed quantity and refuses to go below zero unless the movement is an `adjust` flagged `allow_negative` (counts only).
- `Asset.quantity_on_hand` stays, as a cache = Σ `StockBalance.quantity`. Valuation (`current_value = on_hand × unit_cost`) is unchanged, so D1 is unaffected (D31).
- New `rebuild_stock_balance(org=None, asset=None)` service + management command. It rebuilds balances from the ledger and reports drift.
- `visible_assets_for` is unchanged. Balance reads go through the asset, so visibility follows automatically.

### Backfill (D24)
Backfill runs as a management command with `--dry-run`, not inside the schema migration, because it is large and must be re-runnable:
1. Movements with `source_kind="grn_line"` → the GRN's site → that site's Default location. This step lives in the **procurement** app (procurement may read assets; not the other way).
2. Other movements (opening balance, manual receive, adjust, build, assignment) → `asset.current_site` → Default.
3. Assets with no site → an org-level **Unassigned** site + Default location, flagged on the asset list as needing a site.
4. Rebuild `StockBalance`. Assert Σ balance = `quantity_on_hand` for every asset; list mismatches instead of failing silently.
5. Then a follow-up migration makes `StockMovement.location` NOT NULL for quantity items (check constraint keyed on a denormalised `tracking_mode` column, or enforced in `apply_movement` if the check can't be expressed cleanly).

### Procurement changes in Phase 2
- `post_line_quantity` passes `location = GRN site's Default` (bin choice comes in Phase 3).
- `reverse_line` posts its `adjust` to **the same location and lot** as the receipt (D27).
- MRS stock check (rev 4 §8): available = Σ balance **at the requesting site** − allocated. Org-wide total remains as a secondary figure.

### UI
- Settings → Sites: location tree editor (create/rename/deactivate; deactivation refused while a balance is non-zero).
- Asset detail: on-hand by site/location table; movement list shows location.
- Asset form: bin picker replaces the free-text location for single assets.

## Phase 3: lots and bin-level receipt (P1 gap 6)
### Models
- **`Lot`**: `org`, `asset` (the quantity part), `lot_no`, `mfg_date`, `expiry_date`, `created_from` (`source_kind`/`source_id`). Unique `(asset, lot_no)`.
- **`AssetType.lot_tracked`** and **`expiry_tracked`** (bools, quantity mode only).

### Snapshot extension (D25)
- The rev 4 snapshot becomes `{mode, lot_tracked}`. `POLine.tracking_mode_snapshot` stays for compatibility, plus a new `POLine.lot_tracked_snapshot`.
- In `prevalidate`, a change of `lot_tracked` is treated **exactly like a mode flip**: 409 `line_type_changed`, fixed with `reresolve-line-type`, refused if the line already has receipts under the old setting (D13 unchanged, just a wider trigger).
- A same-settings fork still posts onto the fork automatically.

### GRN / QC changes (D26)
- `GRNLine.location` (optional FK). Default = the site's Default location. `prevalidate` adds: location belongs to the GRN site, is active, `usage=internal` → else 400 `invalid_location`.
- Single-tracked: each `GRNUnit` may override the bin. It is written to `Asset.location_ref`.
- Lot-tracked quantity lines: QC splits the accepted quantity into **`GRNLot`** rows (`lot_no`, `mfg_date`, `expiry_date`, `quantity`). `prevalidate` checks Σ lot qty = accepted qty, expiry present when `expiry_tracked`, expiry ≥ GRN date (warning, not error).
- Posting writes **one `receive` movement per lot**, each with the same `source_id` (the GRN line). `Lot` rows are reused when `(asset, lot_no)` already exists, but mismatched expiry → 409 `lot_conflict`.
- Reversal: one opposite `adjust` per lot. Lots created by this GRN and left with no other movements are soft-deleted.

### UI
- QC grid: lot sub-rows for lot-tracked lines; bin picker per line/unit.
- Asset detail: on-hand by lot with expiry, near-expiry badge.

## Phase 4: inventory documents (P1 gaps 3, 5)
Both live in the **inventory** app and write only via `apply_movement`.

### StockTransfer (D28)
- `StockTransfer`: `org`, `number` (from the human-id service), `from_site`, `to_site`, `status` (`draft` → `in_transit` → `received`, or `cancelled`), `dispatched_at/by`, `received_at/by`, `notes`.
- `TransferLine`: `asset`, `lot` (nullable), `quantity`, `from_location`, `to_location`, `received_qty`.
- **Dispatch** posts `transfer_out` from `from_location` into the org transit location. **Receive** posts `transfer_in` from transit into `to_location`. Short receipts leave the gap in transit and must be resolved by a `receive_remaining` or a `write_off` (adjust to scrap, needs `inventory.count_approve`).
- Same-site bin moves skip transit: one document, posted both legs at once.
- Single-tracked assets: the line names the asset (qty 1). On receive, `current_site` and `location_ref` change. No ledger row (same rule as rev 4: single assets have no ledger entries). The transfer line is the history.
- **Direct edits of `current_site`:** refused for quantity items with a non-zero balance ("use a transfer"). For single assets the edit is allowed but auto-creates a posted same-day transfer so history is never lost.
- `source_kind="transfer_line"`.
- Capability **`inventory.transfer`**; receiving at the destination needs the capability **and** visibility of the destination site.

### StockCount / Audit (D29)
- `StockCount`: `org`, `number`, `site`, `location_scope` (subtree root, nullable = whole site), `kind` (`cycle` | `full` | `asset_audit`), `blind` (hide expected qty), `status` (`draft` → `counting` → `review` → `posted`, or `cancelled`), `frozen_at`.
- `CountLine`: `asset`, `location`, `lot`, `expected_qty` (snapshot at freeze), `counted_qty`, `variance`, `note`; for single assets `expected_present` / `found` / `found_location`.
- Expected quantity at posting = frozen snapshot **+ net movements since `frozen_at`** at that location/lot, so normal work can continue during a count without creating false variances.
- Posting writes one `adjust` movement per non-zero variance (`source_kind="count_line"`, `reason="Count CNT-xxxxx"`). Missing single assets get a configurable status (e.g. "Missing"), never deleted.
- Capabilities **`inventory.count`** (create/enter) and **`inventory.count_approve`** (post when |variance value| > org threshold, set in Settings → Inventory).

### Interaction with GRN reversal (D27)
`reverse_line` already refuses after "later receipts" and assignments. Add:
- refuse if any `transfer_out/transfer_in` or `count_line` movement touched the received asset/location/lot after the receipt;
- refuse if the balance at the receipt's location/lot is below the quantity to reverse (covers consumption by builds or assignments that the checks above already block, as a final safety net).
The refusal lists the blocking documents (TRF-…, CNT-…), the same way rev 4 lists PR/PO references.

### Capabilities (D32)
`inventory.transfer`, `inventory.count`, `inventory.count_approve`, `inventory.locations_manage` → `VALID_CAPABILITIES` and registered in `inventory/apps.py::ready()` through the public `controller_grant_*` adapters, idempotently (same test as D12).

## Phase 5: P2 entities, ordered by procurement impact
Build each as its own reviewed plan. The procurement touchpoint of each is fixed here so the seam doesn't need rework later.

| Order | Entity | Procurement / seam impact |
|---|---|---|
| 5.1 | **AssetEvent / ActivityLog** | GRN post, reversal, transfer, count, reresolve and close-line write events. Build first: every later entity wants history. Single assets finally get a ledger-like trail. |
| 5.2 | **Attachment (1:N, typed)** | GRN attachments (vendor invoice, delivery challan, QC report) link to the created assets. Migrate `AssetImage`/`AssetInvoice` 1:1 rows into it. |
| 5.3 | **AssetModel** | PR/PO lines for new single-tracked items may name an `AssetModel` (manufacturer = partner). GRN-created assets inherit it. Snapshot logic unchanged (model doesn't change tracking mode). |
| 5.4 | **Contract** (warranty, AMC, lease, insurance) | `POLine.warranty_months` → on post, a `warranty` Contract linked to the created assets, with the PO vendor. Reversal soft-deletes contracts it created. |
| 5.5 | **DepreciationSchedule + Entry** | Basis = `purchase_value` (= `posted_purchase_value` for GRN assets; plus landed cost when `capitalize_gst`/landed-cost policy says so), start = `acquired_on` (GRN date). **Reversal refused once any entry is posted** (added to the D11 list). `AssetType.default_depreciation_method`, `useful_life_months`. |
| 5.6 | **Department / CostCenter** | PR gains `requesting_department`. GRN-created assets get it as owner. `AssetAssignment.assignee_type` (user/department/site/asset). |
| 5.7 | **Disposal** | Disposed/retired assets are excluded from the PR item picker and from MRS availability. |
| 5.8 | **Reservation**, **MaintenancePlan** | No direct seam. MaintenancePlan adds a "Raise PR for spares" hook (Phase 6). |

**Asset.status (D30):** when status becomes an enum or an FK to `AssetType.lifecycle_states`, blank must stay valid ("not set"). D6 parity (GRN-created = manual = `""`) must hold after the change.

## Phase 6: P3 and loop-backs
- **"Raise PR" buttons** (rev 4 §8 hook): from low stock (now per site, using `StockBalance` and a per-site reorder point), from `preview_build` shortages, from MaintenancePlan spares, and from near-expiry lots. All use the minimal `lines[].item + quantity` API.
- **Meter + MeterReading**: maintenance only, no seam.
- **Licence + LicenceSeat**: only if IT assets are in scope. If built, PR lines get `kind=licence`; the GRN posts a Licence with seat quantity instead of an Asset. Needs its own seam plan.

---

## Decision log (new)
| # | Decision |
|---|---|
| D20 | Party = the existing `BusinessPartner`; no new table. It must live below assets. `Asset.vendor` set from the PO vendor at GRN post; `AssetInvoice.vendor` becomes an FK with `vendor_text` kept as legacy. |
| D21 | `Location` is a tree under `Site` with a Default root per site and one org transit location. |
| D22 | The ledger stays signed single-leg (not from/to double entry). `StockMovement` gains `location`, `lot` and the kinds `transfer_out`, `transfer_in`, `return`. Transfers are two linked rows. Keeps `apply_movement`, BOM builds and rev 4 unchanged. |
| D23 | `StockBalance` is maintained inside `apply_movement` (same transaction, row lock), not by a DB trigger. `quantity_on_hand` remains a cache. A rebuild command checks drift. |
| D24 | Backfill: GRN movements → GRN site (done from procurement); everything else → `current_site`; no site → org "Unassigned". Command with `--dry-run`; NOT NULL only after it passes. |
| D25 | `AssetType.lot_tracked`/`expiry_tracked`; the line snapshot widens to `{mode, lot_tracked}`; a change is a 409 like a mode flip (D13 mechanism). |
| D26 | GRN line/unit picks a bin (default = site Default); lot-tracked lines split accepted qty into `GRNLot` rows; one receive movement per lot. |
| D27 | Reversal posts to the receipt's own location/lot and is refused after any later transfer/count on it, or if the balance there is short. |
| D28 | `StockTransfer` in the inventory app: dispatch → transit → receive; single assets move by record, not ledger; direct site edits are refused (quantity) or auto-recorded (single). |
| D29 | `StockCount` with freeze + "movements since freeze" so counts don't block work; variances post as `adjust`; approval above a value threshold. |
| D30 | Blank status stays valid when status becomes an enum/FK (keeps D6). |
| D31 | Valuation unchanged: org-wide latest `unit_cost`; per-location value = balance × unit_cost. FIFO/lot costing out of scope. |
| D32 | New capabilities `inventory.transfer`, `inventory.count`, `inventory.count_approve`, `inventory.locations_manage`, registered like D12/D16. |

## Critical files (expected; verify against the branch)
- **New:** `procurement/assets_bridge.py`, asset-history view/serializer, `procurement/apps.py` (Phase 1); `assets` models/migrations for `Location`, `Lot`, `StockBalance` and the rebuild command (Phase 2); `GRNLot` (Phase 3); `inventory/models.py` (`StockTransfer`, `TransferLine`, `StockCount`, `CountLine`), `inventory/services.py`, views, `inventory/apps.py` (Phase 4).
- **Modified:** `assets/services.py` (`apply_movement` location/lot/balance; build checks), `assets/serializers.py` (location FK, vendor FK), `procurement/models.py`/`services.py` (rev 4 fields + `GRNLine.location`, lot snapshot), `access/registry.py`, `project_types/workflow_schema.py` (`VALID_CAPABILITIES`), the `BusinessPartner` app.
- **UI:** `AssetDetailPage.tsx`, `AssetDetailDrawer.tsx`, `AssetDialog.tsx` (bin picker), procurement GRN result + QC grid (bins, lots), `procurementClient.ts`, new `modules/inventory` (transfers, counts), Settings → Sites (location tree) and Settings → Inventory (count threshold).

## Verification
Every phase: `python manage.py makemigrations --check`, `python manage.py test procurement assets inventory access`, `ruff check`, `npx tsc --noEmit`, `npx vitest run`, plus an import-lint test that assets imports neither procurement nor inventory.

**Phase 1:** all rev 4 tests, plus GRN-created asset has `vendor` = PO vendor; vendor-text migration matches by name and reports the rest; `BusinessPartner` move leaves the table untouched.

**Phase 2:**
- backfill dry-run on a production snapshot: zero drift between Σ balance and `quantity_on_hand`, GRN movements land on the GRN site;
- `apply_movement` refuses a negative balance; concurrent receives on one balance row don't lose updates (two-thread test);
- `rebuild_stock_balance` is idempotent;
- GRN post → balance at the site's Default; reversal → back to zero at the same location;
- MRS availability uses the requesting site;
- deactivating a location with stock is refused.

**Phase 3:**
- lot split must sum to accepted qty; missing expiry refused when `expiry_tracked`;
- an existing lot is reused; mismatched expiry → 409 `lot_conflict`;
- turning on `lot_tracked` between PO and GRN → 409; `reresolve-line-type` fixes it; refused after receipts under the old setting;
- reversal removes per-lot quantities and soft-deletes orphaned lots.

**Phase 4:**
- transfer dispatch/receive: balances move source → transit → destination; short receipt leaves the remainder in transit;
- single-asset transfer updates `current_site`/`location_ref` with no ledger row; a direct site edit creates a posted transfer;
- count with movements during counting gives zero false variance; posting writes `adjust` rows; above-threshold posting needs `count_approve`;
- GRN reversal refused after a transfer or count touched the receipt, listing TRF-/CNT- numbers;
- capability registration twice → no duplicates.

**Manual:** the rev 4 end-to-end flow; then receive into a bin with two lots, transfer one lot to another site, cycle-count both sites, and confirm the asset page, the ledger links ("From GRN-…", "TRF-…", "CNT-…") and the balances agree.

## Open questions
1. Where does `BusinessPartner` live today? If in `procurement`, is the move to a shared app acceptable in Phase 1?
2. Should single-tracked assets also write ledger rows once locations exist (so one report covers both), or keep rev 4's rule of no ledger rows for single assets? This plan keeps rev 4's rule.
3. Org "Unassigned" site for assets with no site: acceptable, or should the backfill stop and ask?
4. Count approval threshold: by variance value, by quantity, or both?
5. Does the business need FIFO / batch costing (pharma, food), or is latest cost (D31) enough for now?
6. Are licences (P3) in scope at all? It decides whether `product kind = licence` is planned into the PR line now.

Nothing in the target repositories is committed or pushed until asked. When it is, commits and PRs are authored by divya-rosy with no Claude trailer (standing rule from rev 4).
