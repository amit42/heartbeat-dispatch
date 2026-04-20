# Heartbeat API

Devices periodically check in with the server. On each check-in, the server may return a pending `dispatch_execution` for the device to run. If nothing is pending, the server responds with null. The device may optionally report the result of a previous execution in the same request.

**Heartbeat endpoint:** [POST `/v1/heartbeat`](#post-v1heartbeat)

---

## Terminology

| Term | Description |
|---|---|
| `dispatch_document` | the raw blob delivered to the device, schema-less, any valid JSON value |
| `dispatch_execution` | one traceable delivery attempt, has a unique ID, each retry is a new execution |
| `dispatch_campaign` | top level construct, holds targeting query, documents, retry and fallback config |

---

## Request

<a id="post-v1heartbeat"></a>

### POST `/v1/heartbeat`

```
POST /v1/heartbeat
```

### Headers

| Header | Required | Description |
|---|---|---|
| `Content-Type` | yes | must be `application/json` |
| `X-HD-Device-Id` | yes | unique device identifier |
| `X-HD-Key-Id` | yes | opaque key identifier, server resolves auth strategy from cache |
| `X-HD-Timestamp` | yes | unix epoch in seconds |
| `X-HD-Signature` | yes | HMAC signature of canonical string |

### Auth strategy resolution

`X-HD-Key-Id` is opaque to the device. The server looks it up in cache to determine the auth strategy — derived key, legacy key, or master key. The device has no awareness of which strategy is in use. Strategy is fully managed server-side.

### Signature input

Signature is `HMAC-SHA256` computed over a canonical string:

```
device_id + key_id + timestamp + sha256(body)
```

The result is hex encoded and sent as `X-HD-Signature`.

Requests with a timestamp older than 30 seconds are rejected to prevent replay attacks.

### Body

Both fields are optional. Body can be empty `{}`.

```json
{
  "attributes": {
    "firmware": "1.2.3",
    "region": "us-east",
    "anything": "device wants to send"
  },
  "execution_report": {
    "execution_id": "dex-456",
    "status": "succeeded",
    "details": "any valid json value"
  }
}
```

#### `attributes`

Completely schema-less. Server does not validate, parse, or store attributes. They are used only at request time to evaluate which `dispatch_campaign` applies to this device, then forwarded to the message queue as-is.

#### `execution_report`

Optional. Device sends this when it has a result to report for a previous `dispatch_execution`.

| Field | Type | Description |
|---|---|---|
| `execution_id` | string | ID of the `dispatch_execution` being reported |
| `status` | string | see status behavior below |
| `details` | any valid JSON value | freeform, opaque to server |

#### Status behavior

| Status | Behavior |
|---|---|
| `succeeded` | terminates `dispatch_execution`, marks complete |
| `failed` | terminates `dispatch_execution`, marks failed — campaign retry logic kicks in |
| anything else | recorded as non-terminating, `dispatch_execution` stays active |

---

## Response

All responses share the same envelope.

### Execution pending

```json
{
  "data": {
    "execution": {
      "execution_id": "dex-789",
      "document": "any valid json value"
    }
  },
  "meta": {
    "request_id": "req-abc123",
    "timestamp": 1704067200
  }
}
```

`document` is the `dispatch_document` payload from the matching `dispatch_campaign`. Any valid JSON value — string, object, array. Server does not interpret it.

### Nothing pending

```json
{
  "data": {
    "execution": null
  },
  "meta": {
    "request_id": "req-abc123",
    "timestamp": 1704067200
  }
}
```

### Error

```json
{
  "error": {
    "code": "AUTH_FAILED",
    "message": "signature verification failed"
  },
  "meta": {
    "request_id": "req-abc123",
    "timestamp": 1704067200
  }
}
```

#### Error codes

| Code | Description |
|---|---|
| `AUTH_FAILED` | signature verification failed |
| `INVALID_TIMESTAMP` | timestamp missing or malformed |
| `REPLAY_DETECTED` | timestamp older than 30 seconds |
| `UNKNOWN_DEVICE` | device id not found |
| `UNKNOWN_KEY` | key id not found |
| `INVALID_BODY` | body is not valid json |

---

## What the server does on each heartbeat

1. resolve auth strategy from `X-HD-Key-Id` cache lookup
2. verify signature
3. process `execution_report` if present — update `dispatch_execution` record
4. evaluate pending `dispatch_campaign` targets against incoming `attributes` — in memory, campaigns from cache
5. if match found, create a new `dispatch_execution` and return it
6. forward raw payload to message queue — async, outside request path

## What the server never does

- parse or validate `attributes`
- store `attributes`
- assume any schema on `dispatch_document`
- maintain a persistent connection with the device
- interpret `dispatch_document` content
