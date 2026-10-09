Contents

1. [Endpoint](#endpoint)
2. [Authentication](#auth)
3. [Filters](#filters)
4. [Request builder](#builder)
5. [Pagination](#pagination)
6. [Response](#response)
7. [Record fields](#fields)
8. [Examples](#examples)
9. [Errors](#errors)

eWeighbridgeClient APIVersion 1

# Weighment Reports API

Fetch your weighment tickets as JSON. Filter by date, status, free text or any custom field your sites record, and page through the results.

Method

POST

Format

JSON

Auth header

`x-client-id`

Page size

100 default, 1000 max

Order

Newest first

## Endpoint

Request URL

```
POSThttps://workflow.kalpanix.com/webhook/eweighbridge/client/reports
```

Only `POST` is accepted. Send filters either in the query string or as a JSON body with `Content-Type: application/json`. You can mix the two; when the same filter appears in both, the query string value is used.

Every filter is optional. A request with no filters returns your most recent 100 weighments.

## Authentication

Send your client id in the `x-client-id` header on every request. Kalpanix issues this id, and it decides which weighbridges' data you receive.

Header

```
x-client-id: YOUR-CLIENT-ID
```

**Keep the client id secret.** Anyone who has it can read your weighment records. Call the API from your own server, and don't embed the id in a website or mobile app that runs on other people's devices. If it leaks, ask Kalpanix for a new one.

If the header is missing or the id isn't recognised, the request still succeeds but returns no records (`total_records: 0`). When you get an unexpectedly empty result, check the header first.

## Filters

All filters are combined with AND, so each one you add narrows the result. Results are always sorted by weighment date, newest first; tickets on the same date are ordered most recently created first.

`from`

OptionalDate

Earliest weighment date to include. The date is inclusive.

Format

`YYYY-MM-DD`

Example

`from=2026-09-01`

Matches on

`weighment_date`, the business date printed on the ticket

`to`

OptionalDate

Latest weighment date to include. The date is inclusive, so `to=2026-09-30` includes every ticket dated 30 September.

Format

`YYYY-MM-DD`

Example

`to=2026-09-30`

One day

Send the same date in both: `from=2026-09-15&to=2026-09-15`

`status`

OptionalText

Ticket status. Case-insensitive, so `completed` and `Completed` are the same.

CompletedBoth weighments recorded. Net weight is set.

PendingVehicle weighed once; the second weighment is still to come. Net weight is `null`.

CancelledTicket voided. The `cancellation` object says when, by whom and why.

Wildcard

`%` matches any text, e.g. `status=%cancel%`

`q`

OptionalText

Free-text search. Returns tickets where any of these contain the text, ignoring case:

- ticket number
- vehicle number
- party, station and operator names
- custom field names and values

Example

`q=OD 02` (URL-encoded: `q=OD%2002`)

Spaces

Matched exactly as stored. `OD 02` finds `OD 02 AA 3354`; `OD02` does not.

Wildcards

`%` matches any text and `_` any single character

`field`

OptionalTextPairs with value

Name of a custom field to filter on, such as `Material` or `Transporter`. Case-insensitive.

Custom fields are set up per site, so the names differ between customers. Look at `custom_fields` in any record to see the names your sites use.

Example

`field=Material`

`value`

OptionalTextPairs with field

The value that custom field must have. It matches the whole value, ignoring case: `value=coal` matches `Coal` but not `Coal Fines`.

Example

`field=Material&value=Coal`

Partial match

Add `%`: `value=Coal%` matches `Coal` and `Coal Fines`

Rule

Send `field` and `value` together. If only one of them is sent, both are ignored.

`page`

OptionalInteger

Which page of results to return, starting at 1.

Default

`1`

Limits

Values below 1 are treated as 1. A page past the last one returns an empty `records` list.

`pageSize`

OptionalInteger

How many records to return per page.

Default

`100`

Limits

1 to 1000. Larger values are reduced to 1000; check `page_size` in the response for the size actually used.

Spelling

Case-sensitive: `pageSize`, not `pagesize`

## Request builder

Fill in the filters you need and copy the command. Nothing is sent from this page.

`x-client-id`

`from`

`to`

`status`

`q`

`field`

`value`

`page`

`pageSize`

cURL

```

```

## Pagination

Each response tells you how many records matched (`total_records`) and how many pages that makes at your page size (`total_pages`). To fetch everything, request page 1, then keep increasing `page` until you reach `total_pages`.

Use the same filters and page size on every page. For large exports, use `pageSize=1000` to reduce the number of calls.

JavaScript (Node 18+)

```
const URL = "https://workflow.kalpanix.com/webhook/eweighbridge/client/reports";

async function fetchAll(filters) {
  const records = [];
  let page = 1, totalPages = 1;
  do {
    const res = await fetch(URL, {
      method: "POST",
      headers: {
        "Content-Type": "application/json",
        "x-client-id": process.env.EWB_CLIENT_ID,
      },
      body: JSON.stringify({ ...filters, page, pageSize: 1000 }),
    });
    if (!res.ok) throw new Error(`Request failed: ${res.status}`);
    const data = await res.json();
    records.push(...data.records);
    totalPages = data.total_pages;
    page++;
  } while (page <= totalPages);
  return records;
}

// All completed tickets for September 2026
const tickets = await fetchAll({ from: "2026-09-01", to: "2026-09-30", status: "Completed" });
```

## Response

A successful call returns HTTP `200` with one JSON object. `records` is an empty list when nothing matches.

| Field | Type | Description |
| --- | --- | --- |
| `page` | integer | Page returned, starting at 1 |
| `page_size` | integer | Records per page actually used (after the 1–1000 limit) |
| `total_records` | integer | Records matching your filters, across all pages |
| `total_pages` | integer | Pages at this page size; `0` when nothing matches |
| `generated_at` | string | When the response was produced, UTC, e.g. `2026-10-09T06:47:28Z` |
| `records` | array | The weighment tickets on this page. See [Record fields](#fields). |

Example response (illustrative values)

```
{
  "page": 1,
  "page_size": 100,
  "total_records": 1,
  "total_pages": 1,
  "generated_at": "2026-10-09T06:47:28Z",
  "records": [
    {
      "id": "3f2c9a10-5b7e-4c1d-9a8f-2e6b1d4c7a90",
      "ticket_no": "WB/15/09/100001",
      "status": "Completed",
      "weighment_date": "2026-09-15",
      "station": "Main Gate",
      "party": "Sample Traders",
      "shift": "Afternoon Shift",
      "operator": "R. Das",
      "vehicle": { "number": "OD 02 AA 3354", "type": "Truck" },
      "weights": {
        "entry_kg": 5000, "entry_at": "2026-09-15T16:46:28.897858+00:00",
        "gross_kg": 8000, "gross_at": "2026-09-15T16:52:47.775061+00:00",
        "tare_kg":  5000, "tare_at":  "2026-09-15T16:46:28.897858+00:00",
        "tare_source": "Weighed",
        "exit_kg":  8000, "exit_at":  "2026-09-15T16:52:47.775061+00:00",
        "net_kg": 2900
      },
      "turnaround_min": 6,
      "bags": { "type": { "id": "8d41e2b7-0c3a-4f6e-b5d2-91a7c3e0f4b8", "name": "Jute Bag" }, "count": 100, "deductionKg": 100 },
      "service_charge": { "amount": 50, "currency": "INR" },
      "company": {
        "id": "b7e4d2a1-6c9f-4e3b-8a0d-5f1c2e7b9a34",
        "name": "Sample Weighbridge Pvt Ltd",
        "address": "Plot 12, Industrial Estate, Angul, Odisha",
        "gstIn": "21AAAAA0000A1Z5",
        "licenseNo": "WB-1234"
      },
      "cancellation": null,
      "custom_fields": {
        "Material": "Coal",
        "Transporter": "Sample Logistics",
        "DO / Challan No": "150"
      },
      "created_at": "2026-09-15T16:46:36.617705+00:00",
      "updated_at": "2026-09-15T16:53:21.486932+00:00"
    }
  ]
}
```

## Record fields

Weights are in kilograms and returned as JSON numbers. Timestamps are ISO 8601 with a UTC offset (`+00:00`); convert them to your local time zone for display. Any field can be `null` when the site didn't record it.

### How the weights relate

**8,000**gross_kg

−

**5,000**tare_kg

−

**100**bags.deductionKg

\=

**2,900**net_kg

Gross is the loaded vehicle, tare the empty vehicle. `entry_kg` and `exit_kg` are the readings in the order they were taken: a vehicle bringing material in is heavier on entry, one taking material out is heavier on exit. Use `net_kg` as the billable weight.

| Field | Type | Description |
| --- | --- | --- |
| Ticket |  |  |
| `id` | string (uuid) | Unique, permanent id of the weighment. Use this as your key when storing records. |
| `ticket_no` | string | Ticket number printed on the weighment slip |
| `status` | string | `Completed`, `Pending` or `Cancelled` |
| `weighment_date` | string (date) | Business date of the ticket, `YYYY-MM-DD`. This is the date the `from` and `to` filters use. |
| `station` | string | Weighbridge station where the ticket was made |
| `party` | string | Customer or supplier on the ticket |
| `shift` | string | Shift the ticket was made in |
| `operator` | string | Weighbridge operator who made the ticket |
| Vehicle |  |  |
| `vehicle.number` | string | Registration number as entered, with spaces |
| `vehicle.type` | string | Vehicle type, e.g. Truck or Tractor |
| Weights |  |  |
| `weights.entry_kg` | number | First reading, when the vehicle came in |
| `weights.entry_at` | string (timestamp) | Time of the first reading |
| `weights.exit_kg` | number | Second reading, when the vehicle went out. `null` while the ticket is Pending. |
| `weights.exit_at` | string (timestamp) | Time of the second reading |
| `weights.gross_kg` | number | Loaded vehicle weight |
| `weights.gross_at` | string (timestamp) | When the gross weight was taken |
| `weights.tare_kg` | number | Empty vehicle weight |
| `weights.tare_at` | string (timestamp) | When the tare weight was taken |
| `weights.tare_source` | string | Where the tare came from, e.g. `Weighed` when the empty vehicle was put on the weighbridge |
| `weights.net_kg` | number | Final net weight: gross − tare − bag deduction. `null` until both weighments are done. |
| `turnaround_min` | integer | Minutes between first and second reading. `null` if either is missing. |
| Bags, charges and issuer |  |  |
| `bags` | object | `null` when no bags were declared |
| `bags.type.name` | string | Bag type, e.g. Jute Bag |
| `bags.count` | number | Number of bags |
| `bags.deductionKg` | number | Total weight deducted for the bags, already subtracted from `net_kg` |
| `service_charge` | object | Weighing fee: `amount` (number) and `currency` (e.g. `INR`). `null` if none. |
| `company` | object | Company that issued the ticket: `id`, `name`, `address`, `gstIn`, `licenseNo`. Can be `null` on older tickets. |
| Cancellation |  |  |
| `cancellation` | object | `null` unless the ticket was cancelled |
| `cancellation.at` | string (timestamp) | When it was cancelled. Sent without a time zone suffix, but the time is UTC. |
| `cancellation.by` | string | Who cancelled it |
| `cancellation.reason.text` | string | Reason given at cancellation |
| Custom fields |  |  |
| `custom_fields` | object | Extra fields your sites record, as `{ "Field name": value }`. Names differ between customers and can be added over time, so don't assume a fixed set. Values come back as entered, usually as strings (for example `"150"`). An empty object `{}` means none were recorded. |
| Record history |  |  |
| `created_at` | string (timestamp) | When the ticket was first saved |
| `updated_at` | string (timestamp) | Last change, e.g. second weighment, cancellation or an approved correction |

## Examples

### One month of completed tickets (query string)

cURL

```
curl -X POST \
  'https://workflow.kalpanix.com/webhook/eweighbridge/client/reports?from=2026-09-01&to=2026-09-30&status=Completed' \
  -H 'x-client-id: YOUR-CLIENT-ID'
```

### Coal tickets for one vehicle (JSON body)

cURL

```
curl -X POST 'https://workflow.kalpanix.com/webhook/eweighbridge/client/reports' \
  -H 'x-client-id: YOUR-CLIENT-ID' \
  -H 'Content-Type: application/json' \
  -d '{
    "from": "2026-09-01",
    "to": "2026-09-30",
    "field": "Material",
    "value": "Coal",
    "q": "OD 02 AA 3354"
  }'
```

### Cancelled tickets, second page of 50

cURL

```
curl -X POST \
  'https://workflow.kalpanix.com/webhook/eweighbridge/client/reports?status=Cancelled&page=2&pageSize=50' \
  -H 'x-client-id: YOUR-CLIENT-ID'
```

### Python

requests

```
import os, requests

resp = requests.post(
    "https://workflow.kalpanix.com/webhook/eweighbridge/client/reports",
    headers={"x-client-id": os.environ["EWB_CLIENT_ID"]},
    json={"from": "2026-09-01", "to": "2026-09-30", "pageSize": 1000},
    timeout=60,
)
resp.raise_for_status()
data = resp.json()
print(data["total_records"], "tickets")
for t in data["records"]:
    print(t["ticket_no"], t["vehicle"]["number"], t["weights"]["net_kg"])
```

## Errors and empty results

| What you see | Likely cause | What to do |
| --- | --- | --- |
| `200`, `total_records: 0` | No ticket matches the filters, or the `x-client-id` header is missing or wrong | Check the header, then widen the filters (dates, status, spelling of `field`) |
| `200`, empty `records`, `total_records` above 0 | `page` is past the last page | Request a page from 1 to `total_pages` |
| `500` | A filter has the wrong format, e.g. `from=01-09-2026` or `page=two` | Send dates as `YYYY-MM-DD` and `page` / `pageSize` as whole numbers |
| `404` | Wrong URL, or a method other than POST | Use `POST` and the exact URL above |

If a request keeps failing with valid filters, send Kalpanix support the time of the request and the filters you used. Don't include your client id in emails or tickets.

eWeighbridge Weighment Reports API · Version 1 Kalpanix