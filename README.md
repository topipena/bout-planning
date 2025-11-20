# bout-planning

## APIs

### MVP API Spec (Passenger / Client ↔ Company)

**Base URL**: `https://api.example.com/v1`  
**Auth**: Opaque bearer token (e.g., `Authorization: Bearer <token>`)

---

## 1. Areas & Service Points

### 1.1 List Areas

- **GET** `/areas`
  - **Description**: Returns areas the client can use (from `AREAS` / `COUNTRIES`).

#### Response 200

```json
[
  {
    "id": "area-uuid",
    "name": "Helsinki Archipelago",
    "country_code": "FI"
  }
]
```

### 1.2 List Service Points in Area

- **GET** `/areas/{area_id}/service-points`
  - **Path Params**: `area_id` (UUID of the area)

#### Response 200

```json
[
  {
    "id": "sp-uuid-1",
    "name": "Market Square",
    "area_id": "area-uuid"
  },
  {
    "id": "sp-uuid-2",
    "name": "Suomenlinna",
    "area_id": "area-uuid"
  }
]
```

---

## 2. Availability & Price per Vessel Category

### 2.1 Quote Availability & Price

- **POST** `/availability/quote`
  - **Description**: Evaluates availability and returns price per vessel category for a given request (creates a `BOOKINGS` row in the background).

#### Request Body

```json
{
  "area_id": "area-uuid",
  "pickup_service_point_id": "sp-uuid-1",
  "dropoff_service_point_id": "sp-uuid-2",
  "requested_departure": "2025-07-01T15:00:00Z",
  "pax_count": 4
}
```

#### Response 200 – Availability & Prices

```json
{
  "booking_id": "booking-uuid",
  "status": "priced",
  "categories": [
    {
      "vessel_category_id": "cat-small",
      "name": "Small boat (up to 6 pax)",
      "price": {
        "amount": "95.00",
        "currency": "EUR"
      }
    },
    {
      "vessel_category_id": "cat-large",
      "name": "Large boat (up to 10 pax)",
      "price": {
        "amount": "140.00",
        "currency": "EUR"
      }
    }
  ]
}
```

#### Response 200 – No Availability

```json
{
  "booking_id": "booking-uuid",
  "status": "no_availability",
  "reason": "NO_VESSEL_FOR_CATEGORY"
}
```

---

## 3. Booking Confirmation & Payment

### 3.1 Select Vessel Category

- **POST** `/bookings/{booking_id}/select-category`
  - **Description**: Locks in the chosen vessel category for the booking.

#### Request Body

```json
{
  "vessel_category_id": "cat-small"
}
```

#### Response 200

```json
{
  "booking_id": "booking-uuid",
  "status": "awaiting_payment",
  "vessel_category_id": "cat-small",
  "quoted_price": {
    "amount": "95.00",
    "currency": "EUR"
  }
}
```

### 3.2 Create Payment Intent for Booking

- **POST** `/bookings/{booking_id}/payment-intents`
  - **Description**: Creates a payment intent for the current quoted price (backed by `PAYMENT_INTENTS`).

#### Request Body

```json
{
  "payment_provider": "stripe"
}
```

#### Response 201

```json
{
  "id": "payment-intent-uuid",
  "booking_id": "booking-uuid",
  "amount": "95.00",
  "currency": "EUR",
  "provider": "stripe",
  "provider_intent_id": "pi_12345",
  "client_secret": "pi_12345_secret_abc",
  "status": "requires_payment_method"
}
```

The client uses `client_secret` with the payment SDK.

### 3.3 Payment Webhook (Provider → Company)

- **POST** `/webhooks/payments`
  - **Description**: Used by the payment provider to notify of success/failure. This is what ultimately triggers captain notification.

#### Request Body (Example for Stripe-like Provider)

```json
{
  "type": "payment_intent.succeeded",
  "data": {
    "object": {
      "id": "pi_12345",
      "metadata": {
        "payment_intent_id": "payment-intent-uuid"
      }
    }
  }
}
```

#### Expected Behavior (Internal)

- Update `PAYMENT_INTENTS.status` (succeeded / failed / refunded).
- Update `BOOKINGS.status`:
  - `paid` on success → trigger captain offers & notifications.
  - `payment_failed` on failure.

Response can be a simple `200 OK` with an empty body.

---

## 4. Booking Status & Captain Info

### 4.1 Get Booking

- **GET** `/bookings/{booking_id}`
  - **Description**: Used by the client to track payment state, captain assignment, and trip completion/cancellation.

#### Response 200 – Before Captain Assigned

```json
{
  "id": "booking-uuid",
  "status": "assigning_captain",
  "area_id": "area-uuid",
  "pickup_service_point_id": "sp-uuid-1",
  "dropoff_service_point_id": "sp-uuid-2",
  "requested_departure": "2025-07-01T15:00:00Z",
  "pax_count": 4,
  "vessel_category_id": "cat-small",
  "quoted_price": {
    "amount": "95.00",
    "currency": "EUR"
  },
  "trip_id": null,
  "captain": null
}
```

#### Response 200 – After Captain Accepts

```json
{
  "id": "booking-uuid",
  "status": "confirmed",
  "area_id": "area-uuid",
  "pickup_service_point_id": "sp-uuid-1",
  "dropoff_service_point_id": "sp-uuid-2",
  "requested_departure": "2025-07-01T15:00:00Z",
  "pax_count": 4,
  "vessel_category_id": "cat-small",
  "quoted_price": {
    "amount": "95.00",
    "currency": "EUR"
  },
  "trip_id": "trip-uuid",
  "captain": {
    "user_id": "captain-user-id",
    "name": "Kapteeni Jaakko",
    "avatar_url": "https://images.example.com/captains/jaakko.png"
  }
}
```

#### Response 200 – No Captain Found in Time

```json
{
  "id": "booking-uuid",
  "status": "no_captain",
  "trip_id": null,
  "captain": null
}
```

---

## 5. Trip Status (Success / Cancellation)

### 5.1 Get Trip

- **GET** `/trips/{trip_id}`

#### Response 200

```json
{
  "id": "trip-uuid",
  "booking_id": "booking-uuid",
  "status": "completed", // or pending/started/onboard/in_progress/...
  "area_id": "area-uuid",
  "vessel_id": "vessel-uuid",
  "vessel_name": "Sea Breeze",
  "vessel_category_id": "cat-small",
  "captain": {
    "user_id": "captain-user-id",
    "name": "Kapteeni Jaakko",
    "avatar_url": "https://images.example.com/captains/jaakko.png"
  },
  "start_time": "2025-07-01T15:05:00Z",
  "end_time": "2025-07-01T15:45:00Z"
}
```

#### Possible Status Values (MVP)

- `pending`
- `started`
- `onboard`
- `in_progress`
- `completed`
- `cancelled_by_captain`
- `cancelled_by_passenger`
- `passenger_no_show`

### 5.2 Trip Outcome Reflected via Booking

- **GET** `/bookings/{booking_id}`

#### Example Completed Trip

```json
{
  "id": "booking-uuid",
  "status": "confirmed",
  "trip": {
    "id": "trip-uuid",
    "status": "completed",
    "start_time": "2025-07-01T15:05:00Z",
    "end_time": "2025-07-01T15:45:00Z"
  },
  "captain": {
    "user_id": "captain-user-id",
    "name": "Kapteeni Jaakko",
    "avatar_url": "https://images.example.com/captains/jaakko.png"
  }
}
```

#### Example Cancelled

```json
{
  "id": "booking-uuid",
  "status": "cancelled_by_passenger",
  "trip": null,
  "captain": null
}
```

or

```json
{
  "id": "booking-uuid",
  "status": "confirmed",
  "trip": {
    "id": "trip-uuid",
    "status": "cancelled_by_captain"
  },
  "captain": {
    "user_id": "captain-user-id",
    "name": "Kapteeni Jaakko",
    "avatar_url": "https://images.example.com/captains/jaakko.png"
  }
}
```
