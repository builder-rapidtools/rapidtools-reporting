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
- **Manifest**: https://reporting.rapidtools.dev/manifest.json
- **Terms of Service**: https://reporting.rapidtools.dev/terms.html

## API

**Base URL**: `https://reporting-tool-api.jamesredwards89.workers.dev`

**Authentication**: API key via `x-api-key` header

**Required CSV columns**: `date`, `sessions`, `users`
**Optional CSV columns**: `pageviews`, `page_path`, `page_views`

## Quick flow

1. Register agency → receive API key
2. Create client → receive client ID
3. Upload GA4 CSV for client
4. Send report (generates PDF, sends email)

## Endpoints

- `POST /api/agency/register` — Register new agency
- `POST /api/client` — Create client
- `GET /api/clients` — List all clients
- `POST /api/client/{id}/ga4-csv` — Upload CSV data (Content-Type: text/csv)
- `POST /api/client/{id}/report/send` — Generate and send report
- `GET /api/health` — Health check

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

All endpoints return JSON:

```json
{
  "success": true,
  "client": { ... },
  "nextSteps": {
    "uploadCsv": "/api/client/{id}/ga4-csv",
    "sendReport": "/api/client/{id}/report/send"
  }
}
```

Errors return:

```json
{
  "success": false,
  "error": "Error message"
}
```

## Data handling

- Storage: Cloudflare KV + R2
- Retention: Minimum required for reporting (CSV + generated PDFs)
- Training use: No

## Provider

RapidTools, United Kingdom
Contact: reports@rapidtools.dev

## License

See [Terms of Service](https://reporting.rapidtools.dev/terms.html)
