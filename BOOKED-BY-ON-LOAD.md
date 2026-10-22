# Show who booked a load, on the load itself (internal only)

For: StreamLine TMS, code at `~/TMS` on Laz's Mac (`apps/api` + the web app).
Run this from a Claude session opened in `~/TMS`. Paste this file in, or point the session at it.

---

## 1. Goal

When a load is booked, staff opening that load must see **who booked it**:

- Person's name, email and phone
- Their company: our staff, or the customer account they belong to
- How they booked it: broker quote screen, customer portal, mass rater, API/EDI, or auto-dispatch
- When they booked it

This is for **internal use only**. It must not appear on the BOL, the rate
confirmation or tender email, carrier emails, the customer tracking page, or
customer-portal API responses.

## 2. Before changing anything, find out what exists

Report back on each of these before you write code:

1. The load/shipment model and its migration folder. Is there already a
   `createdBy`, `bookedBy`, `userId` or similar column? Is it actually filled in?
2. **Every** place a load moves to "booked". The known ones are:
   - the broker quote screen's book action
   - the customer portal's book action (it re-rates at booking and calls
     `autoDispatchShipment` with `quoteRef`)
   - the mass rater
   - any API, EDI or webhook booking path
   - `autoDispatchShipment` / `dispatchBookedLoad` in `apps/api/src/services/dispatch.ts`
   Use grep to find any others.
3. How each of those paths knows who the current user is (session or auth
   middleware, `req.user`, API key, and so on).
4. How the status log is written (the helper that produces lines like
   "Dispatched to AAA Cooper via …").
5. Which serializers or DTOs send load data to customer-portal users, carriers
   and public tracking pages.

## 3. Data model

Add these to the load (adjust the names to match the codebase):

| Field | Type | Notes |
|---|---|---|
| `bookedByUserId` | FK → users, nullable | null for API/EDI/system bookings |
| `bookedAt` | timestamp | set once, at the moment the load becomes booked |
| `bookedVia` | enum | `BROKER_UI`, `CUSTOMER_PORTAL`, `MASS_RATER`, `API`, `EDI`, `SYSTEM` |
| `bookedBySnapshot` | JSON | `{ name, email, phone, company, role, apiKeyName? }` copied at booking time |
| `bookedOnBehalfOf` | JSON, nullable | the customer contact, when staff book for a customer |

The snapshot matters because the load has to keep showing the right person if
that user is later renamed, deactivated or deleted.

**Migration and backfill:** for existing booked loads, fill the fields from the
status log or audit trail where possible. Where that isn't possible, leave the
fields null. The UI then shows "Not recorded (booked before this feature)".
Never guess a person.

## 4. Capturing it

- Write a single helper, for example `recordBooking(load, actor, via)`, and
  call it from **every** booking path found in step 2.2. Call it inside the
  same transaction that sets the booked status.
- Take the actor **only** from the authenticated session or API key on the
  server. Never take it from the request body, because a portal user could
  otherwise send someone else's name.
- Customer portal bookings: the actor is the logged-in portal user, and
  `company` is their customer account.
- Staff booking for a customer: the actor is the staff member.
  `bookedOnBehalfOf` holds the customer contact picked on the screen.
- API/EDI/system bookings: there is no user. Set `apiKeyName` (or the
  integration name) in the snapshot and use `bookedVia = API/EDI/SYSTEM`.
- Set the fields once. Re-rating, re-dispatching or editing the load must not
  overwrite them. If a load is cancelled and re-booked, keep the original and
  record the re-booking as its own status-log entry.
- Add a status-log line at booking, for example:
  "Booked by Jane Doe (jane@acme.com, 555-0100) — Acme Corp — via Customer Portal".

## 5. Showing it (internal users only)

- **Load detail page:** add a "Booked by" card in the header area. Show the
  name, a mailto link on the email, a tel link on the phone, the company,
  `bookedVia`, and `bookedAt` in local time. Show "on behalf of …" when that
  field is set.
- **Load list/board:** add an optional "Booked by" column and a filter
  ("loads booked by X", "loads booked via portal").
- **Access control:** return these fields only to internal roles. Strip them
  in the serializers for customer-portal users, carriers and public tracking
  (step 2.5). Do this on the server. Hiding them in the UI is not enough.
- Keep them out of BOL, rate confirmation, tender and "Your shipment is booked"
  templates.

## 6. Tests

- Each booking path (broker UI, portal, mass rater, API) stores the correct
  actor, `bookedVia` and `bookedAt`.
- An actor name in the request body is ignored. The session user wins.
- Re-rating or re-dispatching a booked load leaves `bookedBy*` unchanged.
- Customer-portal, carrier and public-tracking responses do **not** contain
  any `bookedBy*` field. Internal responses do.
- The snapshot still shows the original name after the user is renamed or
  deleted.
- The BOL and rate confirmation output contains no booker details.

## Done means

- Book a test load from the customer portal and another from the broker
  screen. Each load's detail page shows the correct person, contact details,
  company and channel, and the status log has the "Booked by …" line.
- Logged in as that customer in the portal, the same load shows nothing about
  who booked it, and the portal API response has no `bookedBy*` fields.
- Older loads show "Not recorded" instead of a wrong name.
