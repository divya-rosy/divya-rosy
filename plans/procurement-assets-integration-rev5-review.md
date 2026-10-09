# Review of "Procurement ↔ Assets integration plan (rev 5)"

**Reviewed against:**
1. *Inventory & Asset Management Applications* report (Oct 2026), mainly Part 4, the gap analysis of `assets-erd_1`.
2. *Procurement ↔ Assets integration plan* rev 4 (D1–D19).
3. The *Procurement to Payment Process* flowchart (EsysFlow ERP): sections A–E.

**Not checked against code.** The esysflow repositories (`django-backend`, `esysflow_ui`) were not available. Every finding comes from the documents alone. File and line references in rev 5 were not re-verified.

## Verdict
Rev 5 is **not ready to approve as is**.

| Requirement source | Status today | After the fixes below |
|---|---|---|
| Report: 6 P1 gaps | Covered, but with 3 contradictions and 3 design gaps | Fully covered |
| Report: field changes, P2/P3 entities | Partly listed | Every item either built or deferred on purpose |
| Rev 4 (D1–D19) | Kept, one inconsistency (reresolve) | Consistent |
| Flowchart: steps that touch Assets | 6 missing | Covered |
| Flowchart: sourcing, payment, invoice | Not in this plan | Must be confirmed in `procurement-phase1-v4.md` (§6) |

Rev 6 needs: fixes **R1–R6** (§1), **FC1–FC6** (§2), **S1–S7** (§3) and answers to **Q1–Q6** (§4).

---

## 1. Must-fix: contradictions and design gaps

### R1. The two transfer descriptions don't match (contradiction)
- **Where:** D26 vs §0 F4 and the F4 test.
- **Problem:** D26 says a transfer is two movements (− source, + destination). F4 sends stock through a transit site and says "totals never drop mid-transfer". That takes four movements.
- **Fix (recommended):** keep the transit site.
  - Send = `−source` + `+transit`.
  - Receive = `−transit` + `+destination`.
  - A transfer between bins of **one** site skips transit: two movements, posted at once.
  - All legs: `kind="transfer"`, `source_kind="transfer_line"`.
- **Update:** D26 text, F4, the F4 test ("send: source −q, transit +q, total unchanged; receive: transit −q, destination +q").

### R2. `reresolve-line-type` ignores the lot snapshot (contradiction)
- **Where:** §2 "Recovery for a mode change" vs D23.
- **Problem:** §2 says the action "updates only `asset_type` and `tracking_mode_snapshot`". D23 says a `lot_tracked` flip is fixed by this same action.
- **Fix:**
  - The action updates `asset_type`, `tracking_mode_snapshot` **and `lot_tracked_snapshot`**.
  - It is refused when the line already has posted receipts under the old mode **or the old lot setting**.
  - The audit event records old and new mode and lot setting.
- **Test:** a lot flip on a line with no receipts → reresolve succeeds and the re-post creates lots; a lot flip after one posted GRN → reresolve refused.

### R3. Reversal doesn't restore the vendor on an existing part (contradiction)
- **Where:** §3 "later receipts update it" vs §4, which restores only `unit_cost`.
- **Fix (recommended):** don't overwrite it.
  - `Asset.vendor` is set **only when the bridge creates** the asset or part.
  - "Last supplier" of a quantity part is read from asset-history (latest non-reversed receipt), not stored.
  - Nothing then needs restoring on reversal.
- **Alternative:** keep overwriting, store `GRNLine.previous_vendor`, and restore it in `reverse_line` like `unit_cost`.
- **Test:** a second GRN from vendor B on a part created from vendor A → `Asset.vendor` stays A; asset-history shows B as latest.

### R4. Stock taken out without naming a bin is refused (design gap)
- **Where:** F2 `apply_movement` "blocks a negative balance at that site/bin/lot".
- **Problem:** MRS issue, BOM builds and assignments name no bin. Once GRNs put stock into bins, the site-level balance (bin NULL) is 0, so every one of these would be refused.
- **Fix (recommended):** an outflow with `location=None` **picks bins automatically** inside the site:
  1. lots by earliest expiry (when lot-tracked);
  2. then the site-level balance (bin NULL);
  3. then bins in `Location.code` order.
  - A caller can still pass a bin; then only that bin is used.
  - Refused only when the **site** total is short. The error states available vs requested.
- **Test:** 10 in bin A1, 0 at site level; MRS issue of 6 with no bin → succeeds from A1.

### R5. Issue by earliest expiry turns one movement into several (design gap)
- **Where:** F3 ("writes one movement per lot") and R4 (one per bin).
- **Problem:** BOM builds and `AssetAssignment` (which links to its movements) expect one movement back.
- **Fix:**
  - `apply_movement` **always returns a list** of movements. It has one element when there is no split.
  - Callers that link movements store all of them. If `AssetAssignment` → movement is a single FK today, change it to a link table or M2M. Verify this in code.
  - Returns and check-ins reverse **the same** movements (same lot and bin), not a fresh expiry-order pick.
  - The pick is limited to the issue's site, and to its bin when one is given.
- **Test:** an assignment of 15 split across two lots → two linked movements; check-in returns 10 + 5 to the original lots and bins.

### R6. Count snapshot goes stale (design gap)
- **Where:** F5 `expected_qty` = balance when counting starts.
- **Problem:** any receipt or issue during the count shows up as a false variance.
- **Fix (recommended, no lock):** at posting, expected = snapshot + net movements since the snapshot for the same asset, site, bin and lot. Variance = counted − that figure.
- **Test:** snapshot 10; a 4 is issued during counting; counted 6 → variance 0, no adjust written.

---

## 2. Flowchart gaps that touch Assets

| Flowchart step | Problem in rev 5 | Fix |
|---|---|---|
| **D: QC result → Rejected → Concession ("accept with deviation note")** | QC only has accepted and rejected | **FC1** |
| **D: Rejected → Replacement (re-GRN and re-QC)** | Doesn't say rejected qty stays open on the PO | **FC2** |
| **D: Rejected → Return to Vendor (debit note or refund)** | No return document; returning stock after posting only works by reversal | **FC3** |
| **D: "Update Inventory / Asset (Final Update)" after payment** | Plan updates Assets at QC, and invoice-time changes are undefined | **FC4** |
| **E: Stock available for issue? No → Raise PR**, E3 partial issue | §8 raises PRs only from low stock and builds, not from an MRS shortage | **FC5** |
| **Step 3: Item available in stock?** | No check chooses between issuing from stock and buying | **FC6** |

### FC1. QC concession
- QC result per line or unit: `accepted` | `rejected` | `concession`.
- `concession` needs a `deviation_note`. It posts **exactly like accepted** (same bridge path, D1–D4 unchanged).
- Stored as `GRNLine.qc_result` + `deviation_note` + `concession_by/at`.
- Asset-history shows "Accepted on concession: <note>".
- Q3 decides whether a concession needs its own capability.

### FC2. Rejected quantity and replacement
- PO line `received_qty` counts **accepted + concession** only. Rejected quantity goes back into `open_qty`, so a **replacement GRN** can be raised against the same line and re-QC'd.
- New `GRNLine.rejection_disposition`: `replace` | `return` | `close`.
  - `replace` leaves the qty open.
  - `return` creates an RTV (FC3).
  - `close` runs `close-line` with the **Re-procure?** question (D17/D18), as the flowchart shows.
- Rejected quantity **never** reaches the ledger or Assets.

### FC3. Return to vendor (RTV)
- New `procurement.VendorReturn` + `VendorReturnLine` (number `RTV-#####`), each line pointing at a GRN line.
  - **Rejected at QC:** no stock movement (it was never posted). The RTV only records the return and gives finance the debit-note data (qty × PO unit cost + tax).
  - **Already posted (defect found later):** the bridge writes a `return` movement (new `StockMovement` kind; this is also the report's missing `return` kind). The movement goes out of the original site, bin and lot, with `source_kind="rtv_line"`.
  - **Single assets:** the asset is soft-deleted or set to the type's returned/disposed state (Q4). It is never hard-deleted.
- The checks used for reversal (children, BOM use, assignments) also block an RTV of posted stock, but later receipts or transfers elsewhere do not.
- Needs `procurement.reverse` or a new `procurement.return` capability (Q3).
- The debit note and refund themselves are finance documents, specified in v4.

### FC4. "Final update" after invoice and payment
- **Keep posting at QC acceptance.** Stock physically exists then, and MRS needs it. Write down that the flowchart's last box is the *financial* final update, not the first stock entry.
- At **invoice match** (3-way match succeeds), call new bridge function `apply_invoice(grn_line, invoice_line)`:
  - Stores `GRNLine.invoiced_unit_cost`.
  - **Invoice price = PO price:** nothing changes in Assets.
  - **Single-tracked, price differs:** update `purchase_value` to the invoiced cost (0.01) and move `posted_purchase_value` to it, so the reversal cost check (D4) still holds. Then run `sync_single_item_current_value` + `sync_landed_cost`.
  - **Quantity-tracked, price differs:** update `unit_cost` only if this GRN is still the part's latest receipt; otherwise record the variance only. `purchase_value` is never touched (D1).
  - The invoice is linked so asset-history shows "Invoice INV-… · matched".
- At **payment release**, Assets changes nothing. Asset-history shows "Paid on <date>" from procurement data.
- A GRN line with a matched invoice **cannot be reversed**. Use an RTV and a debit note instead.
- Q5 confirms this.

### FC5. MRS shortage → Raise PR, and partial issue
- MRS line availability = issuing-site `StockBalance` − `allocated_quantity` (already in §8).
- Shortfall = requested − available.
- **Partial issue (E3):** issue what is available. The MRS line keeps `pending_qty` open.
- **"Raise PR for shortfall"** creates a PR through the minimal `lines[].item + quantity` API, with deliver-to site = MRS site and a link `PRLine.source_kind="mrs_line"`, `source_id`.
- When the PR's goods are posted, the MRS line shows them as available to issue. It does **not** issue automatically.
- **E6 Close MRS:** the line closes when issued = requested, or on a short-close with a reason.

### FC6. Step 3: "Item available in stock?"
- New read endpoint `inventory/availability/?asset=&site=&qty=` → available by site, bin and lot, plus a yes/no for the qty.
- The requirement entry screen calls it:
  - **yes** → create an MRS (section E);
  - **no** → create a PR (section A/B);
  - **partly** → an MRS for the available quantity and a PR for the shortfall.
- No new "requirement" entity in this plan.

---

## 3. Smaller items

| # | Item | Fix |
|---|---|---|
| S1 | Report: `StockMovement` kind `return` missing | Added by FC3 |
| S2 | Report field changes not mentioned anywhere | Add to the D29 deferred list: barcode/RFID on Asset; `Asset.status` enum (**blank must stay valid**, D6); `AssetAssignment.assignee_type`; `Site.parent`/`timezone`; `MaintenanceLog.plan`/`vendor`; `AssetType` depreciation defaults; many invoices per asset (`AssetInvoice` → Attachment) |
| S3 | F4: direct edit of `current_site` and single assets in transit | Quantity item with a non-zero balance: site edit refused ("use a transfer"). Single asset: the edit auto-creates a posted same-day transfer. While in transit, a single asset's `current_site` = the transit site, shown as "In transit to X" |
| S4 | D27 too strict: any transfer of a non-lot asset blocks reversal | Limit to movements at the receipt's **site and bin** (and lot), plus the balance check |
| S5 | F5: no approval for large count variances | Recommended: `inventory.count_approve` needed to post when total variance value > an org threshold (Settings → Inventory). See Q1 |
| S6 | §0 inventory foundations sit inside the procurement v5 doc | Move §0 to `inventory-foundations.md`; v5 links to it and keeps only the receiving-side rules (bins, lots, D25, D27) |
| S7 | Valuation with bins and lots isn't stated | Add a decision: `unit_cost` stays org-wide latest cost; value per bin or lot = balance × unit_cost; FIFO/lot costing not built. See Q2 |

---

## 4. Decisions needed from the owner

| # | Question | Recommendation |
|---|---|---|
| Q1 | Do posted counts need approval above a variance threshold? | Yes, by value, with `inventory.count_approve` |
| Q2 | FIFO / lot costing, or latest cost? | Latest cost now (S7); revisit for pharma/food customers |
| Q3 | Capabilities for QC concession and RTV | Concession: `inventory.receive` + mandatory note. RTV: new `procurement.return` |
| Q4 | A single asset returned to the vendor after posting: soft-delete, or keep with a "Returned" state? | Keep it with a returned/disposed state, so its history survives |
| Q5 | Is FC4 (stock at QC, cost correction at invoice match) the intended meaning of "Final Update"? | Yes |
| Q6 | The flowchart puts **sourcing (A) before the PR (B)**; rev 4/5 start at the PR. Which order is right? | Confirm in v4: either the RFQ comes from an approved PR, or the PR comes from the selected quote |

---

## 5. New decisions for rev 6 (proposed numbering)

| # | Decision |
|---|---|
| D30 | Transfers go through the transit site: 4 movements between sites, 2 within one site (R1) |
| D31 | `reresolve-line-type` also updates `lot_tracked_snapshot`; refused after receipts under the old mode or lot setting (R2) |
| D32 | `Asset.vendor` is set only when the asset is created; "last supplier" comes from asset-history (R3) |
| D33 | Outflows with no bin pick lots by earliest expiry, then the site-level balance, then bins in code order, within the site; `apply_movement` returns a list (R4, R5) |
| D34 | Count expected = snapshot + net movements since the snapshot; approval above a value threshold (R6, S5) |
| D35 | QC result `concession` with a deviation note posts as accepted (FC1) |
| D36 | Rejected qty returns to `open_qty`; disposition replace/return/close (FC2) |
| D37 | `VendorReturn` (RTV) document; `return` movement for posted stock (FC3) |
| D38 | Stock posts at QC; `apply_invoice` corrects cost at invoice match; no reversal after a matched invoice (FC4) |
| D39 | MRS partial issue + "Raise PR for shortfall" linked by `source_kind="mrs_line"`; availability endpoint for step 3 (FC5, FC6) |
| D40 | Report field changes and P2/P3 entities deferred, listed in D29 (S2) |

---

## 6. For `procurement-phase1-v4.md` (outside this plan)
These flowchart steps don't touch Assets. They must be in the procurement document; this review couldn't confirm that, because v4 wasn't available.

- [ ] Step 1–2: requirement raised by Department/Project; budget approval (and a budget check against the project).
- [ ] **A** sourcing: A1 RFI (loop if information is not OK), A2 RFQ, A3 receive and compare quotes (loop if invalid), A4 technical and commercial evaluation, A5 select supplier and agree terms. Order relative to the PR: Q6.
- [ ] **B**: B2 department approval as its own step; B3 finance approval; rework loop to B1; cancel → close PR.
- [ ] **C2**: supplier acknowledgement.
- [ ] **C3** payment terms:
  - advance: proforma invoice → approval → release advance;
  - milestone: plan → verification and approval per milestone → release;
  - credit: no payment before dispatch;
  - then "supplier confirms order".
- [ ] **D**: supplier dispatch and in transit; gate entry and unloading (before the GRN).
- [ ] **D**: receive tax invoice; 3-way match (PO + GRN accepted qty + invoice); invoice exception → resolve loop; AP liability with due date; payment approval; release payment.
- [ ] Debit note / refund for RTV (FC3).

---

## 7. Tests to add in rev 6
- R1: transfer between sites passes through transit with the total unchanged; transfer between bins of one site posts both legs at once.
- R2: lot flip → reresolve succeeds without receipts and is refused with old-setting receipts.
- R3: second-vendor GRN leaves `Asset.vendor` unchanged.
- R4: issue with no bin draws from bins; refused only when the site total is short.
- R5: split issue returns several movements; assignment check-in returns stock to the same lots and bins.
- R6: movements during counting give no false variance.
- FC1: concession posts like accepted and stores the note.
- FC2: rejected qty reopens the PO line; a replacement GRN posts against it; close with re-procure Yes/No behaves as D18.
- FC3: RTV of rejected goods → no movement; RTV of posted stock → `return` movement from the original site, bin and lot; refused when the asset is in use.
- FC4: invoice at PO price → no change; higher price on a single asset → `purchase_value` and `posted_purchase_value` updated; quantity part with a later receipt → variance only; reversal refused after a matched invoice.
- FC5: MRS of 10 with 6 available → issue 6, pending 4, PR raised for 4 and linked to the MRS line.
- FC6: availability endpoint returns per-site, per-bin and per-lot quantities, and routes to MRS, PR or both.

## 8. Rev 6 acceptance checklist
- [ ] R1–R6 fixed in the decision log, §0, §2, §3, §4 and the tests.
- [ ] FC1–FC6 added (new decisions D35–D39, posting and reversal rules, UI: QC result and disposition, RTV, MRS shortage).
- [ ] S1–S7 applied; D29 lists every deferred report item.
- [ ] Q1–Q6 answered and recorded.
- [ ] §0 moved to an inventory plan (S6), or the reason for keeping it in v5 recorded.
- [ ] v4 checklist (§6) confirmed against `procurement-phase1-v4.md`.
- [ ] Plan re-checked against the code on `feature/procurement-module` / `staging`.
