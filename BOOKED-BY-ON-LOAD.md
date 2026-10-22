# Show who booked a load, on the load itself (staff, and the customer's own team)

For: StreamLine TMS, code at `~/TMS` on Laz's Mac (`apps/api` + the web app).
Run this from a Claude session opened in `~/TMS`. Paste this file in, or point the session at it.

---

## 1. Goal

When a load is booked, staff opening that load must see **who booked it**:

- Person's name, email and phone
- Their company: our staff, or the customer account they belong to
- How they booked it: broker quote screen, customer portal, mass rater, API/EDI, or auto-dispatch
- When they booked it

**Customers see who on their own team booked it.** A customer-portal user
viewing one of their account's loads sees the booker's name, email, phone,
channel and time, but only when the booker belongs to that same customer
account. When our staff booked the load for them, the customer sees "Booked by
<our company name> on behalf of <their contact>". The staff member's name and
contact details stay internal.

**Nobody outside gets it.** The booker's details must not appear on the BOL, the
rate confirmation or tender email, carrier emails, or the public tracking page.

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

## 5. Showing it

### Internal (staff) app

- **Load detail page:** add a "Booked by" card in the header area. Show the
  name, a mailto link on the email, a tel link on the phone, the company,
  `bookedVia`, and `bookedAt` in local time. Show "on behalf of …" when that
  field is set.
- **Load list/board:** add an optional "Booked by" column and a filter
  ("loads booked by X", "loads booked via portal").

### Customer portal

- **Load detail page:** add the same "Booked by" card, filled in by the rules
  below.
- **Load list:** add an optional "Booked by" column and a "Booked by" filter
  limited to users on the customer's own account, for example "show loads Jane
  booked".
- What the portal shows depends on who booked the load. The server decides
  this, using the **booker's** customer account compared with the **viewer's**
  customer account:

  | Who booked it | What the portal user sees |
  |---|---|
  | A portal user on the **same** customer account | Name, email, phone, channel (Portal / API), time |
  | Our staff, with `bookedOnBehalfOf` set | "Booked by <our company name> on behalf of <contact name>", time |
  | Our staff, no on-behalf-of contact | "Booked by <our company name>", time |
  | API key / EDI belonging to that customer | The integration name ("Acme EDI"), time |
  | Not recorded (older load) | "Not recorded" |

  Never send the staff member's name, email or phone to the portal.
- Make sure a portal user can only ever load their own account's loads. Check
  the existing scoping instead of assuming it works. Booker details plus a
  scoping gap would leak one customer's people to another.
- **Status log:** if the portal shows the load's status log, the internal
  "Booked by …" line must follow the same rules. Write a portal-safe version
  of that line, or hide it from the portal.

### Access control (server side)

- Build one function, for example `bookedByForViewer(load, viewer)`, that
  returns the full record for internal roles, the filtered view above for
  portal users, and `null` for anyone else. Every serializer uses it. Hiding
  fields in the UI is not enough.
- Strip all `bookedBy*` fields from carrier and public-tracking responses
  (step 2.5).
- Keep the booker's details out of the BOL, rate confirmation, tender, carrier
  emails and the public tracking page.
- The "Your shipment is booked" email goes to the customer. It may name the
  booker when that person is on the customer's own team, and must never name
  our staff.

## 6. Tests

- Each booking path (broker UI, portal, mass rater, API) stores the correct
  actor, `bookedVia` and `bookedAt`.
- An actor name in the request body is ignored. The session user wins.
- Re-rating or re-dispatching a booked load leaves `bookedBy*` unchanged.
- Internal responses contain the full `bookedBy*` record.
- Portal, load booked by a teammate on the same account: the response shows
  their name, email, phone, channel and time.
- Portal, load booked by our staff: the response shows our company name (plus
  "on behalf of" when set). It does **not** contain the staff member's name,
  email, phone or user ID.
- Portal user from customer A requesting customer B's load: rejected (404/403),
  with no booker details in the error.
- Portal status log contains no staff booker details.
- Carrier and public-tracking responses contain no `bookedBy*` field.
- The snapshot still shows the original name after the user is renamed or
  deleted.
- The BOL and rate confirmation output contains no booker details.

## Done means

- Book a test load from the customer portal and another from the broker
  screen. Each load's detail page shows the correct person, contact details,
  company and channel, and the status log has the "Booked by …" line.
- Logged in as a **different** user on that customer's account, the
  portal-booked load shows the teammate who booked it, with their contact
  details.
- In the same portal login, the broker-booked load shows "Booked by <our
  company name> on behalf of …" and no staff name, email or phone, either on
  screen or in the API response.
- A user from another customer account can't open either load.
- Older loads show "Not recorded" instead of a wrong name.
