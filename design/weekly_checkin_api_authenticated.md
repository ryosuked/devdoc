# Weekly Check-in API (Authenticated User)

This API provides weekly check-in history for the **authenticated user only**, using cursor-based pagination. The client can specify any date, and the API will return the check-ins for the full week containing that date.

---

## 🔐 Authentication

- **Method**: Bearer token authentication (e.g., OAuth2 or JWT)
- **Header Example**:

```
Authorization: Bearer YOUR_ACCESS_TOKEN
```

---

## 📘 Endpoint

```
GET /me/weekly_checkins
```

---

## 🔍 Query Parameters

| Parameter | Type   | Required | Description |
|-----------|--------|----------|-------------|
| `date`    | string (ISO8601) | Yes | A date included in the target week (e.g., `2025-08-05`). The API will return data for the entire week (Monday to Sunday) containing this date. |

---

## 📗 Response Example

```json
{
  "data": [
    {
      "checked_in_at": "2025-08-05T10:00:00Z",
      "location": "Tokyo Office"
    },
    {
      "checked_in_at": "2025-08-07T09:30:00Z",
      "location": "Remote"
    }
  ],
  "range": {
    "start_date": "2025-08-04",
    "end_date": "2025-08-10"
  },
  "navigation": {
    "prev_week": {
      "label": "先週",
      "date": "2025-07-29",
      "start_date": "2025-07-28",
      "end_date": "2025-08-03"
    },
    "this_week": {
      "label": "今週",
      "date": "2025-08-05",
      "start_date": "2025-08-04",
      "end_date": "2025-08-10"
    },
    "next_week": {
      "label": "翌週",
      "date": "2025-08-12",
      "start_date": "2025-08-11",
      "end_date": "2025-08-17"
    }
  }
}
```

---

## 🔁 Pagination Examples

- Previous week:
  ```
  GET /me/weekly_checkins?date=2025-07-29
  ```

- This week:
  ```
  GET /me/weekly_checkins?date=2025-08-05
  ```

- Next week:
  ```
  GET /me/weekly_checkins?date=2025-08-12
  ```

---

## 📝 Notes

- The API only returns data for the authenticated user.
- A "week" is defined as starting on **Monday** and ending on **Sunday**.
- The `navigation` object provides cursors for weekly navigation.
- Dates are assumed to be in UTC unless otherwise specified.


---

# Register Today's Check-in (Authenticated User)

This API allows the **authenticated user** to register a check-in log for the current day.  
For testing or validation purposes, clients can optionally specify the current date via a custom HTTP header.

---

## 🔐 Authentication

- **Method**: Bearer token authentication
- **Header Example**:

```
Authorization: Bearer YOUR_ACCESS_TOKEN
```

---

## 🧾 Custom Header (Optional)

| Header | Type   | Description |
|--------|--------|-------------|
| `X-Client-Date` | string (ISO8601) | Optional client-defined date (e.g., `2025-08-05`). Used for testing or simulating check-ins on a specific day. |

---

## 📘 Endpoint

```
POST /me/checkins
```

---

## 📝 Request Body

```json
{
  "location": "Tokyo Office"
}
```

- `location`: (string, required) The location of the check-in.

---

## 📗 Response Example

```json
{
  "checked_in_at": "2025-08-05T09:00:00Z",
  "location": "Tokyo Office",
  "user_id": "user_001"
}
```

---

## 📝 Notes

- If `X-Client-Date` is specified, the check-in will be recorded for that date (at the current time).
- If the header is omitted, the server uses the current date and time (UTC) as the check-in timestamp.
- This API is intended for use by the authenticated user only.
