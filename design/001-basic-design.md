# IDBuilder Basic Design

| Metadata       | Value                                              |
|----------------|---------------------------------------------------|
| Author(s)      | tiltwind                                          |
| Status         | Draft                                             |
| Created        | 2023-01-11                                        |
| Last Updated   | 2023-01-11                                        |
| Discussion     | https://github.com/idbuilder/proposal/issues/1    |

---

## Table of Contents

1. [Abstract](#abstract)
2. [Background](#background)
3. [Goals and Non-Goals](#goals-and-non-goals)
4. [Architecture](#architecture)
5. [ID Types Overview](#id-types-overview)
6. [API Specification](#api-specification)
   - [API Endpoints Summary](#api-endpoints-summary)
   - [Authentication API](#0-authentication-api)
   - [Auto-Increment Number ID](#1-auto-increment-number-id)
     - [Configuration API](#11-configuration-api)
     - [ID Generation API](#12-id-generation-api)
   - [Snowflake Style Number ID](#2-snowflake-style-number-id)
     - [Configuration API](#21-configuration-api)
     - [ID Generation API](#22-id-generation-api)
   - [Fixed Format String ID](#3-fixed-format-string-id)
     - [Configuration API](#31-configuration-api)
     - [Part Types](#32-part-types)
     - [ID Generation API](#33-id-generation-api)
7. [Error Handling](#error-handling)
8. [Security Considerations](#security-considerations)

---

## Abstract

This document describes the basic design of **IDBuilder**, a distributed ID generation service that supports multiple ID types:

- **Auto-increment number ID** — Sequential numeric identifiers with configurable delta
- **Snowflake style number ID** — Time-based distributed unique identifiers
- **Fixed format string ID** — Customizable string identifiers with composable parts

---

## Background

Existing distributed ID generation solutions typically focus on a single ID type (e.g., Twitter's Snowflake, UUID). However, real-world applications often require diverse ID formats:

- Sequential order numbers for business transactions
- Globally unique numeric IDs for database sharding
- Human-readable string IDs with embedded business semantics (e.g., `INV20230101-0001`)

**IDBuilder** aims to provide a unified solution that addresses all these requirements through a single, flexible service.

---

## Goals and Non-Goals

### Goals

- Provide a unified API for generating multiple types of distributed IDs
- Support high availability and horizontal scalability
- Ensure ID uniqueness across distributed workers
- Allow flexible configuration for different business requirements

### Non-Goals

- UUID generation (use standard libraries instead)
- Persistent ID storage or retrieval
- ID validation or verification services

---

## Architecture

```
┌─────────────┐                          ┌─────────────┐                    ┌─────────────┐
│             │  ──── ID Request ────▶   │             │                    │             │
│ Client/SDK  │                          │   Worker    │ ◀──── Config ────  │ Controller  │
│             │  ◀─── ID Response ────   │             │                    │             │
└─────────────┘                          └─────────────┘                    └─────────────┘
```

### Components

| Component      | Responsibility                                                    |
|----------------|-------------------------------------------------------------------|
| **Controller** | Manages ID generation rules, authentication tokens, and worker coordination |
| **Worker**     | Processes ID generation requests and ensures uniqueness           |
| **Client/SDK** | Provides application interface for requesting IDs                 |

---

## ID Types Overview

| ID Type                    | Format Example                | Generation Location | Use Case                          |
|----------------------------|-------------------------------|---------------------|-----------------------------------|
| Auto-increment Number      | `1, 2, 3, 4, ...`            | Server (Worker)     | Sequential order numbers          |
| Snowflake Style Number     | `6982386234567892992`        | Client (SDK)        | Distributed unique IDs            |
| Fixed Format String        | `P20230101-1234`             | Server (Worker)     | Human-readable business IDs       |

---

## API Specification

### Common Request Headers

| Header          | Required | Description                              |
|-----------------|----------|------------------------------------------|
| `Authorization` | Yes      | Bearer token for authentication          |
| `Content-Type`  | No       | `application/json` (for POST requests)   |

**Token Types:**

| Token Type    | Header Format                         | Used For                                    |
|---------------|---------------------------------------|---------------------------------------------|
| Admin Token   | `Authorization: Bearer <admin_token>` | Configuration APIs (`/v1/config/*`), Token API (`/v1/auth/token`) |
| Key Token     | `Authorization: Bearer <key_token>`   | ID Generation APIs (`/v1/id/*`)             |

### Common Response Format

All successful responses follow this structure:

```json
{
  "code": 0,
  "message": "success",
  "data": { ... }
}
```

### API Endpoints Summary

| Category       | ID Type     | Endpoint                  | Method | Auth Token  | Description                              |
|----------------|-------------|---------------------------|--------|-------------|------------------------------------------|
| Authentication | -           | `/v1/auth/token`          | GET    | Admin Token | Get or create key token (auto-creates if not exists) |
|                | -           | `/v1/auth/tokenreset`     | GET    | Admin Token | Reset (regenerate) key token             |
| Configuration  | Increment   | `/v1/config/increment`    | POST   | Admin Token | Create/Update increment ID config        |
|                |             | `/v1/config/increment`    | GET    | Admin Token | Get increment ID config                  |
|                | Snowflake   | `/v1/config/snowflake`    | POST   | Admin Token | Create/Update snowflake ID config        |
|                |             | `/v1/config/snowflake`    | GET    | Admin Token | Get snowflake ID config                  |
|                | Formatted   | `/v1/config/formatted`    | POST   | Admin Token | Create/Update formatted ID config        |
|                |             | `/v1/config/formatted`    | GET    | Admin Token | Get formatted ID config                  |
| ID Generation  | Increment   | `/v1/id/increment`        | GET    | Key Token   | Generate auto-increment IDs              |
|                | Snowflake   | `/v1/id/snowflake`        | GET    | Key Token   | Get snowflake config for client generation |
|                | Formatted   | `/v1/id/formatted`        | GET    | Key Token   | Generate formatted string IDs            |

---

### 0. Authentication API

Manage key tokens for accessing ID generation APIs. These endpoints require admin token authentication.

**Token Format:** Key tokens are 64-character URL-safe base64 strings (e.g., `A1b2C3d4E5f6G7h8I9j0K1l2M3n4O5p6Q7r8S9t0U1v2W3x4Y5z6A7b8C9d0E1f2`).

#### Get or Create Key Token

Retrieve a key token for the specified key. If no valid token exists for the key, a new one is automatically created.

```
GET /v1/auth/token
```

**Request Parameters:**

| Parameter | Type   | Required | Description                                |
|-----------|--------|----------|--------------------------------------------|
| `key`     | string | Yes      | Identifier for the ID configuration        |

**Request Example:**

```bash
curl -X GET "http://idbuilder/v1/auth/token?key=order-id" \
  -H "Authorization: Bearer <admin_token>"
```

**Response Parameters:**

| Parameter    | Type   | Description                                         |
|--------------|--------|-----------------------------------------------------|
| `key`        | string | The requested key identifier                        |
| `token`      | string | Key token for ID generation API authentication (64 chars) |
| `token_type` | string | Token type (always "key")                           |
| `expires_at` | string | Token expiration time (ISO 8601 format)             |

**Response Example:**

```json
{
  "code": 0,
  "message": "success",
  "data": {
    "key": "order-id",
    "token": "A1b2C3d4E5f6G7h8I9j0K1l2M3n4O5p6Q7r8S9t0U1v2W3x4Y5z6A7b8C9d0E1f2",
    "token_type": "key",
    "expires_at": "2024-01-11T10:00:00Z"
  }
}
```

#### Reset Key Token

Regenerate a new token for the specified key, invalidating any existing token for that key.

```
GET /v1/auth/tokenreset
```

**Request Parameters:**

| Parameter | Type   | Required | Description                                |
|-----------|--------|----------|--------------------------------------------|
| `key`     | string | Yes      | Identifier for the ID configuration        |

**Request Example:**

```bash
curl -X GET "http://idbuilder/v1/auth/tokenreset?key=order-id" \
  -H "Authorization: Bearer <admin_token>"
```

**Response Parameters:**

| Parameter    | Type   | Description                                         |
|--------------|--------|-----------------------------------------------------|
| `key`        | string | The requested key identifier                        |
| `token`      | string | New key token for ID generation API authentication (64 chars) |
| `token_type` | string | Token type (always "key")                           |
| `expires_at` | string | Token expiration time (ISO 8601 format)             |

**Response Example:**

```json
{
  "code": 0,
  "message": "success",
  "data": {
    "key": "order-id",
    "token": "X9y8Z7a6B5c4D3e2F1g0H9i8J7k6L5m4N3o2P1q0R9s8T7u6V5w4X3y2Z1a0B9c8",
    "token_type": "key",
    "expires_at": "2024-02-10T10:00:00Z"
  }
}
```

---

### 1. Auto-Increment Number ID

Generates sequential numeric IDs with configurable increments.

#### 1.1 Configuration API

##### Create/Update Configuration

```
POST /v1/config/increment
```

**Request Body Parameters:**

| Parameter           | Type    | Required | Default | Description                                                |
|---------------------|---------|----------|---------|------------------------------------------------------------|
| `key`               | string  | Yes      | -       | Unique identifier for this configuration                   |
| `name`              | string  | No       | -       | Human-readable name for the configuration                  |
| `base`              | integer | Yes      | -       | Starting number for the sequence                           |
| `delta`             | integer | No       | 1       | Default increment between consecutive IDs                  |
| `max_request_delta` | integer | No       | 100     | Maximum allowed delta per request (prevents abuse)         |
| `rand_delta`        | boolean | No       | false   | Enable randomized delta within the specified range         |

**Request Example:**

```bash
curl -X POST "http://idbuilder/v1/config/increment" \
  -H "Authorization: Bearer <admin_token>" \
  -H "Content-Type: application/json" \
  -d '{
    "key": "order-id",
    "name": "Order ID Generator",
    "base": 1000,
    "delta": 1,
    "max_request_delta": 100,
    "rand_delta": false
  }'
```

**Response Example:**

```json
{
  "code": 0,
  "message": "success"
}
```

##### Get Configuration

```
GET /v1/config/increment
```

**Request Parameters:**

| Parameter | Type   | Required | Description                                |
|-----------|--------|----------|--------------------------------------------|
| `key`     | string | Yes      | Identifier for the ID configuration        |

**Request Example:**

```bash
curl -X GET "http://idbuilder/v1/config/increment?key=order-id" \
  -H "Authorization: Bearer <admin_token>"
```

**Response Example:**

```json
{
  "code": 0,
  "message": "success",
  "data": {
    "key": "order-id",
    "name": "Order ID Generator",
    "base": 1000,
    "current": 1523,
    "delta": 1,
    "max_request_delta": 100,
    "rand_delta": false,
    "created_at": "2023-01-11T10:00:00Z",
    "updated_at": "2023-01-11T15:30:00Z"
  }
}
```

#### 1.2 ID Generation API

##### Request Flow

```
┌─────────────┐      GET /v1/id/increment                   ┌─────────────┐
│             │      Authorization: Bearer <key_token>      │             │
│ Client/SDK  │      key=order-id&size=5&rand_delta=true    │   Worker    │
│             │  ─────────────────────────────────────────▶ │             │
│             │                                             │             │
│             │  ◀─────────────────────────────────────────  │             │
└─────────────┘          {"id": [5, 8, 12, 13, 16]}         └─────────────┘
```

##### API Endpoint

```
GET /v1/id/increment
```

##### Request Parameters

| Parameter    | Type    | Required | Default | Description                                      |
|--------------|---------|----------|---------|--------------------------------------------------|
| `key`        | string  | Yes      | -       | Identifier for the ID configuration              |
| `size`       | integer | No       | 1       | Number of IDs to generate (1-1000)               |
| `delta`      | integer | No       | 1       | Increment between IDs (≤ `max_request_delta`)    |
| `rand_delta` | boolean | No       | false   | Randomize delta within range [1, delta]          |

##### Request Example

```bash
curl -X GET "http://idbuilder/v1/id/increment?key=order-id&size=5&delta=5&rand_delta=true" \
  -H "Authorization: Bearer <key_token>"
```

##### Response Example

```json
{
  "code": 0,
  "message": "success",
  "data": {
    "id": [5, 8, 12, 13, 16]
  }
}
```

---

### 2. Snowflake Style Number ID

Generates time-based distributed unique IDs. The client/SDK fetches configuration once during initialization, then generates IDs locally.

#### ID Structure

```
┌───────────┬─────────────────────────────────────────┬────────────┬──────────────┐
│  Sign Bit │            Timestamp                    │  Worker ID │   Sequence   │
│  (1 bit)  │           (41 bits)                     │  (10 bits) │  (12 bits)   │
└───────────┴─────────────────────────────────────────┴────────────┴──────────────┘
     0       (current_ts - base_ts) in milliseconds      0-1023        0-4095

Total: 64 bits
```

#### 2.1 Configuration API

##### Create/Update Configuration

```
POST /v1/config/snowflake
```

**Request Body Parameters:**

| Parameter        | Type    | Required | Default | Description                                    |
|------------------|---------|----------|---------|------------------------------------------------|
| `key`            | string  | Yes      | -       | Unique identifier for this configuration       |
| `name`           | string  | No       | -       | Human-readable name for the configuration      |
| `skip_size`      | integer | No       | 1       | Number of leading bits to skip (sign bit)      |
| `base_ts`        | integer | Yes      | -       | Base timestamp (epoch) in milliseconds         |
| `ts_size`        | integer | Yes      | -       | Bit width for timestamp portion                |
| `worker_id_size` | integer | Yes      | -       | Bit width for worker ID portion                |
| `seq_size`       | integer | Yes      | -       | Bit width for sequence number portion          |

**Request Example:**

```bash
curl -X POST "http://idbuilder/v1/config/snowflake" \
  -H "Authorization: Bearer <admin_token>" \
  -H "Content-Type: application/json" \
  -d '{
    "key": "user-id",
    "name": "User ID Generator",
    "skip_size": 1,
    "base_ts": 1673606841000,
    "ts_size": 41,
    "worker_id_size": 10,
    "seq_size": 12
  }'
```

**Response Example:**

```json
{
  "code": 0,
  "message": "success"
}
```

##### Get Configuration

```
GET /v1/config/snowflake
```

**Request Parameters:**

| Parameter | Type   | Required | Description                                |
|-----------|--------|----------|--------------------------------------------|
| `key`     | string | Yes      | Identifier for the ID configuration        |

**Request Example:**

```bash
curl -X GET "http://idbuilder/v1/config/snowflake?key=user-id" \
  -H "Authorization: Bearer <admin_token>"
```

**Response Example:**

```json
{
  "code": 0,
  "message": "success",
  "data": {
    "key": "user-id",
    "name": "User ID Generator",
    "skip_size": 1,
    "base_ts": 1673606841000,
    "ts_size": 41,
    "worker_id_size": 10,
    "seq_size": 12,
    "active_workers": 5,
    "created_at": "2023-01-11T10:00:00Z",
    "updated_at": "2023-01-11T15:30:00Z"
  }
}
```

#### 2.2 ID Generation API

The client/SDK requests snowflake configuration to generate IDs locally. The server assigns a unique `worker_id` for each client instance.
The server can generate `worker_id` based on the client's IP address or loop through available worker IDs.
The client/SDK only need to request snowflake configuration once during initialization.

##### Request Flow

```
┌─────────────┐      GET /v1/id/snowflake                   ┌─────────────┐
│             │      Authorization: Bearer <key_token>      │             │
│ Client/SDK  │      key=user-id                            │   Worker    │
│             │  ─────────────────────────────────────────▶ │             │
│             │                                             │             │
│             │  ◀───────────────────────────────────────── │             │
└─────────────┘          {snowflake config + worker_id}     └─────────────┘
      │
      │  Client generates IDs locally using config
      ▼
   ID: 6982386234567892992
```

##### API Endpoint

```
GET /v1/id/snowflake
```

##### Request Parameters

| Parameter | Type   | Required | Description                                |
|-----------|--------|----------|--------------------------------------------|
| `key`     | string | Yes      | Identifier for the ID configuration        |

##### Request Example

```bash
curl -X GET "http://idbuilder/v1/id/snowflake?key=user-id" \
  -H "Authorization: Bearer <key_token>"
```

##### Response Parameters

| Parameter        | Type    | Description                                         |
|------------------|---------|-----------------------------------------------------|
| `skip_size`      | integer | Number of leading bits to skip (sign bit)           |
| `base_ts`        | integer | Base timestamp (epoch) in milliseconds              |
| `ts_size`        | integer | Bit width for timestamp portion                     |
| `worker_id`      | integer | Assigned worker ID for this client instance         |
| `worker_id_size` | integer | Bit width for worker ID portion                     |
| `seq_size`       | integer | Bit width for sequence number portion               |

##### Response Example

```json
{
  "code": 0,
  "message": "success",
  "data": {
    "skip_size": 1,
    "base_ts": 1673606841000,
    "ts_size": 41,
    "worker_id": 5,
    "worker_id_size": 10,
    "seq_size": 12
  }
}
```

---

### 3. Fixed Format String ID

Generates customizable string IDs composed of multiple configurable parts.

#### 3.1 Configuration API

##### Create/Update Configuration

```
POST /v1/config/formatted
```

**Request Body Parameters:**

| Parameter | Type   | Required | Description                                       |
|-----------|--------|----------|---------------------------------------------------|
| `key`     | string | Yes      | Unique identifier for this configuration          |
| `name`    | string | No       | Human-readable name for the configuration         |
| `parts`   | array  | Yes      | Ordered list of part configurations               |

**Request Example:**

```bash
curl -X POST "http://idbuilder/v1/config/formatted" \
  -H "Authorization: Bearer <admin_token>" \
  -H "Content-Type: application/json" \
  -d '{
    "key": "invoice-id",
    "name": "Invoice ID Generator",
    "parts": [
      { "type": "fixed-chars", "value": "INV" },
      { "type": "date-format", "format": "yyyyMMdd", "time_zone": "UTC" },
      { "type": "fixed-chars", "value": "-" },
      {
        "type": "auto-increment",
        "length": 4,
        "length_fixed": true,
        "padding_mode": "prefix",
        "padding_char": "0",
        "reset_scope": "date"
      }
    ]
  }'
```

**Response Example:**

```json
{
  "code": 0,
  "message": "success"
}
```

##### Get Configuration

```
GET /v1/config/formatted
```

**Request Parameters:**

| Parameter | Type   | Required | Description                                |
|-----------|--------|----------|--------------------------------------------|
| `key`     | string | Yes      | Identifier for the ID configuration        |

**Request Example:**

```bash
curl -X GET "http://idbuilder/v1/config/formatted?key=invoice-id" \
  -H "Authorization: Bearer <admin_token>"
```

**Response Example:**

```json
{
  "code": 0,
  "message": "success",
  "data": {
    "key": "invoice-id",
    "name": "Invoice ID Generator",
    "parts": [
      { "type": "fixed-chars", "value": "INV" },
      { "type": "date-format", "format": "yyyyMMdd", "time_zone": "UTC" },
      { "type": "fixed-chars", "value": "-" },
      {
        "type": "auto-increment",
        "length": 4,
        "length_fixed": true,
        "padding_mode": "prefix",
        "padding_char": "0",
        "reset_scope": "date"
      }
    ],
    "sample_id": "INV20230111-0001",
    "created_at": "2023-01-11T10:00:00Z",
    "updated_at": "2023-01-11T15:30:00Z"
  }
}
```

#### 3.2 Part Types

##### `fixed-chars` — Static Characters

| Parameter   | Type   | Required | Description              |
|-------------|--------|----------|--------------------------|
| `type`      | string | Yes      | `"fixed-chars"`          |
| `value`     | string | Yes      | The fixed character(s)   |

**Example:** `"P"`, `"-"`, `"INV"`

##### `fixed-polling-char` — Rotating Character

| Parameter    | Type   | Required | Description                              |
|--------------|--------|----------|------------------------------------------|
| `type`       | string | Yes      | `"fixed-polling-char"`                   |
| `chars_scope`| string | Yes      | Characters to cycle through sequentially |

**Example:** With `chars_scope: "ABC"`, generates `A`, `B`, `C`, `A`, `B`, ...

##### `fixed-random-chars` — Random Characters

| Parameter    | Type    | Required | Description                              |
|--------------|---------|----------|------------------------------------------|
| `type`       | string  | Yes      | `"fixed-random-chars"`                   |
| `chars_scope`| string  | Yes      | Character set for random selection       |
| `length`     | integer | Yes      | Number of random characters to generate  |

**Example:** With `chars_scope: "0123456789ABCDEF"` and `length: 4`, generates `A3F1`, `9B2E`, ...

##### `date-format` — Formatted Date/Time

| Parameter   | Type   | Required | Default | Description                              |
|-------------|--------|----------|---------|------------------------------------------|
| `type`      | string | Yes      | -       | `"date-format"`                          |
| `format`    | string | Yes      | -       | Date format pattern (e.g., `yyyyMMddHHmmssSSS`)   |
| `time_zone` | string | No       | UTC     | Time zone for date formatting            |

**Supported Format Tokens:**

| Token | Description          | Example  |
|-------|----------------------|----------|
| yyyy  | 4-digit year         | 2023     |
| MM    | 2-digit month        | 01-12    |
| dd    | 2-digit day          | 01-31    |
| HH    | 2-digit hour (24h)   | 00-23    |
| mm    | 2-digit minute       | 00-59    |
| ss    | 2-digit second       | 00-59    |
| SSS   | 3-digit millisecond  | 000-999  |

##### `timestamp` — Unix Timestamp (Milliseconds)

| Parameter | Type    | Required | Default | Description                          |
|-----------|---------|----------|---------|--------------------------------------|
| `type`    | string  | Yes      | -       | `"timestamp"`                        |
| `base_ts` | integer | No       | 0       | Base timestamp to subtract           |

##### `unix-seconds` — Unix Timestamp (Seconds)

| Parameter  | Type    | Required | Default | Description                         |
|------------|---------|----------|---------|-------------------------------------|
| `type`     | string  | Yes      | -       | `"unix-seconds"`                    |
| `base_unix`| integer | No       | 0       | Base Unix seconds to subtract       |

##### `auto-increment` — Sequential Number Part

| Parameter     | Type    | Required | Default  | Description                                  |
|---------------|---------|----------|----------|----------------------------------------------|
| `type`        | string  | Yes      | -        | `"auto-increment"`                           |
| `length`      | integer | No       | -        | Fixed length (with padding)                  |
| `length_fixed`| boolean | No       | false    | Enable fixed-length padding                  |
| `padding_mode`| string  | No       | `prefix` | Padding position: `prefix` or `suffix`       |
| `padding_char`| string  | No       | `0`      | Character used for padding                   |
| `number_base` | integer | No       | 10       | Numeric base for formatting (2-36)           |
| `reset_scope` | string  | No       | `none`   | When to reset: `none`, `year`, `month`, `date` |

**Reset Scope Behavior:**

| Value   | Description                                      |
|---------|--------------------------------------------------|
| `none`  | Never reset; continuously increment              |
| `year`  | Reset to initial value at the start of each year |
| `month` | Reset to initial value at the start of each month|
| `date`  | Reset to initial value at the start of each day  |

#### 3.3 ID Generation API

##### Request Flow

```
┌─────────────┐      GET /v1/id/formatted                   ┌─────────────┐
│             │      Authorization: Bearer <key_token>      │             │
│ Client/SDK  │      key=invoice-id&size=2                  │   Worker    │
│             │  ─────────────────────────────────────────▶ │             │
│             │                                             │             │
│             │  ◀─────────────────────────────────────────  │             │
└─────────────┘  {"id": ["INV20230111-0001","INV20230111-0002"]} └─────────────┘
```

##### API Endpoint

```
GET /v1/id/formatted
```

##### Request Parameters

| Parameter | Type    | Required | Default | Description                                |
|-----------|---------|----------|---------|--------------------------------------------|
| `key`     | string  | Yes      | -       | Identifier for the ID configuration        |
| `size`    | integer | No       | 1       | Number of IDs to generate (1-1000)         |

##### Request Example

```bash
curl -X GET "http://idbuilder/v1/id/formatted?key=invoice-id&size=2" \
  -H "Authorization: Bearer <key_token>"
```

##### Response Parameters

| Parameter | Type   | Description                                |
|-----------|--------|--------------------------------------------|
| `id`      | array  | List of generated formatted string IDs     |

##### Response Example

```json
{
  "code": 0,
  "message": "success",
  "data": {
    "id": ["INV20230111-0001", "INV20230111-0002"]
  }
}
```

---

## Error Handling

### Error Response Format

```json
{
  "code": <error_code>,
  "message": "<error_message>",
  "data": null
}
```

### Error Codes

| Code | HTTP Status | Description                                    |
|------|-------------|------------------------------------------------|
| 0    | 200         | Success                                        |
| 1001 | 400         | Invalid request parameters                     |
| 1002 | 400         | Invalid key format                             |
| 1003 | 400         | Requested size exceeds maximum limit           |
| 1004 | 400         | Requested delta exceeds maximum limit          |
| 2001 | 401         | Authentication failed: invalid token           |
| 2002 | 403         | Authorization failed: token not allowed for key|
| 3001 | 404         | Key not found                                  |
| 4001 | 500         | Internal server error                          |
| 4002 | 503         | Service temporarily unavailable                |
| 4003 | 503         | ID sequence exhausted                          |

### Error Response Example

```json
{
  "code": 2001,
  "message": "Authentication failed: invalid token",
  "data": null
}
```

---

## Security Considerations

### Authentication

IDBuilder uses a two-tier token system:

| Token Type  | Purpose                                    | Scope                              |
|-------------|--------------------------------------------|------------------------------------|
| Admin Token | Manage configurations, retrieve key tokens | Full access to all keys            |
| Key Token   | Generate IDs for a specific key            | Limited to single key, time-bound  |

**Security Guidelines:**

- Admin tokens should be stored securely and never exposed to client applications
- Key tokens are designed for client-side use and have limited scope
- Key tokens include expiration time (`expires_at`) and should be refreshed before expiry
- Use `/v1/auth/token` to retrieve or rotate key tokens
- Token rotation is recommended periodically for both token types

### Rate Limiting

- Implement rate limiting per token to prevent abuse
- The `max_request_delta` parameter prevents rapid ID exhaustion
- Maximum `size` per request should be enforced (recommended: 1000)

### Network Security

- All communications should use HTTPS in production
- Consider IP whitelisting for sensitive deployments

---

## Appendix

### Glossary

| Term        | Definition                                                       |
|-------------|------------------------------------------------------------------|
| Key         | Unique identifier for an ID generation configuration             |
| Admin Token | Authentication credential for managing configurations            |
| Key Token   | Authentication credential for generating IDs (scoped to one key) |
| Delta       | The increment value between consecutive auto-increment IDs       |
| Worker ID   | Unique identifier assigned to each worker node                   |
| Sequence    | Counter that resets periodically within snowflake IDs            |

### References

- [Twitter Snowflake](https://github.com/twitter-archive/snowflake) — Original snowflake ID algorithm
- [Leaf](https://github.com/Meituan-Dianping/Leaf) — Meituan's distributed ID generation service
