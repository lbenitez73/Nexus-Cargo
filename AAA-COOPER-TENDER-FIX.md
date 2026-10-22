# AAA Cooper — rates work, tenders don't: diagnosis and fix

For: StreamLine TMS, code at `~/TMS` on Laz's Mac (`apps/api`).
Run this from a Claude session opened in `~/TMS`. Paste this file in, or point the session at it.

---

## 1. What is actually happening

The auto-dispatch code (`apps/api/src/services/dispatch.ts`, `autoDispatchShipment`)
chooses how to hand over a booked load based on **where the accepted rate came from**
(`providerForRef(selectedQuoteRef)`):

| Rate source | `provider` | What booking does |
|---|---|---|
| Worldwide Express (WWEX) | `WWEX` | API booking with WWEX |
| Priority1 | `Priority1` | API booking with Priority1 |
| XPO direct | carrier adapter | BOL API + Pickup Request API (`integrations/carriers/xpo.ts`) |
| **AAA Cooper direct** | **`none`** | **`tenderLoad()`, which emails a rate confirmation and makes no API call** |

So when the AAA Cooper rate comes from your own AAA Cooper rating connection
(account **C# 1167251**), booking never calls AAA Cooper. No pickup request is
created on their side and no BOL is sent electronically. The only thing that goes
out is an email. AAA Cooper does not dispatch a driver from an emailed rate
confirmation.

There is a second way the same symptom shows up. If the AAA Cooper rate came
through **WWEX**, the load is booked with WWEX, not with AAA Cooper on your
account. It will not appear in your AAA Cooper portal under C# 1167251, and WWEX
can substitute the carrier. A past session saw an AAA Cooper offer come back as
Central Transport.

### Check the load from today first (SK-10024, "Pickup # 71785468")

1. Open SK-10024 in StreamLine and read its status log:
   - "Dispatched to AAA Cooper via Worldwide Express …" means WWEX booked it. The
     pickup number is WWEX's or AAA Cooper's under WWEX's account. Check it with WWEX.
   - A rate-confirmation / tender entry and no "accepted the load" means AAA Cooper
     was **never told**. The "Pickup #" on the email did not come from an AAA Cooper
     pickup API. Find out which field it was copied from (most likely the rate
     quote number). **Call AAA Cooper (customer care 800-633-7571, or your rep
     Ryleigh Lanham) and set the pickup up by hand today.**
2. In the AAA Cooper portal (aaacooper.com → Workspace → Pickup), look up 71785468
   under account 1167251.

### Confirmed 2026-10-22: SK-10024 went out through WWEX, but WWEX is supposed to be off

The SK-10024 log reads "Dispatched to AAA Cooper via Worldwide Express". So the
"AAA Cooper" rate on that quote was a **WWEX rate** with a quote ref of the form
`WWEX:<…>:<…>:AACT`, not a rate from your own AAA Cooper account. Two gaps let
that happen:

1. **Rating still returns WWEX offers.** Whatever "temporarily off" sets today, at
   least one rating path ignores it. The broker quote screen, the customer
   portal's re-rate at booking and the mass rater are all candidates. The portal
   re-rates at booking and passes `quoteRef` straight into `autoDispatchShipment`.
2. **Dispatch never checks whether WWEX is enabled.** `dispatchBookedLoad` in
   `services/dispatch.ts` routes on the quote-ref prefix alone
   (`providerForRef(ref) === "WWEX"`, then `wwexBook(...)`). Any `WWEX:` ref gets
   booked with WWEX as long as credentials are still saved, toggle or no toggle.

**Code fix (do this before the AAA Cooper adapter):**
- A single helper, e.g. `isAggregatorEnabled("WWEX")`, read from settings. One
  source of truth for "WWEX is off".
- **Rating:** every path that calls WWEX checks the helper and skips WWEX when it
  is off. That covers the quote screen, the portal (including the re-rate at
  booking), the mass rater, and any cached or saved quotes being re-offered.
- **Dispatch guard:** in `dispatchBookedLoad` and `autoDispatchShipment`, if
  `provider === "WWEX"` and WWEX is off, **do not call `wwexBook`**. Log
  "NOT dispatched — Worldwide Express is turned off; re-rate this load", set
  `dispatchError`, and return `dispatched: false`. Do **not** fall back to
  `tenderLoad()` on a WWEX-priced rate, because that tenders the carrier at a
  price that isn't yours.
- **Tests:** with WWEX off, a `WWEX:…:AACT` ref must never reach `wwexBook`, and
  rating must return no WWEX offers on any path.

**SK-10024 itself:** it is live with Worldwide Express, under their account, at
their rate. If it should move under your own AAA Cooper account (C# 1167251),
**cancel it with WWEX first**, then book directly with AAA Cooper. Two open
bookings mean two trucks.

---

## 2. Credentials to collect from the AAA Cooper portal (Chrome)

1. Sign in at aaacooper.com → **Workspace** with the web login linked to account
   **1167251**.
2. Go to **Web Services** (site footer, or `aaacooper.com/WS/WebServiceRequest.aspx`).
3. Tick every service: **Rate Estimate, Pickup, Tracking, Imaging, Transit Time**
   (and **Bill of Lading / Shipment** if it is listed). The existing token may only
   cover Rate Estimate. That would explain why rating works and nothing else does.
4. Submit. AAA Cooper emails the **API Token** to the account's primary contact.
   Tokens belong to one web user and cannot be transferred, so request it from the
   login StreamLine should act as.
5. Open each service's **Web Service Details** page
   (`aaacooper.com/WS/WebServiceDetails.aspx?service=Pickup`, and the same for the
   BOL service) and **download the WSDL and sample request/response XML**. The code
   must be built against those files. Do not guess field names.

Values StreamLine needs:

| Setting | Value |
|---|---|
| API token | (from the email) |
| Account / BusID | `1167251` |
| Pickup endpoint | `http://wsportal.aaacooper.com:8188/wsportal20/wsPickUpRq` (confirm against the WSDL; use https if the WSDL offers it) |
| Rate endpoint | the one already in use (`wsGenRateEstimate`) |

---

## 3. Code change (for the Claude session on the Mac)

Model it on the XPO direct integration (`integrations/carriers/xpo.ts`,
`__tests__/xpoDispatch.test.ts`), which already does "BOL, then Pickup Request"
after booking.

1. **New adapter `apps/api/src/integrations/carriers/aaaCooper.ts`** implementing the
   same dispatch interface in `carriers/types.ts`:
   - `requestPickup(load)` sends a SOAP 1.1 POST to `wsPickUpRq`.
     `Content-Type: text/xml; charset=ISO-8859-1` (the service expects ISO-8859-1,
     so transliterate or strip non-Latin-1 characters in names and addresses).
     The request element is `wsPickUpRqRequest`. Fill it from the WSDL using the
     token, account 1167251, shipper name/address/contact/phone, pickup date,
     ready time and dock-close time, handling units, pieces, weight, class,
     hazmat flag, destination ZIP, and the BOL / PO reference.
   - Parse the response: **pickup request number**, **error flag**, **error message**.
     If the error flag is set or the number is missing, treat it as a failure. It
     is never a success.
   - If AAA Cooper exposes a BOL / shipment-create service on the account, call it
     **before** the pickup (same pattern as XPO) and store the PRO it returns.
2. **Route AAA Cooper through the adapter.** Where `provider === "none"` today,
   check whether the carrier is AAA Cooper (SCAC `AACT`) with credentials saved.
   If it is, call the adapter. If it isn't, fall back to `tenderLoad()`. Only call
   `markDispatched()` after AAA Cooper returns a pickup number. Write the pickup
   number with `writeBack()` and log "AAA Cooper accepted the load. Pickup ####."
3. **Failure must be loud.** On any error, use the existing pattern: log "NOT
   dispatched to AAA Cooper — <their error>", set `dispatchError`, then fall back
   to `tenderLoad()`. **Never put a quote number or any made-up value in
   `pickupNumber`**, and never send the "Your shipment is booked" email with a
   pickup number unless AAA Cooper actually returned it.
4. **Settings → Carrier connections → AAA Cooper.** Add fields for the token,
   account 1167251 and the pickup endpoint, plus a "Test pickup connection" button.
   It should send a request AAA Cooper rejects harmlessly (for example a past date)
   and show their raw error, so you can confirm the token is authorized for Pickup.
5. **Tests** in `__tests__/aaaCooperDispatch.test.ts`, mirroring
   `xpoDispatch.test.ts`:
   - The request body matches the WSDL sample.
   - Encoding is ISO-8859-1.
   - An error-flag response gives `dispatched: false` and a `dispatchError`.
   - A success response writes the pickup number and stamps `dispatchedAt`.
   - A WWEX-sourced AAA Cooper rate still goes through WWEX.
6. **Mail-loop guard.** AAA Cooper auto-replies are already handled (memory note
   "Mail loop guards"). Keep the RFC 3834 headers on any fallback tender email.

### Done means
- Book a test load on an AAA Cooper direct rate. The status log shows "AAA Cooper
  accepted the load. Pickup #…".
- That pickup number appears in the AAA Cooper portal under account 1167251.
- A deliberately bad token shows "NOT dispatched" on the load and falls back to
  the tender email, with no fake pickup number.

---

Sources: AAA Cooper Web Services (aaacooper.com/web-services), Pickup service
details (aaacooper.com/WS/WebServiceDetails.aspx?service=Pickup), project44 AACT
setup notes, 3PL Systems "AAA Cooper: API Dispatching Documentation" (wsPickUpRq
endpoint, request/response fields), V-Technologies StarShip AAA Cooper setup.
