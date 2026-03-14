
***

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
5. [Webhooks (Real-time Sync)](#5-webhooks-real-time-sync)

---

## 1. Authentication

Security is enforced using a static API token generated for each HRIS partner from the ACAS Partner Dashboard. All requests must include this token in the `Authorization` header using the `Bearer` schema.

**Example Header:**
```http
Authorization: Bearer <YOUR_API_TOKEN>
Content-Type: application/json
```

**Common Status Codes:**
* `200 OK` - Request successful.
* `201 Created` - Resource successfully created.
* `401 Unauthorized` - Token is missing or invalid.
* `403 Forbidden` - Token is valid but lacks permissions for the requested resource.

---

## 2. Integration Workflow

Because the HRIS partner owns the employee data, the workflow for getting an HRIS employee onto the physical ACAS device is as follows:

1. **Provisioning**: The HRIS partner calls `POST /users` passing the employee's name and their internal `hris_user_id`.
2. **Cloud Sync**: The ACAS Cloud receives this and automatically pushes it to the physical device(s) in a "Pending Enrollment" state.
3. **Hardware Enrollment**: The local admin selects the pending user on the physical device screen and completes the Face/Fingerprint scan.
4. **Status Update**: The device updates the ACAS Cloud, marking `is_enrolled: true`.

---

## 3. User Endpoints

### 3.1 Provision a New User
**`POST`** `/users`

Pushes a new employee from the HRIS system to the ACAS Cloud, queuing them up for biometric enrollment on the physical devices.

#### Request Body
| Field | Type | Required | Description |
| :--- | :--- | :---: | :--- |
| `hris_user_id` | `string` | Yes | The unique identifier for the user in the HRIS system. |
| `first_name` | `string` | Yes | User's first name. |
| `last_name` | `string` | Yes | User's last name. |
| `allowed_devices`| `array` | No | Array of Device IDs. If omitted, user syncs to all devices assigned to the partner. |

```json
{
  "hris_user_id": "EMP-8492",
  "first_name": "John",
  "last_name": "Doe",
  "allowed_devices": ["DEV-12345", "DEV-67890"] 
}
```

#### Response (201 Created)
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
**`GET`** `/users`

Retrieves a list of all users, allowing the HRIS to see who has successfully enrolled their biometrics.

#### Query Parameters
| Parameter | Type | Required | Description |
| :--- | :--- | :---: | :--- |
| `status` | `string` | No | Filter by `enrolled` or `pending`. |
| `limit` | `integer`| No | Default `50`, Max `100`. |
| `page` | `integer`| No | Default `1`. |

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
**`GET`** `/users/{hris_user_id}/attendance`

Fetch the clock-in and clock-out logs for a specific employee using their HRIS ID.

#### Query Parameters
| Parameter | Type | Required | Description |
| :--- | :--- | :---: | :--- |
| `start_date` | `string` | No | ISO 8601 format (e.g., `2024-03-01T00:00:00Z`). |
| `end_date` | `string` | No | ISO 8601 format. |
| `limit` | `integer`| No | Default `50`, Max `200`. |

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
**`GET`** `/devices`

View all physical ACAS devices assigned to the HRIS partner, including their active/online status.

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
**`GET`** `/devices/{device_id}`

Fetch deep diagnostic information about a specific device, including its current network connectivity, SIM details, and hardware vitals.

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
**`GET`** `/devices/{device_id}/attendance`

Retrieve an unfiltered stream of attendance logs strictly from a specific device. Useful for HRIS partners who want to pull bulk logs daily and process the users on their own end.

#### Query Parameters
| Parameter | Type | Required | Description |
| :--- | :--- | :---: | :--- |
| `start_date` | `string` | No | ISO 8601 format. |
| `end_date` | `string` | No | ISO 8601 format. |
| `limit` | `integer`| No | Default `100`, Max `500`. |

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
