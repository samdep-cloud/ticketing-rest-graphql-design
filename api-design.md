# Ticketing Service API Design

## Internal REST Service Design 
| Method | Path | Query params | Request body | Response shape | Success code | Error codes | Notes |
| --- | --- | --- | --- | --- | --- | --- | --- |
| GET | `/events/{eventId}` | None | None | [Event](#event) | `200 OK` | [`404 - EVENT_NOT_FOUND`](#error-codes) | **S1:** Supplies the event details (title, description, start time) and the IDs for the venue and organizer lookups. |
| GET | `/venues/{venueId}` | None | None | [Venue](#venue) | `200 OK` | [`404 - VENUE_NOT_FOUND`](#error-codes) | **S1:** Supplies the venue name and full address. |
| GET | `/organizers/{organizerId}` | None | None | [Organizer](#organizer) | `200 OK` | [`404 - ORGANIZER_NOT_FOUND`](#error-codes) | **S1:** Supplies the organizer’s name. |
| GET | `/events/{eventId}/tiers` | None | None | [Tier[]](#tier) | `200 OK` | None | **S1:** Supplies ticket tiers, prices, and remaining ticket count.<br>**S5:** Supplies current ticket count from the Inventory Service. |
| GET | `/events` | `city={city}&start_date={start_date}&end_date={end_date}&limit={limit}` | None | [Event[]](#event) | `200 OK` | [`400 - INVALID_QUERY`](#error-codes) | **S2:** Supplies event titles, start times, and IDs for the mobile browse list. |
| GET | `/tiers` | `eventIds={comma-separated IDs}&lowestPricePerEvent=true` | None | [Tier[]](#tier) | `200 OK` | [`400 - INVALID_QUERY`](#error-codes) | **S2:** Supplies the lowest tier price for each event in one batch. |
| POST | `/events/{eventId}/holds` | None | `{"tierId": "tier_123", "quantity": 2}` | [Hold](#hold) | `201 Created` | [`400 - INVALID_HOLD_REQUEST`](#error-codes)<br>[`404 - TIER_NOT_FOUND`](#error-codes)<br>[`409 - NUM_TICKETS_UNAVAILABLE`](#error-codes) | **S3:** Reserves tickets in the attendee’s selected tier. |
| GET | `/v1/orders` | `createdAtFrom={timestamp}&limit={limit}&cursor={cursor}` | None | [OrderPage](#orderpage) | `200 OK` | [`400 - INVALID_LIMIT`](#error-codes)<br>[`400 - INVALID_CREATED_AT_FROM`](#error-codes)<br>[`400 - INVALID_CURSOR`](#error-codes) | **S4:** Supplies orders for the partner’s nightly sync. |

#### Data Requirements and Expected Behavior

**Event detail — `GET /events/{eventId}`, `GET /venues/{venueId}`, and `GET /organizers/{organizerId}`**

- Match each path ID to the corresponding record’s `id`. Return the full record on a match and the endpoint’s `404` error when there’s no match.
- The gateway checks the Catalog Service for the event first. If the event exists, it uses `venueId`, `organizerId`, and the `eventId` to fetch the related records and assemble one response for the detail screen. If the event is missing, it stops with `EVENT_NOT_FOUND`.
- Every seeded event must reference an existing `venueId` and `organizerId` in the Catalog Service.

**Event tiers and live availability — `GET /events/{eventId}/tiers`**

- Return all tiers that match the `eventId`. Keep sold-out tiers in the list with `quantityRemaining: 0`.
- If no tiers match, return `200 OK` with an empty list `[]`. The Catalog Service handles `EVENT_NOT_FOUND` in the event detail flow.
- When the app requests `tiers.quantityRemaining`, the gateway makes a fresh call to the Inventory Service and uses the counts from that response to ensure the client receives the current count of availability.
- Every seeded ticket tier must have an associated `eventId` that matches an existing event in Catalog.

**Mobile browse — `GET /events`**

- Require a non-blank `city` and both dates. Trim spaces from the city and match irrespective of case. 
- Accept `start_date` and `end_date` as calendar dates in YYYY-MM-DD format. The `end_date` must be on or after the `start_date`. Convert each event’s UTC `startsAt` to the event city’s local timezone before comparing dates. Include both endpoints of the selected date range. For Tampa, use America/New_York.
- Default `limit` to 20 and accept whole numbers from 1 to 20. Sort by `startsAt` and then `id` before applying the limit.
- Check city, dates, and limit in that order. Return `400 INVALID_QUERY` with a message explaining the first issue the caller needs to fix.
- Return `200 OK` with however many events match, up to the limit. A search with no matches, including a typo like `Bston`, returns `[]`.

**Batch tier prices — `GET /tiers`**

- Accept the event IDs as a comma-separated list. Return `400 INVALID_QUERY` for missing or malformed `eventIds` or an invalid `lowestPricePerEvent` value.
- If `lowestPricePerEvent` flag is omitted, empty, or false, return all tiers for the requested events. If it is true, return the full tier record with the lowest `priceCents` for each requested event that has tiers.

**Ticket hold — `POST /events/{eventId}/holds`**

- The gateway checks the Catalog Service first and stops if it returns `EVENT_NOT_FOUND`.
- The Inventory Service requires a non-empty `tierId` and a positive integer `quantity`. Return `400 INVALID_HOLD_REQUEST` for malformed JSON or invalid body fields. If the tier can’t be found under the supplied `eventId`, return `404 TIER_NOT_FOUND`.
- Check current availability when the attendee requests the hold. If enough seats remain, reserve them for 10 minutes, reduce `quantityRemaining` and return the hold with `expiresAt`. Tickets from an an incomplete hold become available again when the hold expires.
- If fewer seats remain than what's requested, return `409 NUM_TICKETS_UNAVAILABLE` with the current `quantityRemaining`. Create no hold and leave availability unchanged. The user would need to decide on the adjustment - the service does not reserve fewer tickets than what's been requested.

**Partner nightly sync — `GET /v1/orders`**

- Require `createdAtFrom`. Include orders created at or after that timestamp and sort by `createdAt`, then `id`.
- Default `limit` to 100 and accept whole numbers from 1 to 100.
- The partner leaves `cursor` out of the first request. For each page after that, it keeps `createdAtFrom` the same and passes the previous response’s `nextCursor` as `cursor`. When `nextCursor` is `null`, the partner stop making requests.

### Response Shapes

Each REST endpoint will return all the fields defined in its response schema, so the same response can support different use cases. The GraphQL gateway will add an additional layer of curation, returning only the fields the cleint ultimately requests.

#### Event
```typescript
interface Event {
id: string;
title: string;
description: string;
startsAt: string;
city: string;
venueId: string;
organizerId: string;
status: string;
}
```
#### Venue
```typescript
interface Venue {
id: string;
name: string;
address: string;
city: string;
capacity: number;
}
```
#### Organizer
```typescript
interface Organizer {
id: string;
name: string;
verified: boolean;
}
```
#### Tier
```typescript
interface Tier {
id: string;
eventId: string;
name: string;
priceCents: number;
quantityTotal: number;
quantityRemaining: number;
}
```
#### Hold
```typescript
interface Hold {id: string;
tierId: string;
quantity: number;
expiresAt: string;
}
```
#### Order
```typescript
interface Order {
id: string;
eventId: string;
tierId: string;
quantity: number;
totalCents: number;
buyerEmail: string;
createdAt: string;
}
```
#### OrderPage

```typescript
interface OrderPage {
  orders: Order[];
  nextCursor: string | null;
}
```

### Error Codes

| HTTP status | Error code | When it happens | Message |
| --- | --- | --- | --- |
| `400 Bad Request` | `INVALID_QUERY` | `city` is missing or blank. | Provide a non-empty city query parameter. |
| `400 Bad Request` | `INVALID_QUERY` | `start_date` or `end_date` is missing or invalid. | Provide start_date and end_date as valid YYYY-MM-DD dates. |
| `400 Bad Request` | `INVALID_QUERY` | The end date is before the start date. | End date must be on or after start date. |
| `400 Bad Request` | `INVALID_QUERY` | The browse limit is invalid. | The limit must be an integer from 1 to 20. |
| `400 Bad Request` | `INVALID_QUERY` | `eventIds` is missing or malformed. | Provide event IDs as a comma-separated list of non-empty event IDs. |
| `400 Bad Request` | `INVALID_QUERY` | `lowestPricePerEvent` is missing or invalid. | Provide lowestPricePerEvent as true or false. |
| `400 Bad Request` | `INVALID_HOLD_REQUEST` | The hold request body is malformed or is not a JSON object. | Provide a JSON object with tierId and quantity. |
| `400 Bad Request` | `INVALID_HOLD_REQUEST` | `tierId` is missing, blank, or the wrong type. | Provide tierId as a non-empty string. |
| `400 Bad Request` | `INVALID_HOLD_REQUEST` | `quantity` is missing or is not a positive integer. | Ticket quantity is required and must be a positive integer. |
| `400 Bad Request` | `INVALID_LIMIT` | The order page limit is invalid. | The limit must be an integer from 1 to 100. |
| `400 Bad Request` | `INVALID_CREATED_AT_FROM` | `createdAtFrom` is missing or invalid. | createdAtFrom must be a valid UTC timestamp, such as 2026-11-20T08:00:00Z. |
| `400 Bad Request` | `INVALID_CURSOR` | A supplied cursor is invalid or belongs to a different `createdAtFrom` value. | Use the nextCursor from the previous response with the same createdAtFrom value. |
| `404 Not Found` | `EVENT_NOT_FOUND` | No event matches the supplied ID. | The event cannot be found. |
| `404 Not Found` | `VENUE_NOT_FOUND` | No venue matches the supplied ID. | The venue cannot be found. |
| `404 Not Found` | `ORGANIZER_NOT_FOUND` | No organizer matches the supplied ID. | The organizer cannot be found. |
| `404 Not Found` | `TIER_NOT_FOUND` | The selected tier cannot be found under the supplied event ID. | The ticket tier cannot be found. |
| `409 Conflict` | `NUM_TICKETS_UNAVAILABLE` | The requested quantity exceeds the seats remaining in the selected tier. | The selected ticket quantity is not available. |

#### Standard error response

```json
{
  "error": {
    "code": "EVENT_NOT_FOUND",
    "message": "The event cannot be found."
  }
}
```

#### Insufficient tickets response

Inventory also returns the actual number of seats remaining in the selected tier.

```json
{
  "error": {
    "code": "NUM_TICKETS_UNAVAILABLE",
    "message": "The selected ticket quantity is not available.",
    "quantityRemaining": 2
  }
}
```

### Key Decisions:

### 1. Pagination (S4): 
I chose cursor pagination because a nightly sync can include up to 50,000+ orders. The partner sends `createdAtFrom` on the first request, and the API returns up to 100 orders per page by default. For each subsequent page, the partner keeps the cutoff unchanged and sends the previous response’s `nextCursor` as `cursor`. The sync ends when `nextCursor` is null. Sorting by `createdAt` and then by`id` (PK) gives every order a consistent position, even when several orders share the same timestamp.

### 2. Batch Fetching (S2): 
The gateway sends up to 20 event IDs to the Inventory Service in one request. The Inventory Service then returns the cheapest tier for each event, and the gateway matches the results by `eventId`. The mobile app just displays the event title, start time, and lowest tier price.

### 3. Versioning:
I versioned the partner route as /v1/orders so the venue’s existing sync can keep running while partners migrate to /v2/orders if a breaking change is introduced at some point. I left internal routes unversioned because the platform controls both the gateway and the services and a change can be deployed more seamlessly in that case.

### 4. Data Freshness (S5):
The gateway can cache the Catalog Service data for up to a minute - including title and event address. However, available ticket counts are refreshed directly from the Inventory Service each and every time the app requests them to ensure the most accurate count is shown. The Inventory Service also checks again when an attendee requests a hold because availability can change after the page loads.

### GraphQL Schema 

```graphql
type Query {
  event(id: ID!): Event
  browseEvents(city: String!, start_date: String!, end_date: String!): [BrowseEvent!]!
}

type Mutation {
  holdTickets(eventId: ID!, tierId: ID!, quantity: Int!): TicketHold
}

type Event {
  title: String!
  description: String!
  startsAt: String!
  venue: Venue
  organizer: Organizer
  tiers: [Tier!]
}

type Venue {
  name: String!
  address: String!
  city: String!
}

type Organizer {
  name: String!
}

type Tier {
  id: ID!
  name: String!
  priceCents: Int!
  quantityRemaining: Int!
}

type BrowseEvent {
  title: String!
  startsAt: String!
  lowestTierPriceCents: Int
}

type TicketHold {
  id: ID!
  tierId: ID!
  quantity: Int!
  expiresAt: String!
}
```

### 1. What does the gateway do that the services do not? 
The gateway brings together data from the Catalog and Inventory services based on what each screen needs. For event detail, it uses the IDs from the event record to look up the venue, organizer, and ticket tiers, then sends everything back in one response. For mobile browse, it matches each event to its lowest-priced tier by `eventId` and returns just the title, start time, and lowest price. Partners only call the Orders Service directly through /v1/orders for their nightly sync, so no need to "compose" a response in this case. 

### 2. Resolver mapping — Which REST calls does each GraphQL field trigger?
The gateway starts each query with the Catalog Service and then fetches related data when the app requests those fields.

| GraphQL field | Service and REST call | How the gateway uses it |
| --- | --- | --- |
| `event(id)` → `title`, `description`, `startsAt` | Catalog · `GET /events/{eventId}` | Looks up the supplied `id` and keeps `venueId` and `organizerId` for related lookups. |
| `event.venue` → `name`, `address`, `city` | Catalog · `GET /venues/{venueId}` | Uses the event’s `venueId`. |
| `event.organizer` → `name` | Catalog · `GET /organizers/{organizerId}` | Uses the event’s `organizerId`. |
| `event.tiers` → `id`, `name`, `priceCents`, `quantityRemaining` | Inventory · `GET /events/{eventId}/tiers` | Fetches all tiers in one call. Seat counts come from a fresh Inventory read, and the tier `id` is used to request a hold. |
| `browseEvents(...)` → `title`, `startsAt` | Catalog · `GET /events` | Sends the city and date range with `limit=20` and keeps the event IDs for the price lookup. |
| `browseEvents(...).lowestTierPriceCents` | Inventory · `GET /tiers?eventIds=…&lowestPricePerEvent=true` | If prices are requested and events match, sends the IDs in one batch. Matches prices by `eventId`. Events with no tiers get `null`. |
| `holdTickets(...)` → `TicketHold` | Catalog · `GET /events/{eventId}`<br>Inventory · `POST /events/{eventId}/holds` | Checks Catalog first and stops on `EVENT_NOT_FOUND`. Otherwise, sends `tierId` and `quantity` to Inventory and returns the hold on `201`. On `409`, returns `data.holdTickets: null` with `extensions.code: NUM_TICKETS_UNAVAILABLE` and `extensions.quantityRemaining`. |


### 3. Why does the partner not get a GraphQL endpoint?

The partner’s script has one main, predictable job - to pull orders created since their last sync. The ticket app, on the other hand, uses GraphQL because each screen needs a different set of fields drawn from both the Catalog and Inventory Services. 

### Consumer Table

| Feature | Attendee apps | Partners |
| --- | --- | --- |
| Ship cadence | App teams can update screens as needed. | Partners can also update on their own schedule. |
| Data needs | It depends on the screen. Event detail needs venue and organizer info, ticket tiers, and live seat counts. Mobile browse only needs the title, start time, and lowest tier price. | The nightly sync needs all orders created since the last sync, up to 50K, in pages that come back in the same format, same fields. |
| Who writes the query? | The app team writes the GraphQL query and picks the fields each screen needs. The gateway code determines which REST service to call to fetch those fields. | The partner sends the timestamp and pagination parameters. The Orders service controls the database query and which fields are returned. |
| Contract stability | Fields can be added to the GraphQL schema while existing app versions keep asking for the fields they already use. | The venue can keep using `/v1/orders` while it plans for a migration. If a breaking change is introduced, versioning can be enabled. |

### Test Cases

Requirements: Seed 200 events, 40 venues, 12 organizers, 3 tiers per event, and 500 orders. 

Test Fixtures:

- Both games are at Amalie Arena (`venue123`), 401 Channelside Drive, Tampa, FL 33602. Set the city to Tampa and use the Tampa Bay Lightning (`organizer123`) as the organizer for both games.
- Use `event123` for Tampa Bay Lightning vs. Boston Bruins. Set `startsAt` to `2026-12-05T00:00:00Z` and the `description` to “The Lightning host the Bruins at Amalie Arena.”
- Use `event456` for Tampa Bay Lightning vs. Florida Panthers. Set `startsAt` to `2026-12-12T00:00:00Z`. These should be the only two Tampa events in the December browse results.
- For the Bruins game, Upper Level (`tier_upper`) starts with 2 seats left at 2,500 cents. Lower Level (`tier_lower`) has 6 left at 3,500 cents, and Club Level (`tier_club`) has 4 left at 5,000 cents. For the Panthers game, `tier_panthers_upper` is the cheapest tier at 4,000 cents.
- Three orders should match the sync cutoff. Set `order001` and `order002` to `2026-11-20T08:00:00Z`, and `order003` to `2026-11-20T08:01:00Z`.
- Reset the seed data before each test. If a test makes multiple requests, keep any changes from the earlier requests so the later ones can check them.


| ID | Scenario | Request | Expected status | Expected response |
| --- | --- | --- | --- | --- |
| R01 | Get one event | `GET /events/event123` | `200 OK` | Returns the Bruins game (`title`, `description`, `startsAt`), as well as `venueId: "venue123"` and `organizerId: "organizer123"`. |
| R02 | Get the event’s venue | `GET /venues/venue123` | `200 OK` | Returns the Amalie Arena venue details with `address: "401 Channelside Drive, Tampa, FL 33602"` and `city: "Tampa"`. |
| R03 | Get the event’s organizer | `GET /organizers/organizer123` | `200 OK` | Returns `organizer123` with `name: "Tampa Bay Lightning"`. |
| R04 | Browse Tampa events in December | `GET /events?city=Tampa&start_date=2026-12-01&end_date=2026-12-31&limit=20` | `200 OK` | Returns `event123` and `event456` in this order. |
| R05 | Get all tiers for the Bruins game | `GET /events/event123/tiers` | `200 OK` | Returns all three seeded tiers with their listed prices and seat counts. |
| R06 | Get the cheapest tier for each game in one call | `GET /tiers?eventIds=event123,event456&lowestPricePerEvent=true` | `200 OK` | Returns `tier_upper` for `event123` at 2,500 cents and `tier_panthers_upper` for `event456` at 4,000 cents. |
| R07 | Hold one Upper Level ticket and verify availability | `POST /events/event123/holds` with `{"tierId":"tier_upper","quantity":1}`. Then, call `GET /events/event123/tiers` in that same test. | `201 Created` for the hold, `200 OK` for the tier lookup | The hold gets a new id, reserves 1 ticket in tier_upper, and has an `expiresAt` 10 minutes after it was created. When you check `tier_upper again`, `quantityRemaining` is 1. |
| R08 | Pull orders across two pages | `GET /v1/orders?createdAtFrom=2026-11-20T08:00:00Z&limit=2`. Repeat with the same filter and limit, passing page 1’s `nextCursor` as `cursor`. | `200 OK` for both calls | The first request returns `order001` followed by `order002`, plus a `cursor` for the next page. Using that `cursor` returns only `order003` and `nextCursor: null`, which means the sync has completed. |
| R09 | Look up an event that doesn’t exist | `GET /events/event_missing` | `404 Not Found` | Returns `error.code: "EVENT_NOT_FOUND"` and `error.message: "The event cannot be found."` |
| R10 | Try to hold 5 tickets with only 2 available | `POST /events/event123/holds` with `{"tierId":"tier_upper","quantity":5}` | `409 Conflict` | Returns `error.code: "NUM_TICKETS_UNAVAILABLE"` and `error.quantityRemaining: 2`. A follow-up tier read should still show 2 seats as none are reserved when quantity requested exceeds availability. |
| R11 | Try to continue after the last page | Run the two requests in R08. After the final page returns `nextCursor: null`, send `GET /v1/orders?createdAtFrom=2026-11-20T08:00:00Z&limit=2&cursor=null`. | `400 Bad Request` on the continuation attempt | Returns `error.code: "INVALID_CURSOR"` where `cursor=null`. |
| R12 | Send an incomplete JSON body for a hold | `POST /events/event123/holds` with `Content-Type: application/json` and the incomplete body `{"tierId":"tier_upper","quantity":` | `400 Bad Request` | Returns `error.code: "INVALID_HOLD_REQUEST"`. No hold is created, and Upper Level should still has 2 tickets left. |
| G01 | Load the full event detail screen in one request | `query { event(id: "event123") { title description startsAt venue { name address city } organizer { name } tiers { id name priceCents quantityRemaining } } }` | `200 OK` | `data.event` returns the Bruins game details, venue, organizer, and all three tiers with their prices and seat counts. No GraphQL errors. |
| G02 | Load the mobile browse list with just the fields it needs | `query { browseEvents(city: "Tampa", start_date: "2026-12-01", end_date: "2026-12-31") { title startsAt lowestTierPriceCents } }` | `200 OK` | `data.browseEvents` returns the Bruins and Panthers games with their respective start times and lowest prices of 2,500 and 4,000 cents, respectively. Each row contains exactly `title`, `startsAt, and `lowestTierPriceCents`. No GraphQL errors.|

