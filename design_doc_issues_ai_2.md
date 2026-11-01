# Design Review: Rodney's Bike Shop System

**Scope:** Logical, structural, consistency, and editorial issues in the current partial draft.
**Severity key:** 🔴 High (blocks implementation or is internally contradictory) · 🟠 Medium (ambiguous or incomplete) · 🟡 Low (polish)

---

## 1. Summary

| Category | 🔴 | 🟠 | 🟡 |
|---|---|---|---|
| Logical contradictions | 7 | 4 | 0 |
| Structural / architectural | 4 | 5 | 1 |
| Data model | 4 | 4 | 1 |
| Interfaces | 3 | 5 | 1 |
| Requirements (NFR, goals, constraints) | 1 | 5 | 0 |
| Failure modes | 3 | 3 | 1 |
| Editorial | 0 | 1 | 4 |

**Top five to fix first**
1. The payment model contradicts itself (pre-auth vs. "charged" vs. cash change).
2. The kiosk's role contradicts the "employee-only pickup" constraint.
3. `cardReader` passes the PIN through software.
4. There is no central backend, so kiosk and website bookings can't be kept consistent.
5. No interface or entity carries rental dates, so availability can't be computed.

---

## 2. Logical Issues

### L1 🔴 Payment model is inconsistent
- **Overview:** the client can "pay for it online."
- **Constraints:** card is pre-authorized at reservation and charged after return.
- **Actors:** Kiosk "Give-Change" implies cash, which appears nowhere else.
- **`cardReader` outcome:** "The customer is charged," but only a pre-authorization should happen at this step.

**Fix:** State one model, e.g. *pre-auth at booking → capture after return → card only*. Remove "Give-Change" or add a cash flow.

### L2 🔴 Kiosk role contradicts the employee-only constraint
- The component table says the kiosk is where "customers can pick the bike and rent it for the day."
- The constraints say all pickups, returns, extensions, swaps, and damage assessments go through an employee, with no self-service.
- "Rent it for the day" also conflicts with 4-hour intervals and a 7-day maximum.

**Fix:** Define the kiosk as a *reservation and payment terminal*. The employee performs the physical handover.

### L3 🔴 "Pin is not saved" vs. PIN as a software parameter
`cardReader(..., pin: Pin, ...)` and "Screen & keypad UI: allows customer to input pin" route the PIN through the Session Controller. Normally the PIN is entered on a certified reader and encrypted there. The Session Controller should never see it. This is also a PCI-scope problem.

### L4 🔴 Inventory is "on paper" but the system depends on a live catalog
The non-goals say inventory is on paper. The design has a Bike Catalog, Update Inventory, availability assumptions, and "Add/Remove Bikes." The doc never says which source is authoritative.

### L5 🔴 Overbooking: non-goal vs. buffer rule
- **Non-goal:** no automatic overbooking handling for damage or late returns.
- **Assumption:** book 4 hours after the *expected* return time.

These conflict when a bike returns late or damaged. State that the system prevents *overlapping scheduled* reservations only, and that staff handle exceptions.

### L6 🔴 "60 rentals per day is the peak"
This doesn't follow from "60 bikes." With 4-hour intervals, one bike could be rented 3 times in a 12-hour day (~180 rentals). With 7-day rentals, it could be well under 60. State the real assumption, such as peak concurrent reservations or peak bookings/day, and its basis.

### L7 🔴 Invariants that are really assumptions
- "The third party bank works every time" contradicts Failure Mode #2.
- "No one is trying to abuse the rental system" is already an assumption.
- "Kiosk can accommodate any type of customer needs" is untestable and contradicts the non-goals.

**Missing real invariants**, for example:
- No bike has two overlapping active reservations.
- Captured amount ≤ authorized amount (or an explicit overage rule).
- Every captured payment has exactly one transaction-log entry.
- A bike in maintenance cannot be reserved for that window.

### L8 🟠 Maintenance assumption is unsafe
"A bike out of service will be available the next day" can't hold for a bent frame or a part on order. It also conflicts with "put bikes into maintenance as needed," which implies a variable duration. The model needs an explicit `maintenance_until` value.

### L9 🟠 Pre-authorization timing is undefined
- **Booking horizon:** how far in advance can someone book? Pre-auth holds typically expire in about 7 days, and a 7-day rental plus any advance booking exceeds that.
- **Amount:** is it the full rental, a deposit, or the rental plus a deposit?

### L10 🟠 No-show and cancellation behavior is missing
"Mass no-show abuse is low risk" is fine, but ordinary no-shows still need a rule. Is the hold released or a fee charged? Who decides? The Employee actor has "Cancel-Orders" and "Execute Refunds" with no corresponding interface.

### L11 🟠 Terminology drift
*Order*, *reservation*, *booking*, *rental*, and *transaction* are used interchangeably. Add a glossary and use one term per concept.

### L12 🟠 "Card failures are caught"
This is an assumption that asserts the thing to be designed. Replace it with specifics: "decline, expired, and insufficient-funds responses are returned by the processor."

---

## 3. Structural / Architectural Issues

### S1 🔴 No central backend
Kiosk, website, and employee dashboard all appear to talk to a kiosk-resident Session Controller. There is no shared reservation service or database. Consequences:
- A website booking and a kiosk booking for the same bike can race (**double booking**).
- The Transaction Log appears to be kiosk-local, but the goal is to "log all orders" including web orders.
- If the kiosk drive fails (Failure Mode #4), the log is lost.

### S2 🔴 Online Website is an actor but not in the architecture
It has no component, no flow, and no data path in the diagram. The Session Controller is described as "software run on the kiosk," so how does the website use it?

### S3 🔴 Diagram and component table disagree
| Diagram element | Problem |
|---|---|
| `Payment-Handler` | Not in the component table. Is it the same as the Bank Gateway? |
| `Bike-Catalog`, `Order-Sent` | Shown as nodes but not described |
| `Update-Inventory` | Table says it "pulls from the employee dashboard," but the diagram connects Employee → Update-Inventory with no dashboard link |
| `Employee Dashboard` | Table says it "pulls the order from session controller"; the diagram shows Order-Sent (a push) |
| Kiosk | Diagram draws Card Reader → Kiosk but UI → SC directly, so it's unclear what the Kiosk box contains |
| Receipt Printer | Appears to exist only at the kiosk, so how do web customers get a receipt? |

### S4 🔴 Actors list mixes actors and systems
Kiosk and Online Website are channels or systems, not actors. Their "responsibilities" (Scroll, Add to Cart) are UI features. Employee is missing **Set Prices** and **Put Bike into Maintenance**, which the overview requires. The roles (Owner/Manager/Employee) are defined in constraints but never mapped to actions.

### S5 🟠 Push vs. pull is unspecified
Order notification, inventory updates, and catalog refresh don't say whether they are polled or pushed, or what happens when they fail. This matters for the 10-second order-sent NFR.

### S6 🟠 Missing sections that existing content already needs
- **Reservation lifecycle / state diagram.** Many failure modes refer to states that aren't defined.
- **Pricing rules** with a worked example (see §9).
- **Sequence diagram** for booking → pre-auth → pickup → return → capture.
- **Security / PCI** boundary.
- **Open questions** list.

### S7 🟠 "Non-goals" contains non-goals and status notes
"No current inventory system, it's on paper" is a fact, not a non-goal. "Ad system handled by other team" is never referenced anywhere else. Is there an ad integration point that should be in scope or at least in the diagram?

### S8 🟠 Goals aren't verifiable
"Charge correct prices" and "Alert staff automatically" have no acceptance criteria. Example: "For any rental of N intervals, the pre-auth amount equals price(N) as defined in §Pricing."

### S9 🟠 Customer-facing flows aren't covered
Actors list "Place Order, Insert Payment, Take receipt" but nothing for cancel, extend, or view-my-booking, even though "extensions" and "size swaps" are in the constraints.

### S10 🟠 Business hours are used but never defined
Pickup and return happen "during business hours," and rentals run in 4-hour intervals. How do intervals align with open hours, and what happens to a bike due back after closing?

### S11 🟡 Section ordering
Interfaces are specified before the data they depend on is stable, and Failure Modes sit after Invariants. Consider: Overview → Goals → Constraints → Architecture → Data → Lifecycle → Interfaces → Invariants → Failures → NFRs.

---

## 4. Data Model Issues

### D1 🔴 Missing core entities
`Bike`, `Reservation` (start, end, status, channel), `Payment` (auth id, captured amount, status), `Refund`, `MaintenanceWindow`, `Price`/`PriceRule`, `EmployeeRole`. Maintenance and prices are headline requirements and have no storage.

### D2 🔴 `RentedBike` has no key, no bike reference, and no dates
It holds `customer_id`, `name`, and `CheckedBike`. Availability can't be computed without a `bike_id` and a time range.

### D3 🔴 `CheckedBike` modeled incorrectly
`RentedBike.CheckedBike` is an FK to `Employee.CheckedBike`, which is a boolean on an employee and isn't a key. A flag about a *bike inspection* belongs on a Rental or Return record with `checked_by_employee_id` and a timestamp.

### D4 🔴 Kiosk and OnlineWebsite modeled as customer-linked tables
- `Kisosk` has `customer_id` FK plus `kisok_num` PK, so one kiosk could only ever serve one customer.
- Both tables duplicate `name` and `email` from Customer.
- `OnlineWebsite.ip_address` as an `int` primary key breaks for IPv6, shared IPs, and repeat visits.

**Fix:** a `channel` enum (`kiosk`, `web`) plus optional `kiosk_id` on Reservation.

### D5 🟠 Duplicate `transaction_id` on Customer
Customer lists two FKs with the same name. The relationship should run Reservation → Payment, not Customer → Transaction.

### D6 🟠 FK references to non-key columns
Customer's FKs reference `Kisosk.transaction_id` and `OnlineWebsite.transaction_id`, which are not marked PK or unique.

### D7 🟠 Guest identity unspecified
"No account needed" but there's a `customer_id`. State that Customer is a guest record created per reservation, and how repeat guests are deduplicated (or that they aren't).

### D8 🟠 Cardinalities are wrong
`Customer ||..o{ Kisosk` reads as "a customer has many kiosks."

### D9 🟡 Class diagram duplicates the ERD
It adds no methods or behavior. Replace it with a diagram that shows behavior (state machine or component classes).

---

## 5. Interface Issues

### I1 🔴 `cardReader` takes a PIN and a customer name
See L3. A name is irrelevant to reading a card. The outcome "customer is charged" conflicts with pre-auth (L1). The return type `sentToBankGateway` isn't a result.

### I2 🔴 `bikeQuery` has no date or time range
Availability depends on when the customer wants the bike. It also takes `transactionID`, `Customer_name`, and `price` as inputs. Name and transaction ID aren't needed for a catalog search, and price should be an *output* (or a filter range).

### I3 🔴 `logTransaction` is self-contradictory
- **Requires** "transaction confirmed flag," but **Guarantees** "a temp log is put in place … and updated if the log works." That implies logging *before* confirmation.
- Logging and staff alerting are coupled in the outcome. Why should a logging failure change who is alerted?
- "Staff flagged after 1 minute" conflicts with the 10-second order-sent NFR.
- "Invalid information, try again" doesn't apply to an internal DB write.

### I4 🟠 `printReceipt` mismatches
- Requires a customer name for no stated reason. It should require a *confirmed transaction*.
- "Approved/Declined" are payment terms, not print outcomes. Use `Printed | PaperOut | PrinterFault`.
- Notes say receipts are optional, but the goal says "Print receipt" and the NFR says 100% in under 5 seconds.
- "Guarantees: the transaction ID must be valid if it prints or it doesn't" is circular.

### I5 🟠 `bikeQuery` guarantee is circular
"Requires catalog up to date" and "Guarantees catalog up to date" are the same statement, and the Notes say it calls update-catalog and retries once. If that update fails, the guarantee is false. Specify the failure outcome (e.g., stale-catalog warning).

### I6 🟠 Unresolved questions in the spec text
"Payment failed try again?" and "Retry the search once ?" are open questions embedded in outcomes. Resolve them or move them to an Open Questions list.

### I7 🟠 Types and signatures
- `TransID`, `Name`, `Card`, `Pin`, `Model`, `Price` are undefined.
- `printReceipt(..., Customer_name: Name,)` has a trailing comma.
- Return types like `printed` and `Requested Bike Page` aren't types. A UI page is not a data result.

### I8 🟠 Missing operations
`checkAvailability`, `reserve`, `preAuthorize`, `capture`, `release`, `refund`, `cancel`, `extend`, `setMaintenance`, `setPrice`, `addBike`, `removeBike`. Each money-moving call needs an **idempotency key** so a retry after a gateway timeout can't double-charge.

### I9 🟡 Inconsistent naming
`transactionID` / `Customer_name` / `bikeModels` mix camelCase and snake_case.

---

## 6. Requirements Issues

### R1 🔴 NFRs conflict with failure handling
- Payment confirmation in <10 s for 95% of cases, but Failure Mode #2 retries at 5 s and again at 30 s (up to 35 s+). The other 5% is unbounded.
- Wifi-loss handling waits 45 s before alerting staff.
- Restart "under 3 minutes" conflicts with Failure Mode #4, where the kiosk is *disabled*.

### R2 🟠 "100% of receipts under 5 seconds"
This fails the first time paper runs out. Use 99% and exclude printer faults, or state that paper-out is a separate failure mode.

### R3 🟠 "9.99 seconds"
False precision. Use 10 seconds.

### R4 🟠 "0 instances of wrong payment info placing an order"
Not measurable as written. Reframe: "No reservation reaches *Confirmed* without a successful pre-authorization response."

### R5 🟠 NFRs cover only the kiosk
"Staff receive order … from kiosk" and the printed receipt ignore the website channel.

### R6 🟠 Missing NFR categories
Availability/uptime target, PCI scope, audit-log retention, offline operation time, accessibility of the kiosk UI, and data-privacy handling for name and email.

---

## 7. Failure Mode Issues

### F1 🔴 Row 1 contradicts the interface
It says "No transaction is created," but `cardReader` *requires* a valid transaction ID before reading. Also, `awaitCard` is not defined in the Interfaces section.

### F2 🔴 Row 3: wifi loss
- The kiosk can't alert staff over wifi that's down. A local alarm or a cellular fallback is needed.
- "Transaction canceled" in the customer message, but if pre-auth already succeeded, the hold must be voided. State how.

### F3 🔴 Row 4 is incomplete and unclear
"Disables the kiosk" with response "If card in unlock card" is garbled. The detection ("data differs from earlier scan") is undefined. What happens to unsynced local logs?

### F4 🟠 Row 2: gateway failure
- "Retry once in 5 seconds then in another 30 seconds" is two retries, not one.
- Only connection failure is detected, not timeouts, 5xx responses, or an *unknown* outcome (request sent, no response). The unknown-outcome case is the dangerous one and needs idempotent retry or a lookup.
- The reservation state after failure isn't defined.

### F5 🟠 Table format
Rows 2 and 4 have trailing-pipe mismatches, and the failure column mixes failure names and detection details.

### F6 🟠 Important failures not yet listed (even in a partial doc)
Capture fails after return, pre-auth expires on long rentals, kiosk/website double-booking race, printer out of paper, power loss mid-payment, employee dashboard down while orders queue.

### F7 🟡 Row naming
Row 2 ("Third Party Gateway") names a component, not a failure. Use a failure name such as "Gateway unreachable."

---

## 8. Editorial Issues

| Found | Should be |
|---|---|
| maintence | maintenance |
| Excute | Execute |
| automically | automatically |
| diffrent | different |
| limitatiosn | limitations |
| intervaention | intervention |
| ownerm | owner, |
| avaiable | available |
| Kisosk / kisok / kisok_num | Kiosk / kiosk / kiosk_num |
| recieve | receive |
| transcation | transaction |
| vaild / Invaild | valid / Invalid |
| paymen | payment |
| databse | database |
| componets | components |
| "functionally needed" | "functionality needed" |
| "Customers pins" | "Customers' PINs" |
| "Card failure are caught" | "Card failures are caught" |
| "Role base permissons" | "Role-based permissions" |

Other editorial points:
- 🟠 The title "Design Doc" is generic. Add the project name, author, date, version, and status.
- 🟡 Code fences use "``` mermaid" with a space, which some renderers don't recognize. Use "```mermaid".
- 🟡 "Online Website" is missing its colon and em-separator, unlike the other actors.
- 🟡 Invariants and Failure Modes lack the bullet or numbering used elsewhere.
- 🟡 Mixed capitalization in list items and actor verbs (`Place Order` vs. `Add to Cart` vs. `provide receipt`).

---

## 9. Worked Examples

### 9.1 Pricing (needs confirmation)
Current rule: "4-hour intervals, day's fee = 12 hours." One possible reading, assuming a day = $60 and an interval = $20:

| Rental | Billed as | Charge |
|---|---|---|
| 4 h | 1 interval | $20 |
| 8 h | 2 intervals | $40 |
| 12 h | 1 day | $60 |
| 16 h | 1 day + 1 interval | $80 |
| 24 h | 2 days | $120 |

**Open question:** does anything over 12 h round up to a full day, or is billing per interval up to a daily cap? The doc should also give a pre-auth example, such as *rental total + fixed deposit*.

### 9.2 Double-booking race (shows S1)
```
10:00:00  Kiosk shows Bike #12 available for Sat 9:00–13:00
10:00:03  Website shows Bike #12 available for Sat 9:00–13:00
10:00:20  Kiosk customer pre-auths and confirms
10:00:22  Web customer pre-auths and confirms
Result:   Two confirmed reservations, two holds, one bike.
```
Needs a single reservation store with an atomic "reserve if no overlap" operation.

### 9.3 Suggested `cardReader` shape (shows I1, L3)
```python
startPayment(reservationId: ResId, amount: Money, idempotencyKey: Key)
    -> PaymentResult  # Authorized(authId) | Declined(reason) | Unknown | CardError
```
- The PIN is entered and encrypted on the card reader and never reaches the Session Controller.
- The name is not an input.
- `Unknown` triggers the "look up by idempotency key" path in the failure table.

### 9.4 Example failure-mode row (shows F2, F4)
| # | Failure | Detection | Response | Reservation state | Customer sees |
|---|---|---|---|---|---|
| 3 | Wifi lost after pre-auth succeeded | `wifi-status == false` and `authId` present | Retry connection for 45 s; local alarm to staff; on reconnect, void the auth if no confirmed reservation | `PendingVoid` | "Connection lost. Your card will not be charged. Please see the front desk." |

---

## 10. Recommended Order of Work

1. Resolve the payment model (L1) and kiosk role (L2).
2. Add the central reservation service to the architecture and website path (S1, S2, S3).
3. Add the missing entities and fix keys (D1–D4).
4. Rewrite `cardReader`, `bikeQuery`, and `logTransaction` (I1–I3).
5. Add the reservation state diagram and the missing operations (S6, I8).
6. Reconcile NFRs with the failure table (R1, F1–F6).
7. Run a spelling pass (§8).