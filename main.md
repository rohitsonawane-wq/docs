# Booking Service - Frontend API Reference

All communication happens via **message patterns** (RabbitMQ). Every response follows the uniform wrapper format.

---

## Table of Contents

- [Response Format](#response-format)
- [Error Format](#error-format)
- [Enums](#enums)
- [Booking Endpoints](#booking-endpoints)
  - [booking.create](#1-bookingcreate---create-booking)
  - [booking.draft](#2-bookingdraft---save-draft)
  - [booking.final](#3-bookingfinal---submit-final-booking)
  - [booking.list](#4-bookinglist---list-bookings)
  - [booking.preview](#5-bookingpreview---preview-booking)
  - [booking.findOne](#6-bookingfindone---find-one-booking)
  - [booking.listCompletedTrips](#7-bookinglistcompletedtrips---completed-trips)
  - [booking.update](#8-bookingupdate---update-booking)
  - [booking.remove](#9-bookingremove---remove-booking)
- [Itinerary Endpoints](#itinerary-endpoints)
  - [itinerary.create](#1-itinerarycreate---create-itinerary)
  - [itinerary.list](#2-itinerarylist---list-itineraries)
  - [itinerary.findOne](#3-itineraryfindone---find-one-itinerary)
  - [itinerary.update](#4-itineraryupdate---update-itinerary)
  - [itinerary.remove](#5-itineraryremove---remove-itinerary)

---

## Response Format

### Success Response

```json
{
  "success": true,
  "message": "Booking created successfully",
  "meta": null,
  "data": { ... }
}
```

### Paginated Success Response

```json
{
  "success": true,
  "message": "Bookings retrieved successfully",
  "meta": {
    "page": 1,
    "limit": 20,
    "total": 55,
    "current": 20
  },
  "data": [ ... ]
}
```

| Key | Type | Description |
|-----|------|-------------|
| `success` | `boolean` | `true` for success, `false` for error |
| `message` | `string` | Human-readable message |
| `meta` | `object \| null` | Pagination info for list endpoints, `null` otherwise |
| `meta.page` | `number` | Current page number |
| `meta.limit` | `number` | Items per page |
| `meta.total` | `number` | Total items in database matching the query |
| `meta.current` | `number` | Number of items returned in this response |
| `data` | `object \| array \| null` | Response payload |

---

## Error Format

### Validation Error

```json
{
  "success": false,
  "message": "firstName must be a string, email must be an email",
  "meta": null,
  "data": null
}
```

### Not Found Error

```json
{
  "success": false,
  "message": "Booking #999 not found",
  "meta": null,
  "data": null
}
```

### Finalization Error (missing fields)

```json
{
  "success": false,
  "message": "Cannot finalize booking. Missing required fields.",
  "meta": {
    "missingFields": ["firstName", "telephone", "itineraries"]
  },
  "data": null
}
```

---

## Enums

### ApplicantEnum

| Value | Description |
|-------|-------------|
| `"self"` | Booking for self |
| `"arranger"` | Booking arranged for someone else |

### GenderEnum

| Value |
|-------|
| `"male"` |
| `"female"` |

### TravelerTypeEnum

| Value |
|-------|
| `"internal"` |
| `"external"` |

### BookingTypesEnum

| Value | Description |
|-------|-------------|
| `"flight"` | Online flight booking |
| `"hotel"` | Online hotel booking |
| `"offline_flight"` | Offline flight booking |
| `"offline_hotel"` | Offline hotel booking |
| `"offline_jr"` | Offline JR (train) booking |
| `"offline_car"` | Offline car rental booking |
| `"offline_other"` | Offline other type booking |
| `"train"` | Online train booking |
| `"car"` | Online car booking |

### BookingStatus

| Value | Description |
|-------|-------------|
| `"draft"` | Initial state, editable |
| `"pending"` | Submitted for approval |
| `"confirmed"` | Approved/confirmed |
| `"cancelled"` | Cancelled |

### BookingApprovalStatus

| Value |
|-------|
| `"pending"` |
| `"approved"` |
| `"rejected"` |

---

## Booking Endpoints

---

### 1. `booking.create` - Create Booking

Creates a new booking with all required fields. Status is set to `draft`.

#### Request Payload

```json
{
  "applicant": "self",
  "firstName": "Taro",
  "lastName": "Yamada",
  "telephone": "090-1234-5678",
  "extensionNo": "101",
  "email": "taro.yamada@example.com",
  "travellerId": 123,
  "japaneseFirstName": "太郎",
  "japaneseLastName": "山田",
  "fullNameAsPerPassport": "TARO YAMADA",
  "gender": "male",
  "travelerType": "internal",
  "meetingNo": "MTG-2026-001",
  "itineraries": [
    {
      "type": "offline_flight",
      "departureDateTime": "2026-05-01T09:00:00.000Z",
      "departureCity": "Tokyo",
      "arrivalDateTime": "2026-05-01T12:00:00.000Z",
      "arrivalCity": "Osaka",
      "airline": "ANA",
      "flightNumber": "NH123",
      "cabinClass": "economy",
      "numberOfPassengers": 1,
      "remarks": "Window seat preferred"
    },
    {
      "type": "offline_hotel",
      "checkIn": "2026-05-01",
      "checkOut": "2026-05-03",
      "accommodationCity": "Osaka",
      "firstPreference": "Hilton Osaka",
      "secondPreference": "Marriott Osaka",
      "budgetMin": 10000,
      "budgetMax": 20000,
      "roomCondition": "Non-smoking",
      "amenities": "WiFi, Breakfast",
      "roomCount": 1,
      "roomType": "Single",
      "remarks": "Late check-in"
    }
  ]
}
```

#### Request Keys

| Key | Type | Required | Validation | Description |
|-----|------|----------|------------|-------------|
| `applicant` | `string` | Yes | Enum: `"self"`, `"arranger"` | Who is making the booking |
| `firstName` | `string` | Yes | Non-empty string | First name (English) |
| `lastName` | `string` | Yes | Non-empty string | Last name (English) |
| `telephone` | `string` | Yes | Non-empty string | Phone number |
| `extensionNo` | `string` | No | String | Phone extension number |
| `email` | `string` | Yes | Valid email format | Email address |
| `travellerId` | `number` | Yes | Number | ID of the traveller |
| `japaneseFirstName` | `string` | Yes | Non-empty string | First name in Japanese |
| `japaneseLastName` | `string` | Yes | Non-empty string | Last name in Japanese |
| `fullNameAsPerPassport` | `string` | Yes | Non-empty string | Full name as printed on passport |
| `gender` | `string` | Yes | Enum: `"male"`, `"female"` | Gender |
| `travelerType` | `string` | Yes | Enum: `"internal"`, `"external"` | Traveler type |
| `meetingNo` | `string` | No | String | Meeting reference number |
| `itineraries` | `array` | No | Array of itinerary objects | List of itineraries (see [Itinerary Object](#itinerary-object-createitinerarydto)) |

#### Response

```json
{
  "success": true,
  "message": "Booking created successfully",
  "meta": null,
  "data": {
    "id": 1,
    "tenantId": 1,
    "travellerId": 123,
    "createdBy": 2,
    "applicant": "self",
    "firstName": "Taro",
    "lastName": "Yamada",
    "telephone": "090-1234-5678",
    "extensionNo": "101",
    "email": "taro.yamada@example.com",
    "japaneseFirstName": "太郎",
    "japaneseLastName": "山田",
    "fullNameAsPerPassport": "TARO YAMADA",
    "gender": "male",
    "travelerType": "internal",
    "meetingNo": "MTG-2026-001",
    "status": "draft",
    "approvalStatus": null,
    "approverId": null,
    "approvedAt": null,
    "createdAt": "2026-04-14T10:00:00.000Z",
    "updatedAt": "2026-04-14T10:00:00.000Z",
    "itineraries": [
      {
        "itineraryId": 1,
        "bookingId": 1,
        "tenantId": 1,
        "type": "offline_flight",
        "sortOrder": 0,
        "createdAt": "2026-04-14T10:00:00.000Z",
        "departureDateTime": "2026-05-01T09:00:00.000Z",
        "departureCity": "Tokyo",
        "arrivalDateTime": "2026-05-01T12:00:00.000Z",
        "arrivalCity": "Osaka",
        "airline": "ANA",
        "flightNumber": "NH123",
        "cabinClass": "economy",
        "numberOfPassengers": 1,
        "remarks": "Window seat preferred",
        "requestedAt": "2026-04-14T10:00:00.000Z",
        "processedAt": null
      },
      {
        "itineraryId": 2,
        "bookingId": 1,
        "tenantId": 1,
        "type": "offline_hotel",
        "sortOrder": 1,
        "createdAt": "2026-04-14T10:00:00.000Z",
        "checkIn": "2026-05-01",
        "checkOut": "2026-05-03",
        "accommodationCity": "Osaka",
        "firstPreference": "Hilton Osaka",
        "secondPreference": "Marriott Osaka",
        "budgetMin": 10000,
        "budgetMax": 20000,
        "roomCondition": "Non-smoking",
        "amenities": "WiFi, Breakfast",
        "roomCount": 1,
        "roomType": "Single",
        "remarks": "Late check-in",
        "requestedAt": "2026-04-14T10:00:00.000Z",
        "processedAt": null
      }
    ]
  }
}
```

#### Response Keys (`data`)

| Key | Type | Nullable | Description |
|-----|------|----------|-------------|
| `id` | `number` | No | Booking ID (auto-generated) |
| `tenantId` | `number` | No | Tenant ID (set by server) |
| `travellerId` | `number` | Yes | Traveller ID |
| `createdBy` | `number` | No | User ID who created the booking (set by server) |
| `applicant` | `string` | Yes | `"self"` or `"arranger"` |
| `firstName` | `string` | Yes | First name |
| `lastName` | `string` | Yes | Last name |
| `telephone` | `string` | Yes | Phone number |
| `extensionNo` | `string` | Yes | Extension number |
| `email` | `string` | Yes | Email address |
| `japaneseFirstName` | `string` | Yes | Japanese first name |
| `japaneseLastName` | `string` | Yes | Japanese last name |
| `fullNameAsPerPassport` | `string` | Yes | Passport full name |
| `gender` | `string` | Yes | `"male"` or `"female"` |
| `travelerType` | `string` | Yes | `"internal"` or `"external"` |
| `meetingNo` | `string` | Yes | Meeting number |
| `status` | `string` | No | `"draft"`, `"pending"`, `"confirmed"`, `"cancelled"` |
| `approvalStatus` | `string` | Yes | `"pending"`, `"approved"`, `"rejected"` |
| `approverId` | `string` | Yes | UUID of approver |
| `approvedAt` | `string (ISO 8601)` | Yes | Approval timestamp |
| `createdAt` | `string (ISO 8601)` | No | Creation timestamp |
| `updatedAt` | `string (ISO 8601)` | No | Last update timestamp |
| `itineraries` | `array` | No | Array of flattened itinerary objects (see [Itinerary Response](#itinerary-response-object)) |

---

### 2. `booking.draft` - Save Draft

Save a partial booking as draft. All fields are optional. Pass `id: null` or omit `id` to create a new draft. Pass an existing `id` to update an existing draft.

#### Request Payload (new draft)

```json
{
  "id": null,
  "applicant": "self",
  "firstName": "Taro",
  "email": "taro@example.com",
  "travellerId": 123
}
```

#### Request Payload (update existing draft)

```json
{
  "id": 1,
  "lastName": "Yamada",
  "telephone": "090-1234-5678",
  "itineraries": [
    {
      "type": "offline_flight",
      "departureCity": "Tokyo",
      "arrivalCity": "Osaka"
    }
  ]
}
```

#### Request Keys

| Key | Type | Required | Validation | Description |
|-----|------|----------|------------|-------------|
| `id` | `number \| null` | No | Integer or null | `null`/omitted = create new draft, number = update existing draft |
| `applicant` | `string` | No | Enum: `"self"`, `"arranger"` | Who is making the booking |
| `firstName` | `string` | No | String | First name |
| `lastName` | `string` | No | String | Last name |
| `telephone` | `string` | No | String | Phone number |
| `extensionNo` | `string` | No | String | Phone extension |
| `email` | `string` | No | Valid email if provided | Email address |
| `travellerId` | `number` | No | Number | Traveller ID |
| `japaneseFirstName` | `string` | No | String | Japanese first name |
| `japaneseLastName` | `string` | No | String | Japanese last name |
| `fullNameAsPerPassport` | `string` | No | String | Passport name |
| `gender` | `string` | No | Enum: `"male"`, `"female"` | Gender |
| `travelerType` | `string` | No | Enum: `"internal"`, `"external"` | Traveler type |
| `meetingNo` | `string` | No | String | Meeting number |
| `itineraries` | `array` | No | Array of itinerary objects | Replaces ALL existing itineraries when provided |

> **Important:** When `itineraries` is provided, ALL existing itineraries for the booking are deleted and replaced with the new array. Omit `itineraries` to keep existing ones unchanged.

#### Response

```json
{
  "success": true,
  "message": "Draft saved successfully",
  "meta": null,
  "data": {
    "id": 1,
    "tenantId": 1,
    "travellerId": 123,
    "createdBy": 2,
    "applicant": "self",
    "firstName": "Taro",
    "lastName": "Yamada",
    "telephone": "090-1234-5678",
    "extensionNo": null,
    "email": "taro@example.com",
    "japaneseFirstName": null,
    "japaneseLastName": null,
    "fullNameAsPerPassport": null,
    "gender": null,
    "travelerType": null,
    "meetingNo": null,
    "status": "draft",
    "approvalStatus": null,
    "approverId": null,
    "approvedAt": null,
    "createdAt": "2026-04-14T10:00:00.000Z",
    "updatedAt": "2026-04-14T10:05:00.000Z",
    "itineraries": [
      {
        "itineraryId": 3,
        "bookingId": 1,
        "tenantId": 1,
        "type": "offline_flight",
        "sortOrder": 0,
        "createdAt": "2026-04-14T10:05:00.000Z",
        "departureDateTime": null,
        "departureCity": "Tokyo",
        "arrivalDateTime": null,
        "arrivalCity": "Osaka",
        "airline": null,
        "flightNumber": null,
        "cabinClass": null,
        "numberOfPassengers": null,
        "remarks": null,
        "requestedAt": "2026-04-14T10:05:00.000Z",
        "processedAt": null
      }
    ]
  }
}
```

#### Errors

| Condition | HTTP Status | Message |
|-----------|-------------|---------|
| Booking ID not found | 404 | `Booking #${id} not found` |
| Booking is not a draft | 400 | `Booking #${id} is not a draft and cannot be updated via draft save` |

---

### 3. `booking.final` - Submit Final Booking

Finalize a draft booking. Changes status from `draft` to `pending`.

#### Request Payload

```json
{
  "id": 1
}
```

#### Request Keys

| Key | Type | Required | Validation | Description |
|-----|------|----------|------------|-------------|
| `id` | `number` | Yes | Integer | Booking ID to finalize |

#### Validation Rules

The following fields must be filled (non-null, non-empty) on the booking before finalization:

- `firstName`
- `lastName`
- `telephone`
- `email`
- `japaneseFirstName`
- `japaneseLastName`
- `fullNameAsPerPassport`
- `gender`
- `travelerType`
- `applicant`
- At least **one itinerary** must exist

#### Response (Success)

```json
{
  "success": true,
  "message": "Booking submitted successfully",
  "meta": null,
  "data": {
    "id": 1,
    "tenantId": 1,
    "travellerId": 123,
    "createdBy": 2,
    "applicant": "self",
    "firstName": "Taro",
    "lastName": "Yamada",
    "telephone": "090-1234-5678",
    "extensionNo": "101",
    "email": "taro.yamada@example.com",
    "japaneseFirstName": "太郎",
    "japaneseLastName": "山田",
    "fullNameAsPerPassport": "TARO YAMADA",
    "gender": "male",
    "travelerType": "internal",
    "meetingNo": "MTG-2026-001",
    "status": "pending",
    "approvalStatus": null,
    "approverId": null,
    "approvedAt": null,
    "createdAt": "2026-04-14T10:00:00.000Z",
    "updatedAt": "2026-04-14T10:10:00.000Z",
    "itineraries": [ ... ]
  }
}
```

#### Errors

| Condition | HTTP Status | Message | Meta |
|-----------|-------------|---------|------|
| Booking not found | 404 | `Booking #${id} not found` | `null` |
| Not a draft | 400 | `Booking #${id} cannot be finalized because it is not a draft.` | `null` |
| Missing required fields | 400 | `Cannot finalize booking. Missing required fields.` | `{ "missingFields": ["firstName", "telephone", "itineraries"] }` |

---

### 4. `booking.list` - List Bookings

Paginated list of bookings with optional filters. Ordered by `createdAt DESC`.

#### Request Payload

```json
{
  "type": "self",
  "status": "draft",
  "approvalStatus": "pending",
  "travellerId": 123,
  "createdBy": 2,
  "page": 1,
  "limit": 20
}
```

#### Request Keys

| Key | Type | Required | Default | Validation | Description |
|-----|------|----------|---------|------------|-------------|
| `type` | `string` | No | - | Enum: `"self"`, `"arranger"` | Filter by applicant type |
| `status` | `string` | No | - | Enum: `"draft"`, `"pending"`, `"confirmed"`, `"cancelled"` | Filter by booking status |
| `approvalStatus` | `string` | No | - | Enum: `"pending"`, `"approved"`, `"rejected"` | Filter by approval status |
| `travellerId` | `number` | No | - | Integer | Filter by traveller ID |
| `createdBy` | `number` | No | - | Integer | Filter by creator user ID |
| `page` | `number` | No | `1` | Integer, min: 1 | Page number |
| `limit` | `number` | No | `20` | Integer, min: 1, max: 100 | Items per page |

#### Response

```json
{
  "success": true,
  "message": "Bookings retrieved successfully",
  "meta": {
    "page": 1,
    "limit": 20,
    "total": 55,
    "current": 20
  },
  "data": [
    {
      "id": 1,
      "tenantId": 1,
      "travellerId": 123,
      "createdBy": 2,
      "applicant": "self",
      "firstName": "Taro",
      "lastName": "Yamada",
      "telephone": "090-1234-5678",
      "extensionNo": "101",
      "email": "taro.yamada@example.com",
      "japaneseFirstName": "太郎",
      "japaneseLastName": "山田",
      "fullNameAsPerPassport": "TARO YAMADA",
      "gender": "male",
      "travelerType": "internal",
      "meetingNo": "MTG-2026-001",
      "status": "draft",
      "approvalStatus": null,
      "approverId": null,
      "approvedAt": null,
      "createdAt": "2026-04-14T10:00:00.000Z",
      "updatedAt": "2026-04-14T10:00:00.000Z",
      "itineraries": [
        {
          "itineraryId": 1,
          "bookingId": 1,
          "tenantId": 1,
          "type": "offline_flight",
          "sortOrder": 0,
          "createdAt": "2026-04-14T10:00:00.000Z",
          "departureDateTime": "2026-05-01T09:00:00.000Z",
          "departureCity": "Tokyo",
          "arrivalDateTime": "2026-05-01T12:00:00.000Z",
          "arrivalCity": "Osaka",
          "airline": "ANA",
          "flightNumber": "NH123",
          "cabinClass": "economy",
          "numberOfPassengers": 1,
          "remarks": null,
          "requestedAt": "2026-04-14T10:00:00.000Z",
          "processedAt": null
        }
      ]
    }
  ]
}
```

---

### 5. `booking.preview` - Preview Booking

Fetch a single booking with all itineraries for preview.

#### Request Payload

```json
{
  "id": 1
}
```

#### Request Keys

| Key | Type | Required | Description |
|-----|------|----------|-------------|
| `id` | `number` | Yes | Booking ID |

#### Response

```json
{
  "success": true,
  "message": "Booking retrieved successfully",
  "meta": null,
  "data": {
    "id": 1,
    "tenantId": 1,
    "travellerId": 123,
    "createdBy": 2,
    "applicant": "self",
    "firstName": "Taro",
    "lastName": "Yamada",
    "telephone": "090-1234-5678",
    "extensionNo": "101",
    "email": "taro.yamada@example.com",
    "japaneseFirstName": "太郎",
    "japaneseLastName": "山田",
    "fullNameAsPerPassport": "TARO YAMADA",
    "gender": "male",
    "travelerType": "internal",
    "meetingNo": "MTG-2026-001",
    "status": "draft",
    "approvalStatus": null,
    "approverId": null,
    "approvedAt": null,
    "createdAt": "2026-04-14T10:00:00.000Z",
    "updatedAt": "2026-04-14T10:00:00.000Z",
    "itineraries": [ ... ]
  }
}
```

---

### 6. `booking.findOne` - Find One Booking

Identical to preview. Fetch a single booking by ID with all itineraries.

#### Request Payload

```json
{
  "id": 1
}
```

#### Request Keys

| Key | Type | Required | Description |
|-----|------|----------|-------------|
| `id` | `number` | Yes | Booking ID |

#### Response

Same structure as `booking.preview`.

---

### 7. `booking.listCompletedTrips` - Completed Trips

Paginated list of confirmed bookings for a specific traveller. Only returns bookings with `status: "confirmed"`. Ordered by `createdAt DESC`.

#### Request Payload

```json
{
  "travellerId": 123,
  "page": 1,
  "limit": 20
}
```

#### Request Keys

| Key | Type | Required | Default | Validation | Description |
|-----|------|----------|---------|------------|-------------|
| `travellerId` | `number` | Yes | - | Integer | Traveller ID |
| `page` | `number` | No | `1` | Integer, min: 1 | Page number |
| `limit` | `number` | No | `20` | Integer, min: 1, max: 100 | Items per page |

#### Response

```json
{
  "success": true,
  "message": "Completed trips retrieved successfully",
  "meta": {
    "page": 1,
    "limit": 20,
    "total": 5,
    "current": 5
  },
  "data": [
    {
      "id": 10,
      "tenantId": 1,
      "travellerId": 123,
      "createdBy": 2,
      "applicant": "self",
      "firstName": "Taro",
      "lastName": "Yamada",
      "telephone": "090-1234-5678",
      "extensionNo": null,
      "email": "taro@example.com",
      "japaneseFirstName": "太郎",
      "japaneseLastName": "山田",
      "fullNameAsPerPassport": "TARO YAMADA",
      "gender": "male",
      "travelerType": "internal",
      "meetingNo": null,
      "status": "confirmed",
      "approvalStatus": "approved",
      "approverId": "a1b2c3d4-e5f6-7890-abcd-ef1234567890",
      "approvedAt": "2026-04-10T15:00:00.000Z",
      "createdAt": "2026-04-01T10:00:00.000Z",
      "updatedAt": "2026-04-10T15:00:00.000Z",
      "itineraries": [ ... ]
    }
  ]
}
```

---

### 8. `booking.update` - Update Booking

> **Note:** Not fully implemented. Returns a placeholder string.

#### Request Payload

```json
{
  "id": 1,
  "firstName": "Updated Name",
  "email": "new@example.com"
}
```

#### Request Keys

| Key | Type | Required | Description |
|-----|------|----------|-------------|
| `id` | `number` | Yes | Booking ID |
| _(all fields from create)_ | | No | Any booking field to update |

---

### 9. `booking.remove` - Remove Booking

> **Note:** Not fully implemented. Returns a placeholder string.

#### Request Payload

```json
{
  "id": 1
}
```

#### Request Keys

| Key | Type | Required | Description |
|-----|------|----------|-------------|
| `id` | `number` | Yes | Booking ID |

---

## Itinerary Endpoints

---

### Itinerary Object (`CreateItineraryDto`)

This is the shape used when sending itineraries in create/draft requests. Send only the fields relevant to the itinerary `type`.

#### Common Fields

| Key | Type | Required | Validation | Description |
|-----|------|----------|------------|-------------|
| `type` | `string` | Yes | BookingTypesEnum | Type of itinerary |
| `remarks` | `string` | No | String | Additional remarks (max 1000 chars in DB) |

#### Flight Fields (use when `type` = `"flight"` or `"offline_flight"`)

| Key | Type | Required | Validation | Description |
|-----|------|----------|------------|-------------|
| `departureDateTime` | `string` | No | ISO 8601 datetime | Departure date and time |
| `departureCity` | `string` | No | String | Departure city |
| `arrivalDateTime` | `string` | No | ISO 8601 datetime | Arrival date and time |
| `arrivalCity` | `string` | No | String | Arrival city |
| `airline` | `string` | No | String | Airline name |
| `flightNumber` | `string` | No | String | Flight number |
| `cabinClass` | `string` | No | String | Cabin class (e.g., economy, business) |
| `numberOfPassengers` | `number` | No | Integer, min: 1 | Number of passengers |

**Example:**
```json
{
  "type": "offline_flight",
  "departureDateTime": "2026-05-01T09:00:00.000Z",
  "departureCity": "Tokyo",
  "arrivalDateTime": "2026-05-01T12:00:00.000Z",
  "arrivalCity": "Osaka",
  "airline": "ANA",
  "flightNumber": "NH123",
  "cabinClass": "economy",
  "numberOfPassengers": 1,
  "remarks": "Window seat preferred"
}
```

#### Hotel Fields (use when `type` = `"offline_hotel"`)

| Key | Type | Required | Validation | Description |
|-----|------|----------|------------|-------------|
| `checkIn` | `string` | No | ISO 8601 date (`YYYY-MM-DD`) | Check-in date |
| `checkOut` | `string` | No | ISO 8601 date (`YYYY-MM-DD`) | Check-out date |
| `accommodationCity` | `string` | No | String | City of accommodation |
| `firstPreference` | `string` | No | String | First hotel preference |
| `secondPreference` | `string` | No | String | Second hotel preference |
| `budgetMin` | `number` | No | Integer, min: 0 | Minimum budget |
| `budgetMax` | `number` | No | Integer, min: 0 | Maximum budget |
| `roomCondition` | `string` | No | String | Room condition (e.g., Non-smoking) |
| `amenities` | `string` | No | String | Desired amenities |
| `roomCount` | `number` | No | Integer, min: 1 | Number of rooms |
| `roomType` | `string` | No | String | Room type (e.g., Single, Double) |

**Example:**
```json
{
  "type": "offline_hotel",
  "checkIn": "2026-05-01",
  "checkOut": "2026-05-03",
  "accommodationCity": "Osaka",
  "firstPreference": "Hilton Osaka",
  "secondPreference": "Marriott Osaka",
  "budgetMin": 10000,
  "budgetMax": 20000,
  "roomCondition": "Non-smoking",
  "amenities": "WiFi, Breakfast",
  "roomCount": 1,
  "roomType": "Single",
  "remarks": "Late check-in around 22:00"
}
```

#### JR / Train Fields (use when `type` = `"offline_jr"`)

| Key | Type | Required | Validation | Description |
|-----|------|----------|------------|-------------|
| `transportationType` | `string` | No | String | Type of transportation (e.g., Shinkansen) |
| `ticketType` | `string` | No | String | Ticket type (e.g., Round-trip) |
| `departureDate` | `string` | No | ISO 8601 date (`YYYY-MM-DD`) | Departure date |
| `origin` | `string` | No | String | Origin station |
| `destination` | `string` | No | String | Destination station |
| `departureTime` | `string` | No | String | Departure time |
| `arrivalTime` | `string` | No | String | Arrival time |
| `trainName` | `string` | No | String | Train name (e.g., Nozomi) |
| `trainNo` | `string` | No | String | Train number |
| `seats` | `string` | No | String | Seat information |
| `ticketOrigin` | `string` | No | String | Ticket origin station |
| `ticketDestination` | `string` | No | String | Ticket destination station |
| `seatPreference` | `string` | No | String | Seat preference (e.g., Window) |
| `seatType` | `string` | No | String | Seat type (e.g., Reserved, Non-reserved) |

**Example:**
```json
{
  "type": "offline_jr",
  "transportationType": "Shinkansen",
  "ticketType": "Round-trip",
  "departureDate": "2026-05-01",
  "origin": "Tokyo Station",
  "destination": "Shin-Osaka Station",
  "departureTime": "08:00",
  "arrivalTime": "10:30",
  "trainName": "Nozomi",
  "trainNo": "1",
  "seats": "Car 7, Seat 3A",
  "ticketOrigin": "Tokyo",
  "ticketDestination": "Shin-Osaka",
  "seatPreference": "Window",
  "seatType": "Reserved",
  "remarks": "Green car if available"
}
```

#### Car Rental Fields (use when `type` = `"offline_car"`)

| Key | Type | Required | Validation | Description |
|-----|------|----------|------------|-------------|
| `rentalDateTime` | `string` | No | ISO 8601 datetime | Rental pickup date and time |
| `rentalCity` | `string` | No | String | Pickup city |
| `returnDateTime` | `string` | No | ISO 8601 datetime | Return date and time |
| `returnCity` | `string` | No | String | Return city |
| `numberOfCars` | `number` | No | Integer, min: 1 | Number of cars |
| `rentalCarCompany` | `string` | No | String | Rental car company |
| `carSize` | `string` | No | String | Car size (e.g., Compact, Standard) |
| `driver` | `string` | No | String | Driver name |

**Example:**
```json
{
  "type": "offline_car",
  "rentalDateTime": "2026-05-01T10:00:00.000Z",
  "rentalCity": "Osaka",
  "returnDateTime": "2026-05-03T18:00:00.000Z",
  "returnCity": "Osaka",
  "numberOfCars": 1,
  "rentalCarCompany": "Toyota Rent a Car",
  "carSize": "Compact",
  "driver": "Taro Yamada",
  "remarks": "GPS navigation required"
}
```

---

### Itinerary Response Object

All itinerary responses are **flattened** - the type-specific fields from the sub-entity are merged into the root level. The key `itineraryId` replaces `id`.

#### Common Response Keys (present in all types)

| Key | Type | Description |
|-----|------|-------------|
| `itineraryId` | `number` | Itinerary ID |
| `bookingId` | `number` | Parent booking ID |
| `tenantId` | `number` | Tenant ID |
| `type` | `string` | BookingTypesEnum value |
| `sortOrder` | `number` | Sort position (0-based) |
| `createdAt` | `string (ISO 8601)` | Creation timestamp |

#### Flight Response Keys (when `type` = `"flight"` or `"offline_flight"`)

| Key | Type | Nullable | Description |
|-----|------|----------|-------------|
| `tenantId` | `number` | No | Tenant ID (from sub-entity) |
| `departureDateTime` | `string (ISO 8601)` | No | Departure timestamp |
| `departureCity` | `string` | No | Departure city |
| `arrivalDateTime` | `string (ISO 8601)` | No | Arrival timestamp |
| `arrivalCity` | `string` | No | Arrival city |
| `airline` | `string` | Yes | Airline name |
| `flightNumber` | `string` | Yes | Flight number |
| `cabinClass` | `string` | No | Cabin class |
| `numberOfPassengers` | `number` | No | Passenger count |
| `remarks` | `string` | Yes | Remarks (max 1000 chars) |
| `requestedAt` | `string (ISO 8601)` | No | When the itinerary was requested |
| `processedAt` | `string (ISO 8601)` | Yes | When the itinerary was processed (null if not yet processed) |

#### Hotel Response Keys (when `type` = `"offline_hotel"`)

| Key | Type | Nullable | Description |
|-----|------|----------|-------------|
| `tenantId` | `number` | No | Tenant ID |
| `checkIn` | `string (YYYY-MM-DD)` | No | Check-in date |
| `checkOut` | `string (YYYY-MM-DD)` | No | Check-out date |
| `accommodationCity` | `string` | No | City |
| `firstPreference` | `string` | Yes | First hotel preference |
| `secondPreference` | `string` | Yes | Second hotel preference |
| `budgetMin` | `number` | No | Minimum budget |
| `budgetMax` | `number` | No | Maximum budget |
| `roomCondition` | `string` | No | Room condition |
| `amenities` | `string` | No | Amenities |
| `roomCount` | `number` | No | Number of rooms |
| `roomType` | `string` | No | Room type |
| `remarks` | `string` | Yes | Remarks |
| `requestedAt` | `string (ISO 8601)` | No | Request timestamp |
| `processedAt` | `string (ISO 8601)` | Yes | Process timestamp |

#### JR Response Keys (when `type` = `"offline_jr"`)

| Key | Type | Nullable | Description |
|-----|------|----------|-------------|
| `tenantId` | `number` | No | Tenant ID |
| `transportationType` | `string` | Yes | Transportation type |
| `ticketType` | `string` | Yes | Ticket type |
| `departureDate` | `string (YYYY-MM-DD)` | Yes | Departure date |
| `origin` | `string` | Yes | Origin station |
| `destination` | `string` | Yes | Destination station |
| `departureTime` | `string` | Yes | Departure time |
| `arrivalTime` | `string` | Yes | Arrival time |
| `trainName` | `string` | Yes | Train name |
| `trainNo` | `string` | Yes | Train number |
| `seats` | `string` | Yes | Seat info |
| `ticketOrigin` | `string` | Yes | Ticket origin |
| `ticketDestination` | `string` | Yes | Ticket destination |
| `seatPreference` | `string` | Yes | Seat preference |
| `seatType` | `string` | Yes | Seat type |
| `remarks` | `string` | Yes | Remarks |
| `requestedAt` | `string (ISO 8601)` | No | Request timestamp |
| `processedAt` | `string (ISO 8601)` | Yes | Process timestamp |

#### Car Rental Response Keys (when `type` = `"offline_car"`)

| Key | Type | Nullable | Description |
|-----|------|----------|-------------|
| `tenantId` | `number` | No | Tenant ID |
| `rentalDateTime` | `string (ISO 8601)` | No | Rental pickup datetime |
| `rentalCity` | `string` | No | Pickup city |
| `returnDateTime` | `string (ISO 8601)` | No | Return datetime |
| `returnCity` | `string` | No | Return city |
| `numberOfCars` | `number` | No | Number of cars |
| `rentalCarCompany` | `string` | Yes | Rental company |
| `carSize` | `string` | No | Car size |
| `driver` | `string` | No | Driver name |
| `remarks` | `string` | Yes | Remarks |
| `requestedAt` | `string (ISO 8601)` | No | Request timestamp |
| `processedAt` | `string (ISO 8601)` | Yes | Process timestamp |

---

### 1. `itinerary.create` - Create Itinerary

Add a new itinerary to an existing booking.

#### Request Payload

```json
{
  "bookingId": 1,
  "type": "offline_flight",
  "departureDateTime": "2026-05-01T09:00:00.000Z",
  "departureCity": "Tokyo",
  "arrivalDateTime": "2026-05-01T12:00:00.000Z",
  "arrivalCity": "Osaka",
  "airline": "ANA",
  "flightNumber": "NH123",
  "cabinClass": "economy",
  "numberOfPassengers": 1,
  "remarks": "Window seat preferred"
}
```

#### Request Keys

| Key | Type | Required | Description |
|-----|------|----------|-------------|
| `bookingId` | `number` | Yes | Parent booking ID |
| _(all fields from itinerary object)_ | | | See [Itinerary Object](#itinerary-object-createitinerarydto) |

#### Response

```json
{
  "success": true,
  "message": "Success",
  "meta": null,
  "data": {
    "itineraryId": 5,
    "bookingId": 1,
    "tenantId": 1,
    "type": "offline_flight",
    "sortOrder": 0,
    "createdAt": "2026-04-14T10:00:00.000Z",
    "departureDateTime": "2026-05-01T09:00:00.000Z",
    "departureCity": "Tokyo",
    "arrivalDateTime": "2026-05-01T12:00:00.000Z",
    "arrivalCity": "Osaka",
    "airline": "ANA",
    "flightNumber": "NH123",
    "cabinClass": "economy",
    "numberOfPassengers": 1,
    "remarks": "Window seat preferred",
    "requestedAt": "2026-04-14T10:00:00.000Z",
    "processedAt": null
  }
}
```

---

### 2. `itinerary.list` - List Itineraries

Paginated list of itineraries for a booking.

#### Request Payload

```json
{
  "bookingId": 1,
  "type": "offline_flight",
  "page": 1,
  "limit": 20
}
```

#### Request Keys

| Key | Type | Required | Default | Validation | Description |
|-----|------|----------|---------|------------|-------------|
| `bookingId` | `number` | Yes | - | Integer | Parent booking ID |
| `type` | `string` | No | - | BookingTypesEnum | Filter by itinerary type |
| `page` | `number` | No | `1` | Integer, min: 1 | Page number |
| `limit` | `number` | No | `20` | Integer, min: 1, max: 100 | Items per page |

#### Response

```json
{
  "success": true,
  "message": "Success",
  "meta": {
    "page": 1,
    "limit": 20,
    "total": 3,
    "current": 3
  },
  "data": [
    {
      "itineraryId": 1,
      "bookingId": 1,
      "tenantId": 1,
      "type": "offline_flight",
      "sortOrder": 0,
      "createdAt": "2026-04-14T10:00:00.000Z",
      "departureDateTime": "2026-05-01T09:00:00.000Z",
      "departureCity": "Tokyo",
      "arrivalDateTime": "2026-05-01T12:00:00.000Z",
      "arrivalCity": "Osaka",
      "airline": "ANA",
      "flightNumber": "NH123",
      "cabinClass": "economy",
      "numberOfPassengers": 1,
      "remarks": null,
      "requestedAt": "2026-04-14T10:00:00.000Z",
      "processedAt": null
    },
    {
      "itineraryId": 2,
      "bookingId": 1,
      "tenantId": 1,
      "type": "offline_hotel",
      "sortOrder": 1,
      "createdAt": "2026-04-14T10:00:00.000Z",
      "checkIn": "2026-05-01",
      "checkOut": "2026-05-03",
      "accommodationCity": "Osaka",
      "firstPreference": "Hilton Osaka",
      "secondPreference": null,
      "budgetMin": 10000,
      "budgetMax": 20000,
      "roomCondition": "Non-smoking",
      "amenities": "WiFi",
      "roomCount": 1,
      "roomType": "Single",
      "remarks": null,
      "requestedAt": "2026-04-14T10:00:00.000Z",
      "processedAt": null
    }
  ]
}
```

---

### 3. `itinerary.findOne` - Find One Itinerary

#### Request Payload

```json
{
  "bookingId": 1,
  "id": 5
}
```

#### Request Keys

| Key | Type | Required | Description |
|-----|------|----------|-------------|
| `bookingId` | `number` | Yes | Parent booking ID |
| `id` | `number` | Yes | Itinerary ID |

#### Response

```json
{
  "success": true,
  "message": "Success",
  "meta": null,
  "data": {
    "itineraryId": 5,
    "bookingId": 1,
    "tenantId": 1,
    "type": "offline_jr",
    "sortOrder": 2,
    "createdAt": "2026-04-14T10:00:00.000Z",
    "transportationType": "Shinkansen",
    "ticketType": "Round-trip",
    "departureDate": "2026-05-01",
    "origin": "Tokyo Station",
    "destination": "Shin-Osaka Station",
    "departureTime": "08:00",
    "arrivalTime": "10:30",
    "trainName": "Nozomi",
    "trainNo": "1",
    "seats": "Car 7, Seat 3A",
    "ticketOrigin": "Tokyo",
    "ticketDestination": "Shin-Osaka",
    "seatPreference": "Window",
    "seatType": "Reserved",
    "remarks": null,
    "requestedAt": "2026-04-14T10:00:00.000Z",
    "processedAt": null
  }
}
```

---

### 4. `itinerary.update` - Update Itinerary

Update type-specific fields on an existing itinerary. Only send the fields you want to change.

#### Request Payload

```json
{
  "bookingId": 1,
  "id": 5,
  "departureCity": "Nagoya",
  "arrivalCity": "Fukuoka",
  "remarks": "Updated remarks"
}
```

#### Request Keys

| Key | Type | Required | Description |
|-----|------|----------|-------------|
| `bookingId` | `number` | Yes | Parent booking ID |
| `id` | `number` | Yes | Itinerary ID |
| _(all type-specific fields)_ | | No | Any field from the itinerary type (see tables above) |

#### Response

Returns the updated flattened itinerary object.

---

### 5. `itinerary.remove` - Remove Itinerary

#### Request Payload

```json
{
  "bookingId": 1,
  "id": 5
}
```

#### Request Keys

| Key | Type | Required | Description |
|-----|------|----------|-------------|
| `bookingId` | `number` | Yes | Parent booking ID |
| `id` | `number` | Yes | Itinerary ID |

#### Response

```json
{
  "success": true,
  "message": "Success",
  "meta": null,
  "data": null
}
```

---

## Key Notes for Frontend

1. **Uniform response wrapper**: Every response has `success`, `message`, `meta`, `data`. Always check `success` first.
2. **Itineraries are flattened**: Type-specific fields appear at the root level of each itinerary object, not nested. The original `id` is renamed to `itineraryId`.
3. **Draft flow**: Create draft (`booking.draft`) -> Add/modify itineraries -> Submit final (`booking.final`). Drafts accept partial data.
4. **Draft itinerary replacement**: When sending `itineraries` array in `booking.draft`, ALL existing itineraries are deleted and replaced.
5. **Finalization validation**: `booking.final` checks all required fields + at least one itinerary. On failure, `meta.missingFields` tells you exactly what's missing.
6. **Status transitions**: `draft` -> `pending` (via `booking.final`). `confirmed` and `cancelled` are set by other services/processes.
7. **Pagination**: Use `page` and `limit`. Check `meta.total` for total count, `meta.current` for items in current page.
8. **Cascade delete**: Deleting a booking automatically removes all its itineraries and sub-records.
9. **Server-set fields**: `id`, `tenantId`, `createdBy`, `status`, `createdAt`, `updatedAt`, `requestedAt`, `processedAt`, `sortOrder` are set by the server. Do not send these in requests.
10. **Null values**: Unfilled optional fields return `null` in responses, not `undefined`.
