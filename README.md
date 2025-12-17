# RapidTools Reporting

Automated weekly client reporting service that generates branded PDF reports from analytics data and delivers them via email.

## What it does

- Accepts GA4 CSV analytics exports (per client)
- Generates summary PDF reports
- Delivers reports via email + secure download link
- Supports weekly automated scheduling

## What it doesn't do

- Does not connect to GA4 directly (CSV upload required)
- Does not verify business accuracy of uploaded data (format validation only)
- Does not provide real-time analytics or dashboards

## Links

- **Service landing page**: https://reporting.rapidtools.dev
- **Manifest (v1 contract)**: https://reporting.rapidtools.dev/manifest.json
- **Terms of Service**: https://reporting.rapidtools.dev/terms.html
- **Documentation**: https://github.com/builder-rapidtools/rapidtools-reporting

## Machine Contract

This service follows **RapidTools machine contract v1** (`schema_version: "1.0"`):

- **Capabilities**: Array of operation descriptors with `id`, `method`, `path`, `idempotent`, `side_effects`
- **Authentication**: Structured with `type`, `location`, `header_name`, `scope`
- **Limits**: Rate limits (60/min, burst 10) and payload limits (10MB, 100k rows) with `enforced: true`
- **Idempotency**: Supported via `Idempotency-Key` header, 86400s TTL, per-agency scope
- **Errors**: Structured format with success/error schemas, error codes, and retryable codes
- **Stability**: Beta level with 30-day advance notice for breaking changes
- **Versioning**: API v1 with 90-day deprecation notice period

## API

**Base URL**: `https://reporting-tool-api.jamesredwards89.workers.dev`

**Authentication**: API key via `x-api-key` header (per-agency scope)

**Required CSV columns**: `date`, `sessions`, `users`, `pageviews`

## Capabilities

The service exposes 6 operations (see manifest for full details):

1. **health_check** - `GET /api/health` - Service health and availability
2. **create_client** - `POST /api/client` - Register new client [storage]
3. **list_clients** - `GET /api/clients` - Retrieve all clients
4. **upload_ga4_csv** - `PUT /api/client/{id}/ga4-csv` - Upload CSV data [storage]
5. **preview_report** - `GET /api/client/{id}/report/preview` - Generate preview
6. **send_report** - `POST /api/client/{id}/report/send` - Generate and send [email, storage]

All operations are idempotent. Operations marked with `[storage]` or `[email]` indicate side effects.

## Quick flow

1. Register agency → receive API key
2. Create client → receive client ID
3. Upload GA4 CSV for client
4. Send report (generates PDF, sends email)

## Example usage

See `examples/` folder for shell scripts demonstrating each step:

```bash
# 1. Register agency
API_BASE=https://reporting-tool-api.jamesredwards89.workers.dev \
  ./examples/01-register-agency.sh

# 2. Create client
API_BASE=https://reporting-tool-api.jamesredwards89.workers.dev \
API_KEY=your-api-key \
  ./examples/02-create-client.sh

# 3. Upload CSV
API_BASE=https://reporting-tool-api.jamesredwards89.workers.dev \
API_KEY=your-api-key \
CLIENT_ID=client-id \
CSV_PATH=./data.csv \
  ./examples/03-upload-csv.sh

# 4. Send report
API_BASE=https://reporting-tool-api.jamesredwards89.workers.dev \
API_KEY=your-api-key \
CLIENT_ID=client-id \
  ./examples/04-send-report.sh
```

## Response format

All endpoints follow the v1 envelope structure:

**Success:**

```json
{
  "ok": true,
  "data": {
    "client": { ... },
    "nextSteps": {
      "uploadCsv": "/api/client/{id}/ga4-csv",
      "sendReport": "/api/client/{id}/report/send"
    }
  }
}
```

**Error:**

```json
{
  "ok": false,
  "error": {
    "code": "CLIENT_NOT_FOUND",
    "message": "Client with ID abc123 not found"
  }
}
```

**Health check example:**

```bash
curl https://reporting-tool-api.jamesredwards89.workers.dev/api/health
```

```json
{
  "ok": true,
  "data": {
    "status": "ok",
    "env": "prod",
    "timestamp": "2025-12-17T16:28:58.108Z"
  }
}
```

**Auth failure example:**

```bash
curl https://reporting-tool-api.jamesredwards89.workers.dev/api/clients
```

```json
{
  "ok": false,
  "error": {
    "code": "UNAUTHORIZED",
    "message": "Missing x-api-key header"
  }
}
```

## Error codes

**Application errors:**
- `UNAUTHORIZED` - Invalid or missing API key
- `SUBSCRIPTION_INACTIVE` - Agency subscription is not active
- `FORBIDDEN` - Insufficient permissions
- `MISSING_REQUIRED_FIELDS` - Required fields missing in request
- `INVALID_EMAIL` - Email format is invalid
- `CLIENT_NOT_FOUND` - Client ID does not exist
- `MISSING_CLIENT_ID` - Client ID parameter missing
- `INVALID_CSV` - CSV format is invalid
- `NO_DATA_UPLOADED` - No CSV data has been uploaded yet
- `DATA_NOT_FOUND` - Uploaded data not found
- `REPORT_SEND_FAILED` - Report sending failed
- `AGENCY_NOT_FOUND` - Agency not found
- `WEBHOOK_ERROR` - Webhook processing error
- `INTERNAL_ERROR` - Internal server error
- `NOT_FOUND` - Resource not found
- `IDEMPOTENCY_KEY_REUSE_MISMATCH` - Idempotency key reused with different payload

**Note**: Clients should implement their own retry logic with appropriate backoff based on error type and use case.

## Rate limits

- **Requests per minute**: 60
- **Burst allowance**: 10 requests
- **Enforcement**: Enabled
- **Scope**: Per API key (per-agency)

## Payload limits

- **Max CSV size**: 10,485,760 bytes (10MB)
- **Max CSV rows**: 100,000 rows
- **Enforcement**: Enabled

## Idempotency

**Optional** support via `idempotency-key` header:
- **Supported endpoints**: `send_report` only
- **TTL**: 86,400 seconds (24 hours)
- **Scope**: `per_agency_per_client`
- Repeat requests with same key and payload return cached result with `replayed: true`
- Same key with different payload returns `409 IDEMPOTENCY_KEY_REUSE_MISMATCH`

**Example: First call with idempotency key**

```bash
curl -X POST "https://reporting-tool-api.jamesredwards89.workers.dev/api/client/{id}/report/send" \
  -H "x-api-key: your-api-key" \
  -H "idempotency-key: unique-key-123"
```

```json
{
  "ok": true,
  "data": {
    "clientId": "...",
    "sentTo": "client@example.com",
    "pdfKey": "...",
    "sentAt": "2025-12-17T17:00:00.000Z"
  }
}
```

**Example: Replay with same key (no duplicate email)**

```bash
curl -X POST "https://reporting-tool-api.jamesredwards89.workers.dev/api/client/{id}/report/send" \
  -H "x-api-key: your-api-key" \
  -H "idempotency-key: unique-key-123"
```

```json
{
  "ok": true,
  "data": {
    "clientId": "...",
    "sentTo": "client@example.com",
    "pdfKey": "...",
    "sentAt": "2025-12-17T17:00:00.000Z",
    "replayed": true
  }
}
```

**Example: Same key with different payload (conflict)**

```bash
curl -X POST "https://reporting-tool-api.jamesredwards89.workers.dev/api/client/different-id/report/send" \
  -H "x-api-key: your-api-key" \
  -H "idempotency-key: unique-key-123"
```

```json
{
  "ok": false,
  "error": {
    "code": "IDEMPOTENCY_KEY_REUSE_MISMATCH",
    "message": "Idempotency key was already used with a different request payload"
  }
}
```

## Data handling

- **Storage**: Cloudflare KV + R2
- **Retention**: CSV data and generated PDFs retained for active subscriptions
- **Training use**: No

## Provider

RapidTools, United Kingdom
Contact: reports@rapidtools.dev

## License

See [Terms of Service](https://reporting.rapidtools.dev/terms.html)
