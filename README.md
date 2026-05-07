# ☁️ ACAS Cloud API Documentation (v1)

**Base URL:** `https://acascloud.com/`

---

# 📑 Table of Contents
1. Authentication
2. Integration Workflow
3. User Endpoints
   - Provision a New User
   - List All Registered Users
   - Get User Attendance History
4. Device & Hardware Endpoints
   - List All Registered Devices
   - Get Device Details
   - Update Device Name & Location
   - Get Device Attendance History
5. General Attendance Endpoints
   - Get Clock-Ins by Time Range
   - Get Clock-Outs by Time Range
   - Get All Attendance Logs
   - Get All Device Attendance Logs

---

# 1. Authentication

Security is enforced using a static API token generated for each HRIS partner from the ACAS Partner Dashboard.

Because the token is passed in the request body, **all API endpoints use the `POST` method**.

Every request must include:

```http
Content-Type: application/json
```

And include the `api_token` field inside the JSON request body.

## Example Request Payload

```json
{
  "api_token": "your_secure_api_token_here"
}
```

## Common Status Codes

| Code | Meaning |
|---|---|
| `200` | Request successful |
| `201` | Resource created successfully |
| `400` | Invalid request body |
| `401` | Missing authentication token |
| `403` | Invalid token or unauthorized |
| `404` | Resource not found |
| `405` | Invalid HTTP method |

---

# 2. Integration Workflow

1. HRIS partner provisions a user using `/users/provision`
2. ACAS Cloud syncs the user to connected devices
3. User completes biometric enrollment on the physical device
4. Device syncs biometric enrollment status back to ACAS Cloud

---

# 3. User Endpoints

## 3.1 Provision a New User

**POST** `/users/provision`

Pushes a new employee from the HRIS system to ACAS Cloud.

### Request Body

| Field | Type | Required | Description |
|---|---|---|---|
| `api_token` | string | Yes | Partner API token |
| `hris_user_id` | string | Yes | Unique HRIS user ID |
| `first_name` | string | Yes | User first name |
| `last_name` | string | Yes | User last name |

### Example Request

```json
{
  "api_token": "your_secure_api_token_here",
  "hris_user_id": "EMP-8492",
  "first_name": "John",
  "last_name": "Doe"
}
```

### Example Response

```json
{
  "success": true,
  "data": {
    "acas_user_id": 1042,
    "hris_user_id": "EMP-8492",
    "status": "pending_enrollment"
  }
}
```

---

## 3.2 List All Registered Users

**POST** `/users/list`

Returns all users associated with the authenticated partner.

### Request Body

| Field | Type | Required | Description |
|---|---|---|---|
| `api_token` | string | Yes | Partner API token |
| `status` | string | No | Filter by `enrolled` or `pending` |
| `limit` | integer | No | Default `50`, max `500` |

### Example Request

```json
{
  "api_token": "your_secure_api_token_here",
  "status": "enrolled",
  "limit": 50
}
```

### Example Response

```json
{
  "success": true,
  "data": [
    {
      "hris_user_id": "EMP-8492",
      "first_name": "John",
      "last_name": "Doe",
      "device_first_name": "John",
      "device_last_name": "Doe",
      "is_enrolled": true,
      "biometrics": {
        "has_face": true,
        "has_fingerprint": true
      },
      "last_sync": "2024-03-08T22:38:22Z"
    }
  ]
}
```

---

## 3.3 Get User Attendance History

**POST** `/users/{hris_user_id}/attendance`

Returns attendance logs for a specific HRIS user.

### Request Body

| Field | Type | Required | Description |
|---|---|---|---|
| `api_token` | string | Yes | Partner API token |
| `limit` | integer | No | Default `50`, max `500` |

### Example Request

```json
{
  "api_token": "your_secure_api_token_here",
  "limit": 50
}
```

### Example Response

```json
{
  "success": true,
  "data": [
    {
      "log_id": "log_DEV-12345_1001",
      "device_id": "DEV-12345",
      "timestamp": "2024-03-08T08:30:15+00:00",
      "action": 1,
      "action_type": "CLOCK_IN",
      "method": "Face"
    }
  ]
}
```

---

# 4. Device & Hardware Endpoints

## 4.1 List All Registered Devices

**POST** `/devices/list`

Returns all devices registered under the authenticated partner.

### Request Body

| Field | Type | Required | Description |
|---|---|---|---|
| `api_token` | string | Yes | Partner API token |

### Example Response

```json
{
  "success": true,
  "data": [
    {
      "device_id": "DEV-12345",
      "name": "Main Entrance Station",
      "location": "Umuahia North, Abia State",
      "last_ping": "2024-03-08T22:35:00+00:00",
      "firmware_version": "v1.0.4"
    }
  ]
}
```

> Note: Device `status` is no longer returned by this endpoint.

---

## 4.2 Get Device Details

**POST** `/devices/{device_id}/details`

Returns detailed information about a specific device.

### Request Body

| Field | Type | Required | Description |
|---|---|---|---|
| `api_token` | string | Yes | Partner API token |

### Example Request

```json
{
  "api_token": "your_secure_api_token_here"
}
```

### Example Response

```json
{
  "success": true,
  "data": {
    "device_id": "DEV-12345",
    "name": "Main Entrance Station",
    "last_ping": "2024-03-08T22:35:00+00:00",
    "hardware": {
      "cpu_temp_c": 42.5,
      "memory_usage_mb": 1200,
      "storage_usage_gb": 8.5
    },
    "connectivity": {
      "active_connection": "wifi",
      "wifi": {
        "ssid": "Office_Network_5G",
        "signal_strength": 85
      },
      "sim": {
        "carrier": "MTN",
        "signal_quality": "Good",
        "phone_number": "+2348030000000",
        "sms_fallback_enabled": true
      }
    }
  }
}
```

> Note: Device `status` is no longer included in the device details response.

---

## 4.3 Update Device Name & Location

**POST** `/devices/{device_id}/update`

Updates the display name and location of a device.

### Request Body

| Field | Type | Required | Description |
|---|---|---|---|
| `api_token` | string | Yes | Partner API token |
| `name` | string | Yes | New device name |
| `location` | string | Yes | New device location |

### Example Request

```json
{
  "api_token": "your_secure_api_token_here",
  "name": "Reception Terminal",
  "location": "Lagos HQ"
}
```

### Example Success Response

```json
{
  "success": true,
  "message": "Device updated successfully."
}
```

### Example Error Response

```json
{
  "success": false,
  "message": "Device not found or no changes made."
}
```

---

## 4.4 Get Device Attendance History

**POST** `/devices/{device_id}/attendance`

Returns attendance logs from a specific device.

### Request Body

| Field | Type | Required | Description |
|---|---|---|---|
| `api_token` | string | Yes | Partner API token |
| `start_date` | string | No | Start date filter |
| `end_date` | string | No | End date filter |
| `limit` | integer | No | Default `50`, max `500` |

### Example Response

```json
{
  "success": true,
  "data": [
    {
      "log_id": "log_DEV-12345_1001",
      "hris_user_id": "EMP-8492",
      "first_name": "John",
      "last_name": "Doe",
      "timestamp": "2024-03-08T08:30:15+00:00",
      "action": 1,
      "action_type": "CLOCK_IN",
      "method": "Face"
    }
  ]
}
```

---

# 5. General Attendance Endpoints

## 5.1 Get Clock-Ins by Time Range

**POST** `/attendance/clock-ins`

Returns all clock-in records.

### Optional Filters

- `device_id`
- `start_date`
- `end_date`
- `limit`

### Example Response

```json
{
  "success": true,
  "data": [
    {
      "log_id": "log_DEV-12345_1001",
      "hris_user_id": "EMP-8492",
      "device_id": "DEV-12345",
      "timestamp": "2024-03-08T08:30:15+00:00",
      "action": 1,
      "action_type": "CLOCK_IN",
      "method": "Face"
    }
  ]
}
```

---

## 5.2 Get Clock-Outs by Time Range

**POST** `/attendance/clock-outs`

Returns all clock-out records.

### Optional Filters

- `device_id`
- `start_date`
- `end_date`
- `limit`

### Example Response

```json
{
  "success": true,
  "data": [
    {
      "log_id": "log_DEV-12345_1002",
      "hris_user_id": "EMP-8492",
      "device_id": "DEV-12345",
      "timestamp": "2024-03-08T17:45:10+00:00",
      "action": 0,
      "action_type": "CLOCK_OUT",
      "method": "Fingerprint"
    }
  ]
}
```

---

## 5.3 Get All Attendance Logs

**POST** `/attendance/all`

Returns attendance logs across all devices.

### Optional Filters

| Field | Type |
|---|---|
| `start_date` | string |
| `end_date` | string |
| `hris_user_id` | string |
| `no_limit` | boolean |

---

## 5.4 Get All Device Attendance Logs

**POST** `/attendance/device/all`

Returns attendance logs across all devices with optional filtering.

### Optional Filters

| Field | Type |
|---|---|
| `device_id` | string |
| `hris_user_id` | string |
| `start_date` | string |
| `end_date` | string |
| `no_limit` | boolean |

---

# Notes

- All timestamps are returned in ISO 8601 format.
- Maximum supported request limit is `500`.
- All endpoints require `POST`.
- Attendance `action` values:
  - `1` = CLOCK_IN
  - `0` = CLOCK_OUT
