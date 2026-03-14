

# ☁️ ACAS Cloud API Documentation (v1)

**Base URL:** `https://api.acascloud.com/v1`

---

## 📑 Table of Contents
1. [Authentication](#1-authentication)
2. [Integration Workflow](#2-integration-workflow)
3. [User Endpoints](#3-user-endpoints)
    - [Provision a New User](#31-provision-a-new-user)
    - [List All Registered Users](#32-list-all-registered-users)
    - [Get User Attendance History](#33-get-user-attendance-history)
4. [Device & Hardware Endpoints](#4-device--hardware-endpoints)
    - [List All Registered Devices](#41-list-all-registered-devices)
    - [Get Device Details & SIM Status](#42-get-device-details--sim-status)
    - [Get Device Attendance History](#43-get-device-attendance-history)
5. [General Attendance Endpoints](#5-general-attendance-endpoints)
    - [Get Clock-Ins by Time Range](#51-get-clock-ins-by-time-range)
    - [Get Clock-Outs by Time Range](#52-get-clock-outs-by-time-range)

---

## 1. Authentication

Security is enforced using a static API token generated for each HRIS partner from the ACAS Partner Dashboard. 

Because the token is passed in the request body, **all API endpoints use the `POST` method**. Every request must include `Content-Type: application/json` in the header, and the `api_token` in the root of the JSON body.

**Example Request Payload:**
```json
{
  "api_token": "your_secure_api_token_here",
  "...": "other_parameters"
}
```

**Common Status Codes:**
* `200 OK` - Request successful.
* `401 Unauthorized` - Token is missing or invalid.
* `403 Forbidden` - Token is valid but lacks permissions for the requested resource.

---

## 2. Integration Workflow

Because the HRIS partner owns the employee data, the workflow for getting an HRIS employee onto the physical ACAS device is as follows:

1. **Provisioning**: The HRIS partner calls `POST /users/provision` passing the employee's name and their internal `hris_user_id`.
2. **Cloud Sync**: The ACAS Cloud receives this and automatically pushes it to the physical device(s) in a "Pending Enrollment" state.
3. **Hardware Enrollment**: The local admin selects the pending user on the physical device screen and completes the Face/Fingerprint scan.
4. **Status Update**: The device updates the ACAS Cloud, marking `is_enrolled: true`.

---

## 3. User Endpoints

### 3.1 Provision a New User
**`POST`** `/users/provision`

Pushes a new employee from the HRIS system to the ACAS Cloud, queuing them up for biometric enrollment on the physical devices.

#### Request Body
| Field | Type | Required | Description |
| :--- | :--- | :---: | :--- |
| `api_token` | `string` | Yes | Your partner API Token. |
| `hris_user_id` | `string` | Yes | The unique identifier for the user in the HRIS system. |
| `first_name` | `string` | Yes | User's first name. |
| `last_name` | `string` | Yes | User's last name. |
| `allowed_devices`| `array` | No | Array of Device IDs. If omitted, user syncs to all devices assigned to the partner. |

```json
{
  "api_token": "your_secure_api_token_here",
  "hris_user_id": "EMP-8492",
  "first_name": "John",
  "last_name": "Doe",
  "allowed_devices": ["DEV-12345", "DEV-67890"] 
}
```

#### Response (200 OK)
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

### 3.2 List All Registered Users
**`POST`** `/users/list`

Retrieves a list of all users, allowing the HRIS to see who has successfully enrolled their biometrics.

#### Request Body
| Field | Type | Required | Description |
| :--- | :--- | :---: | :--- |
| `api_token` | `string` | Yes | Your partner API Token. |
| `status` | `string` | No | Filter by `enrolled` or `pending`. |
| `limit` | `integer`| No | Default `50`, Max `100`. |
| `page` | `integer`| No | Default `1`. |

```json
{
  "api_token": "your_secure_api_token_here",
  "status": "enrolled",
  "limit": 50,
  "page": 1
}
```

#### Response (200 OK)
```json
{
  "success": true,
  "pagination": {
    "total": 145,
    "page": 1,
    "pages": 3
  },
  "data": [
    {
      "hris_user_id": "EMP-8492",
      "first_name": "John",
      "last_name": "Doe",
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

### 3.3 Get User Attendance History
**`POST`** `/users/{hris_user_id}/attendance`

Fetch the clock-in and clock-out logs for a specific employee using their HRIS ID in the URL.

#### Request Body
| Field | Type | Required | Description |
| :--- | :--- | :---: | :--- |
| `api_token` | `string` | Yes | Your partner API Token. |
| `start_date` | `string` | No | ISO 8601 format (e.g., `2024-03-01T00:00:00Z`). |
| `end_date` | `string` | No | ISO 8601 format. |
| `limit` | `integer`| No | Default `50`, Max `200`. |

```json
{
  "api_token": "your_secure_api_token_here",
  "start_date": "2024-03-01T00:00:00Z",
  "limit": 50
}
```

#### Response (200 OK)
> **Note:** `action: 1` represents Clock In, `action: 0` represents Clock Out.

```json
{
  "success": true,
  "data": [
    {
      "log_id": "log_99283",
      "device_id": "DEV-12345",
      "timestamp": "2024-03-08T08:30:15Z",
      "action": 1, 
      "action_type": "CLOCK_IN",
      "method": "Face"
    },
    {
      "log_id": "log_99304",
      "device_id": "DEV-12345",
      "timestamp": "2024-03-08T17:45:10Z",
      "action": 0,
      "action_type": "CLOCK_OUT",
      "method": "Fingerprint"
    }
  ]
}
```

---

## 4. Device & Hardware Endpoints

### 4.1 List All Registered Devices
**`POST`** `/devices/list`

View all physical ACAS devices assigned to the HRIS partner, including their active/online status.

#### Request Body
| Field | Type | Required | Description |
| :--- | :--- | :---: | :--- |
| `api_token` | `string` | Yes | Your partner API Token. |

```json
{
  "api_token": "your_secure_api_token_here"
}
```

#### Response (200 OK)
```json
{
  "success": true,
  "data": [
    {
      "device_id": "DEV-12345",
      "name": "Main Entrance Station",
      "location": "Umuahia North, Abia State",
      "status": "online",
      "last_ping": "2024-03-08T22:35:00Z",
      "firmware_version": "v1.0.4"
    }
  ]
}
```

### 4.2 Get Device Details & SIM Status
**`POST`** `/devices/{device_id}/details`

Fetch deep diagnostic information about a specific device, including its current network connectivity, SIM details, and hardware vitals.

#### Request Body
| Field | Type | Required | Description |
| :--- | :--- | :---: | :--- |
| `api_token` | `string` | Yes | Your partner API Token. |

```json
{
  "api_token": "your_secure_api_token_here"
}
```

#### Response (200 OK)
```json
{
  "success": true,
  "data": {
    "device_id": "DEV-12345",
    "name": "Main Entrance Station",
    "status": "online",
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
        "carrier": "MTN Nigeria",
        "signal_quality": "Good",
        "phone_number": "+2348030000000",
        "sms_fallback_enabled": true
      }
    }
  }
}
```

### 4.3 Get Device Attendance History
**`POST`** `/devices/{device_id}/attendance`

Retrieve an unfiltered stream of attendance logs strictly from a specific device. Useful for HRIS partners who want to pull bulk logs daily and process the users on their own end.

#### Request Body
| Field | Type | Required | Description |
| :--- | :--- | :---: | :--- |
| `api_token` | `string` | Yes | Your partner API Token. |
| `start_date` | `string` | No | ISO 8601 format. |
| `end_date` | `string` | No | ISO 8601 format. |
| `limit` | `integer`| No | Default `100`, Max `500`. |

```json
{
  "api_token": "your_secure_api_token_here",
  "start_date": "2024-03-01T00:00:00Z",
  "limit": 100
}
```

#### Response (200 OK)
```json
{
  "success": true,
  "data": [
    {
      "log_id": "log_99283",
      "hris_user_id": "EMP-8492",
      "first_name": "John",
      "last_name": "Doe",
      "timestamp": "2024-03-08T08:30:15Z",
      "action": 1,
      "action_type": "CLOCK_IN",
      "method": "Face"
    }
  ]
}
```

---

## 5. General Attendance Endpoints

These endpoints allow you to query global attendance records across all your company's users and devices. This is particularly useful for generating daily "Who is present today?" reports.

### 5.1 Get Clock-Ins by Time Range
**`POST`** `/attendance/clock-ins`

Retrieve only the **Clock In** records for a specific date or time range. You can optionally filter this down to a specific device.

#### Request Body
| Field | Type | Required | Description |
| :--- | :--- | :---: | :--- |
| `api_token` | `string` | Yes | Your partner API Token. |
| `start_date` | `string` | No | ISO 8601 format. Defaults to beginning of the current day (UTC) if omitted. |
| `end_date` | `string` | No | ISO 8601 format. Defaults to current time if omitted. |
| `device_id` | `string` | No | Optional. Pass a specific device ID to only show clock-ins for that device. |
| `limit` | `integer`| No | Default `100`, Max `500`. |

```json
{
  "api_token": "your_secure_api_token_here",
  "start_date": "2024-03-08T00:00:00Z",
  "end_date": "2024-03-08T23:59:59Z",
  "device_id": "DEV-12345"
}
```

#### Response (200 OK)
```json
{
  "success": true,
  "data": [
    {
      "log_id": "log_99283",
      "hris_user_id": "EMP-8492",
      "first_name": "John",
      "last_name": "Doe",
      "device_id": "DEV-12345",
      "timestamp": "2024-03-08T08:30:15Z",
      "action": 1,
      "action_type": "CLOCK_IN",
      "method": "Face"
    },
    {
      "log_id": "log_99284",
      "hris_user_id": "EMP-1022",
      "first_name": "Jane",
      "last_name": "Smith",
      "device_id": "DEV-12345",
      "timestamp": "2024-03-08T08:45:00Z",
      "action": 1,
      "action_type": "CLOCK_IN",
      "method": "Fingerprint"
    }
  ]
}
```

### 5.2 Get Clock-Outs by Time Range
**`POST`** `/attendance/clock-outs`

Retrieve only the **Clock Out** records for a specific date or time range. You can optionally filter this down to a specific device.

#### Request Body
| Field | Type | Required | Description |
| :--- | :--- | :---: | :--- |
| `api_token` | `string` | Yes | Your partner API Token. |
| `start_date` | `string` | No | ISO 8601 format. Defaults to beginning of the current day (UTC) if omitted. |
| `end_date` | `string` | No | ISO 8601 format. Defaults to current time if omitted. |
| `device_id` | `string` | No | Optional. Pass a specific device ID to only show clock-outs for that device. |
| `limit` | `integer`| No | Default `100`, Max `500`. |

```json
{
  "api_token": "your_secure_api_token_here",
  "start_date": "2024-03-08T00:00:00Z",
  "end_date": "2024-03-08T23:59:59Z"
}
```

#### Response (200 OK)
```json
{
  "success": true,
  "data": [
    {
      "log_id": "log_99304",
      "hris_user_id": "EMP-8492",
      "first_name": "John",
      "last_name": "Doe",
      "device_id": "DEV-67890",
      "timestamp": "2024-03-08T17:45:10Z",
      "action": 0,
      "action_type": "CLOCK_OUT",
      "method": "Fingerprint"
    }
  ]
}
```
```
