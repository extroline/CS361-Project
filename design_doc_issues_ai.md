# Review Report: Rodney's Bike Shop Design Doc

**Severity key:** 🔴 High (blocks implementation or is self-contradictory) · 🟠 Medium (ambiguous or incomplete) · 🟡 Low (polish)

## Summary

| Category | 🔴 | 🟠 | 🟡 |
|---|---|---|---|
| Logical | 7 | 8 | 1 |
| Structural | 3 | 6 | 3 |
| Diagram | 4 | 4 | 2 |
| Language / consistency | 0 | 1 | 4 |

The biggest problems are the contradictions around inventory and availability, the unresolved "online vs. kiosk" scope, undefined pricing rules, and a component diagram whose data flow doesn't match the component table.

---

## 1. Logical Issues

### 🔴 High

**L-1. Scope contradiction: "online" vs. kiosk-only**
- *Overview:* clients "pay for it online."
- *Everything else:* a physical kiosk with a card reader, a printer, and a keypad. The only mention of a website is in the `Update Inventory` description.
- *Fix:* state whether a web or online channel exists. If it does, it is missing from Actors, Components, and the Diagram. If not, remove "online."

**L-2. Inventory contradiction**
- *Non-Goals:* "No current inventory system, it's on paper."
- *Overview, Actors, Components:* staff "list bikes," "Add/Remove Bikes," and an `Update Inventory` component feeds the kiosk.
- The kiosk cannot show available bikes without some digital bike catalog and availability data. The doc never defines where this data lives or who owns it.
- *Fix:* define a minimal digital catalog and availability store, and scope what stays on paper (for example, only physical stock counts).

**L-3. The availability rules require a system the Non-Goals disclaim**
- The assumptions ("bookable 4 hours after expected return," "available the day after service") need availability calculation.
- The Non-Goals say there is no auto system for overbooking, damage, late returns, or stock limits.
- Using *expected* return time means a late return directly causes a double booking, which is the case the Non-Goals say will not be handled. The doc never says who resolves it or how.
- *Fix:* separate "prevent double-booking under normal conditions" (in scope) from "recover from late and damaged returns" (human process), and document the manual fallback.

**L-4. Pricing rules are ambiguous and incomplete**
- "4-hour intervals" and "a day's fee equals 12 hours" don't define billing for in-between durations (for example, 16 hours: one day plus one interval, or two days?).
- Is the day fee equal to 3 intervals (a discount), or something else?
- It's unclear whether the rental clock runs overnight or only during business hours.
- Nothing defines price per bike, per bike type or size, or globally.
- Tax, fees, deposits, and tips are never mentioned, yet "Charge correct prices" is a goal.
- *Fix:* add a pricing section with a formula and a table of worked examples.

**L-5. Payment lifecycle gaps**
- The pre-auth amount is undefined. It must cover extensions, late fees, and damage, or those charges need a separate mechanism.
- Pre-auth holds expire (often after about 7 days). The doc doesn't say what happens for advance reservations or 7-day rentals.
- The final charge after return needs a stored card token. This conflicts with "no account needed," and the doc doesn't say where the token lives.
- Employee "Execute Refunds" conflates two operations: voiding a pre-auth and refunding a settled charge.
- Replacement charges are "human intervention," but there is no mechanism and no authority to charge beyond the pre-auth.
- A failed final charge after the bike has been returned is not handled. "Card failures are caught" only covers the reservation step.

**L-6. Reservation model is undefined**
- The Overview says "reserve," but the Kiosk description says "rent it for the day," and rentals run up to 7 days.
- Can customers book for future dates at the kiosk, or only start a rental now?
- Pickup is employee-only during business hours, so what does a reservation add over a walk-up rental?
- With no account, how does a customer or employee look up an order for pickup, cancellation, or extension (confirmation code, name, phone)? The receipt is the only link, and nothing covers a lost one.

**L-7. Capacity math is unsound**
- "60 bikes → about 60 rentals per day" ignores 4-hour intervals (a bike can be rented several times a day) and multi-bike or multi-day orders (a bike can be out for 7 days).
- "Not high-concurrency" is plausible, but it isn't derived from the numbers.
- Concurrency still matters: an employee cancelling or putting a bike into maintenance while a customer books the last bike. Define consistency rules, such as the last-bike race and the number of kiosks.

### 🟠 Medium

**L-8. Maintenance behavior is undefined**
- "Next day" availability is assumed, but staff put bikes into maintenance "as needed," so duration is unknown.
- What happens to existing reservations for a bike put into maintenance (reassign, cancel, refund)?

**L-9. Return buffer ignores business hours**
- A bike returned at 5 PM becomes bookable at 9 PM, which is outside the pickup hours. Business hours, time zone, and holidays are never defined.

**L-10. "Give-Change" implies cash**
- Cash handling appears in the Kiosk actor ("Give-Change"), but the rest of the doc is card-only (pre-auth) with no cash hardware in the diagram. Either remove it or add cash flows.

**L-11. PIN handling**
- The UI "allows the customer to input PIN," but PIN entry normally belongs on a certified card-reader PIN pad. A general keypad UI raises PCI concerns. Also, most credit-card pre-auths don't need a PIN. State the card types supported.

**L-12. Actors don't cover stated capabilities**
- The Overview says staff "set prices" and "put bikes into maintenance." Neither appears in any actor's action list.
- "Remove Bikes" is not the same as maintenance.
- Extensions, size swaps, and damage assessment appear in Constraints but not in Actors.
- Owner and manager are never defined as actors, and the permission matrix is missing.
- Bike types and sizes (needed for size swaps) have no model.

**L-13. Transaction Log scope is too narrow**
- It logs "transactions occurring on the terminal," but the Goals say "log all orders." Employee actions (cancel, refund, price changes, maintenance) happen on the dashboard and are not covered.
- Declined attempts aren't specified, nor are log retention, immutability, or PII/card-data rules (the log must never store full card numbers).

**L-14. Failure modes beyond restart are missing**
- Not covered: printer out of paper or jammed, processor timeout or outage, network loss to the dashboard, dashboard offline, power loss mid-payment.
- Questions to answer:
  - Is an order still valid if the receipt fails to print?
  - How are retries kept from double-charging (idempotency)?
  - After a restart, how are orphaned pre-auths (authorized but not logged) reconciled?

**L-15. No-show and abuse assumption conflicts with other rules**
- "Don't design against mass no-shows" is fine, but with no deposit and no account, one customer could hold many bikes.
- The doc doesn't say what happens to the pre-auth on a no-show.
- The cancellation and refund policy is absent altogether.

### 🟡 Low

**L-16. Unreferenced Non-Goal**
- "Ad system handled by other team" appears nowhere else. If ads show on the kiosk, the interface is undefined. If they don't, remove it.

---

## 2. Structural Issues

### 🔴 High

**S-1. Misclassified Constraints vs. Assumptions**
- "Role-based permissions" is a requirement, not a constraint.
- "Card failures are caught" is a requirement, not an assumption.
- "Bike out of service available next day" and the return-buffer rule are business rules, not assumptions.
- "All pickups/returns go through an employee" is a scope/business rule.
- Real constraints (60 bikes, 7-day cap, processor API) are mixed with these.
- *Fix:* split into Constraints, Assumptions (to validate), and Business Rules or Requirements.

**S-2. Missing core sections**
- Data model (Bike, Order, Booking, Transaction, User/Role, and the order state machine: reserved → picked up → returned → charged / cancelled / refunded)
- Interfaces and APIs (kiosk ↔ backend, dashboard ↔ backend, gateway ↔ processor)
- Error handling and failure modes
- Security and privacy (PCI scope, tokenization, RBAC enforcement, audit)
- Pricing and refund/cancellation rules
- Open questions, alternatives considered, testing, and rollout

**S-3. Non-functional requirements are mostly functional or unmeasurable**
- "Wrong Payment Info = 0 instances" is a functional requirement and can't be guaranteed (the system only knows what the processor declines).
- "100% under 5 seconds" is unrealistic (printer failures) and has no start or end event.
- "Payment confirmation 95% < 9.99s" is false precision, with no handling defined for the 5% tail (timeout, retry).
- Missing categories: availability/uptime, security, auditability, data retention, accessibility (for example ADA kiosk access), usability.
- "System Restart < 3 minutes" doesn't define the preserved state.

### 🟠 Medium

**S-4. No document metadata.** The title is just "Design Doc." Add author, date, status, version, and reviewers.

**S-5. Goals are unmeasurable and overlap with NFRs.** "Alert staff automatically" has no mechanism (dashboard, sound, push) and "Charge correct prices" has no definition of correct.

**S-6. Non-Goals contain facts that aren't goals.** "Paper inventory" is a current-state fact and belongs in Context/Constraints. Several items are negations of features nobody proposed.

**S-7. Actors section mixes concepts**
- Kiosk is both an actor and a component, and its actions (Scroll, Add to Cart, Remove Bike) are customer UI actions.
- No descriptions or use-case grouping.

**S-8. Component table is incomplete.** It omits Payment-Handler and Order-Sent and describes Update Inventory differently from the diagram. Descriptions are vague (see D-6).

**S-9. No scenarios or examples tied to requirements.** The task called for examples, but none exist yet.

### 🟡 Low

**S-10. Inconsistent formatting.** Actor actions are sometimes hyphenated ("Cancel-Orders") and sometimes spaced ("Place Order"); capitalization is inconsistent.

**S-11. No glossary.** Order, booking, reservation, rental, transaction, and session are used interchangeably. The same goes for customer/client, employee/staff/manager, and kiosk/terminal/machine.

**S-12. The doc has no table of contents or numbered sections**, which makes review comments hard to reference.

---

## 3. Component Diagram Issues

### 🔴 High

**D-1. Kiosk is drawn as a sibling, not a container**
- The table says the Session Controller runs *on* the kiosk, and the UI, Card Reader, and Printer are kiosk hardware. The diagram makes Kiosk a separate node with one arrow into SC.
- *Fix:* use a `subgraph Kiosk` containing UI, CR, SC, RP, and the local log (if local).

**D-2. Data flow is broken or backwards**
- `Customer --> UI` and `Customer --> CR` show input, but nothing flows from UI or CR into SC.
- Meanwhile `SC --> UI` and `SC --> CR` show control. Selections and card data never reach the controller.
- `CR --> Kiosk --> SC` is the only inbound path, and it bypasses the UI.

**D-3. Undefined or misused nodes**
- `Payment-Handler`: no description, and the external payment processor is never shown.
- `Order-Sent` is an event, not a component.
- `Update-Inventory` is an action drawn as a node.
- `Bank Gateway → Payment-Handler` has no return path, so approvals and declines can't reach SC.

**D-4. Missing components for stated functionality**
- A bike catalog / availability / reservation store
- A pricing configuration
- Authentication and RBAC for the dashboard
- A path for employee refunds, cancellations, and maintenance (the Employee currently connects only to the Dashboard and Update-Inventory)
- A backend or server (where does the dashboard get its data, and is the log local to the kiosk?)

### 🟠 Medium

**D-5. Push vs. pull mismatch.** The table says the Dashboard "pulls the order from the Session Controller," but the diagram and NFR describe the order being "sent."

**D-6. Update Inventory path mismatch.** The table says it "pulls from the employee dashboard," but the diagram has `Employee --> Update-Inventory --> Kiosk` with no Dashboard link.

**D-7. Vague descriptions**
- Session Controller: circular wording ("information about the machine and the functionality needed from other components").
- Bank Gateway: "connects the user's information."
- Kiosk: "rent it for the day" contradicts multi-day rentals.

**D-8. Inconsistent node shapes and names.** Actors, the Kiosk, and components share the stadium shape, and names mix hyphens and spaces.

### 🟡 Low

**D-9. Single-kiosk assumption unstated.** Multiple kiosks would change the architecture (shared state, double booking).

**D-10. No legend or labelled arrows.** Arrows should say what flows (cart, card data, auth request, order event).

---

## 4. Language and Consistency

**Spelling errors:** maintence, Excute, automically, diffrent, limitatiosn, intervaention, permissons, ownerm, avaiable, recieve, Kisosk, componets.

**Grammar:** "Card failure are caught," "Role base permissons," "a bike(s)," "as well the client."

**Terminology drift:** see S-11. Pick one term per concept and define it.

**Style:** mixed sentence fragments and full sentences in bullets; inconsistent hyphenation and capitalization.

---

## 5. Recommended Next Steps

1. Resolve the scope contradictions (L-1, L-2, L-3, L-6) with the stakeholder before editing anything else.
2. Add Pricing and Cancellation/Refund sections, then rewrite the Goals and NFRs against them.
3. Redraw the diagram with a Kiosk subgraph, an external payment processor, an inventory/reservation store, and labelled bidirectional flows.
4. Add a Data Model, an order state machine, a Failure Modes table, and a Security/PCI section.
5. Reclassify the Constraints/Assumptions, add a glossary, and run a spell check.