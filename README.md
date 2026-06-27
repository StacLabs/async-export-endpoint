# STAC API - Async Export Extension

* **Title:** Async Export Endpoint
* **Conformance Classes:**
  * `https://api.stacspec.org/v1.0.0/core` (required)
  * `https://api.stacspec.org/v1.0.0-beta.1/async-export` (required)
  * `https://api.stacspec.org/v1.0.0-rc.2/multi-tenant-catalogs` (required if implementing `/catalogs/{catalogId}/export`)
  * `https://api.stacspec.org/v1.0.0-rc.2/multi-tenant-catalogs/search` (required if implementing `/catalogs/{catalogId}/export`)


* **Scope:** STAC API - Core
* **Extension Maturity Classification:** Proposal
* **Dependencies:**
  * [STAC API - Core](https://github.com/radiantearth/stac-api-spec/blob/main/core)
  * [Multi-Tenant Catalogs Extension](https://github.com/StacLabs/multi-tenant-catalogs) (Required if implementing `/catalogs/{catalogId}/export`)
  * [Multi-Tenant Catalogs Search Conformance](https://github.com/StacLabs/multi-tenant-catalogs#scoped-search-recursive-traversal) (Required if implementing `/catalogs/{catalogId}/export`)


* **Owner:** @jonhealy1

## Introduction

**About:** STAC API Extension to support asynchronous bulk data extraction via a dedicated `/export` endpoint. (Background worker architecture).

This extension introduces a dedicated `/export` endpoint to the STAC API, enabling the asynchronous, bulk extraction of STAC Items.

As STAC catalogs grow to millions of records, paginating through massive result sets over HTTP becomes fragile, bandwidth-heavy, and highly inefficient for both the client and the server. This extension allows clients to submit a standard STAC search query and request the server to compile the matching results into a single, highly compressed file (such as GeoParquet, GeoJSON, or CSV) via a background worker.

A core tenet of this extension is **Flexibility of Delivery**. While it provides a standardized polling mechanism by default, the extension defines optional delivery capabilities (like Webhooks or direct-to-cloud-bucket pushes) to seamlessly integrate STAC APIs into automated enterprise data pipelines.

## Endpoints

### Core Endpoints (Required)

Implementations MUST support the following core endpoints to conform to this extension. These endpoints operate globally. To export specific subsets of data, clients MUST pass the appropriate parameters (like `collections` or `bbox`) in the JSON request body.

| Method | URI | Description |
| --- | --- | --- |
| `GET` | `/export/capabilities` | Discovery. Lists the supported output formats and delivery methods for this API. |
| `POST` | `/export` | Initiate Export. Submits a query and begins the asynchronous background task. |
| `GET` | `/export/{exportId}` | Status Check. Retrieves the current status of the task and the download link if completed. |

### Catalog-Scoped Endpoint (Optional)

To provide a scoped export workflow, implementations that support Multi-Tenant Catalogs MAY append `/export` to the catalog route.

This allows clients to initiate an export that is inherently constrained by the catalog path, reducing the need for complex filter definitions in the request body.

| Method | URI | Description |
| --- | --- | --- |
| `POST` | `/catalogs/{catalogId}/export` | **Catalog-Scoped Export.** (For use with the *Multi-Tenant Catalogs Extension*). The worker MUST restrict output to items belonging to collections within the `{catalogId}` tree. |

**Note on Status Checks for Appended Routes:** If an API implements the optional search-appended initiation routes, it MUST return a `status_url` in the `202 Accepted` response that points to the core status endpoint (e.g., `https://api.example.com/export/{jobID}`). It is NOT required to mirror the status check endpoints across all search paths.

## STAC-Native Discovery via Hypermedia Links

STAC heavily relies on hypermedia (links) for discoverability. Rather than requiring clients to hardcode the `/export` path, implementations MUST advertise export capabilities through standardized link relations in the API's root catalog (landing page).

### Root Catalog Links

The root catalog (`GET /`) MUST include the following link relations to enable discovery:

```json
{
  "links": [
    {
      "rel": "export-capabilities",
      "href": "https://api.example.com/export/capabilities",
      "type": "application/json",
      "title": "Export Capabilities"
    },
    {
      "rel": "export",
      "href": "https://api.example.com/export",
      "type": "application/json",
      "title": "Initiate Export"
    }
  ]
}
```

### Link Relation Definitions

- **`rel="export-capabilities"`** (REQUIRED): Points to the `/export/capabilities` endpoint. Clients use this to discover supported formats and delivery methods before initiating an export.
- **`rel="export"`** (REQUIRED): Points to the `/export` endpoint for initiating export jobs. Clients POST search queries and export configurations to this URL.

### Catalog-Scoped Export Links

If implementing the optional `/catalogs/{catalogId}/export` endpoint, each catalog SHOULD include a link with `rel="export"` pointing to its scoped export endpoint:

```json
{
  "id": "sentinel-2-l2a",
  "links": [
    {
      "rel": "export",
      "href": "https://api.example.com/catalogs/sentinel-2-l2a/export",
      "type": "application/json",
      "title": "Export from this Catalog"
    }
  ]
}
```

### Discovery Workflow

1. **Client discovers export capability:** Client fetches the root catalog and searches for `rel="export-capabilities"` link
2. **Client queries capabilities:** Client follows the link to retrieve supported formats and delivery methods
3. **Client initiates export:** Client follows the `rel="export"` link to POST an export request
4. **Client monitors progress:** Client uses the `status_url` returned in the `202 Accepted` response to poll for job status

This approach ensures:
- **Zero hardcoding:** Clients don't need to know the `/export` path structure
- **Version flexibility:** APIs can reorganize paths without breaking clients
- **Crawlability:** Automated tools and crawlers can discover export features
- **STAC conformance:** Aligns with STAC's hypermedia-first design philosophy

## Delivery Behavior & Conformance

To support a wide range of infrastructure—from lightweight local deployments to enterprise cloud clusters—implementations are NOT required to support all delivery methods or file formats.

### 1. Capabilities Discovery (`GET /export/capabilities`)

Clients SHOULD query the capabilities endpoint prior to initiating an export to determine what the server supports. If a client requests an unsupported format or delivery method, the server MUST return a `400 Bad Request` or `501 Not Implemented`.

### 2. Initiating an Export (`POST /export`)

The request body accepts all standard STAC Item Search parameters (e.g., `bbox`, `datetime`, `collections`, `filter`). Additionally, it MUST include an `export` object defining the output configuration.

* **`format` (Required):** The desired output file type (e.g., `geoparquet`, `geojson`).
* **`delivery.method` (Optional):** How the client wishes to receive the data. Defaults to `polling` if omitted.
* `polling`: The server hosts the file; the client must query `GET /export/{exportId}` to retrieve the download URL.
* `webhook`: The server sends an HTTP POST to the provided `webhook_url` when the file is ready.
* `s3` / `gcs` / `azure`: The server pushes the file directly to the client's provided storage bucket.



Because the task is asynchronous, the server MUST respond immediately with a `202 Accepted` status code, containing the task `export_id` and a `status_url`.

If a query matches 0 items, the server MAY either return a `400 Bad Request` immediately or complete the task and produce an empty export file, depending on the requested format and delivery method.

### 3. Polling and Auto-Downloading (`GET /export/{exportId}`)

When the delivery method is `polling`, clients query this endpoint to check the task status (`pending`, `processing`, `completed`, `failed`).

Once the status is `completed`, the response MUST include a `download_url`.

**The `?redirect=true` Parameter:**
To facilitate seamless browser downloads, servers SHOULD support a `redirect=true` query parameter. If this parameter is present and the task is `successful`, the server MUST respond with a `303 See Other` HTTP status, redirecting the client directly to the `download_url`. This allows UI clients to trigger native "Save As..." browser dialogs.

## Polling Etiquette & Rate Control

When clients poll the status endpoint, they must do so responsibly to avoid overwhelming the server. This section defines polling best practices and server-side rate control mechanisms.

### Retry-After Header (REQUIRED)

Servers MUST include the standard `Retry-After` HTTP header in the following responses:

1. **`202 Accepted` response** (when job is queued)
2. **`200 OK` response** (when job status is `accepted` or `running`)

The `Retry-After` header tells clients the minimum number of seconds to wait before polling again.

**Header Format:**
```
Retry-After: <seconds>
```

**Example:**
```
HTTP/1.1 202 Accepted
Retry-After: 5
Content-Type: application/json

{
  "jobID": "exp-a1b2c3d4",
  "status": "accepted",
  "message": "Export task has been queued."
}
```

### Recommended Retry-After Values

- **Initial submission (202 Accepted):** `Retry-After: 5` (5 seconds minimum)
- **Early processing (status: accepted):** `Retry-After: 5-10` (5-10 seconds)
- **Active processing (status: running):** `Retry-After: 10-30` (10-30 seconds, can increase with progress)
- **Completed (status: successful/failed):** No `Retry-After` needed (client should not poll further)

Servers MAY increase `Retry-After` values dynamically based on:
- Current server load
- Estimated time to completion
- Progress percentage (e.g., increase wait time as progress approaches 100%)

### Client Polling Behavior

Clients MUST respect the `Retry-After` header:

1. **Extract the header:** Read the `Retry-After` value from the response
2. **Wait the specified duration:** Do not poll before the specified number of seconds have elapsed
3. **Implement exponential backoff:** If the server is unavailable (5xx errors), apply exponential backoff with jitter
4. **Set a maximum poll interval:** Do not poll more frequently than the server's `Retry-After` suggests

**Example (Python):**
```python
import time
import requests

def poll_export_status(status_url, max_attempts=100):
    for attempt in range(max_attempts):
        response = requests.get(status_url)
        
        # Extract Retry-After header
        retry_after = int(response.headers.get("Retry-After", 5))
        
        if response.status_code == 200:
            data = response.json()
            if data["status"] in ["successful", "failed"]:
                return data  # Job complete, stop polling
            
            # Job still running, wait before next poll
            print(f"Job still running. Waiting {retry_after} seconds...")
            time.sleep(retry_after)
        else:
            # Server error, apply exponential backoff
            wait_time = min(retry_after * (2 ** attempt), 300)  # Cap at 5 minutes
            print(f"Server error. Waiting {wait_time} seconds...")
            time.sleep(wait_time)
    
    raise TimeoutError("Export job did not complete within timeout")
```

### Server Recommendations

1. **Always include Retry-After:** Never omit this header from polling responses
2. **Adaptive timing:** Adjust `Retry-After` based on job progress and server load
3. **Exponential increase:** Consider increasing `Retry-After` as the job progresses (e.g., 5s → 10s → 20s)
4. **Document defaults:** Clearly document the default `Retry-After` value in API documentation
5. **Rate limiting:** Optionally implement rate limiting (429 Too Many Requests) for clients that ignore `Retry-After`

### Rate Limiting (Optional)

Servers MAY implement rate limiting to protect against aggressive polling:

- Return `429 Too Many Requests` if a client polls more frequently than the `Retry-After` interval
- Include `Retry-After` header in the 429 response indicating when the client can retry
- Log rate limit violations for monitoring and debugging

**Example 429 Response:**
```
HTTP/1.1 429 Too Many Requests
Retry-After: 30
Content-Type: application/json

{
  "code": "TooManyRequests",
  "description": "You are polling too frequently. Please wait 30 seconds before the next request."
}
```

## Error Handling and Limit Enforcement

### Item Count Limits

The capabilities endpoint advertises `max_items_limit`, which defines the maximum number of items a single export can contain.

#### Behavior When Query Exceeds Limit

When a query matches more items than `max_items_limit`:

**Option A: Immediate Rejection (Recommended)**
- The server returns `400 Bad Request` immediately upon receiving the request
- Response includes an error message indicating the limit was exceeded and the estimated item count
- Client must refine the query (e.g., smaller bbox, narrower datetime range) and retry

```json
{
  "code": "ItemLimitExceeded",
  "description": "Query matches 600,000 items, but max_items_limit is 500,000. Please refine your search criteria."
}
```

**Option B: Truncation with Warning**
- The server accepts the request and returns `202 Accepted`
- Processing proceeds, but only the first `max_items_limit` items are exported
- The completed status response includes a `truncated` flag and warning message
- Client can determine if the partial export is acceptable

```json
{
  "jobID": "exp-a1b2c3d4",
  "status": "successful",
  "item_count": 500000,
  "estimated_items": 600000,
  "truncated": true,
  "message": "Export truncated to max_items_limit of 500,000. Query matched 600,000 items."
}
```

**Recommendation:** Implementations SHOULD choose Option A (immediate rejection) for predictability and to prevent client confusion. Option B is acceptable for implementations that prefer to provide partial results.

### Empty Result Sets

If a query matches 0 items:

- **Option 1:** Return `400 Bad Request` immediately with a descriptive error message
- **Option 2:** Accept the request and complete the task with an empty export file

The choice depends on the requested format and delivery method. For example:
- GeoParquet with S3 delivery might reject empty results (invalid Parquet)
- GeoJSON with polling delivery might accept empty results (valid empty FeatureCollection)

Servers MUST document their behavior for empty results in the API documentation.

### Failed Jobs

When a job's status is `failed`, the response MUST include one of the following:

1. **`error` field (string):** A human-readable error message explaining the failure
2. **`error` object:** A structured error with `code` and `description` fields

```json
{
  "jobID": "exp-a1b2c3d4",
  "status": "failed",
  "error": "Insufficient disk space to generate export file. Please try again later."
}
```

Or:

```json
{
  "jobID": "exp-a1b2c3d4",
  "status": "failed",
  "error": {
    "code": "InsufficientStorage",
    "description": "Insufficient disk space to generate export file. Please try again later."
  }
}
```

**Common Failure Reasons:**
- `ItemLimitExceeded`: Query matched more items than `max_items_limit`
- `UnsupportedFormat`: Requested format is not supported
- `UnsupportedDeliveryMethod`: Requested delivery method is not supported
- `InvalidCredentials`: Cloud credentials (S3, GCS, Azure) are invalid or expired
- `DestinationUnreachable`: Cannot write to the provided `destination_uri`
- `ProcessingTimeout`: Export processing exceeded the server's timeout limit
- `InsufficientStorage`: Server ran out of disk space during processing
- `InvalidQuery`: Search parameters are malformed or invalid

### HTTP Status Codes

| Status | Scenario | Response Body |
| --- | --- | --- |
| `202 Accepted` | Job successfully queued | ExportAccepted schema |
| `400 Bad Request` | Invalid request, unsupported format, limit exceeded, or empty results rejected | Error object with `code` and `description` |
| `404 Not Found` | Job ID does not exist | Error object |
| `501 Not Implemented` | Requested format or delivery method not supported | Error object |
| `503 Service Unavailable` | Server temporarily unable to process exports | Error object |

---

## Response Examples

### 1. The Capabilities Endpoint (`GET /export/capabilities`)

```json
{
  "supported_formats": [
    "geoparquet",
    "geojson",
    "csv"
  ],
  "supported_delivery_methods": [
    "polling",
    "webhook",
    "s3"
  ],
  "max_items_limit": 500000,
  "storage_retention_days": 7
}

```

### 2. Initiating an Export (`POST /export`)

**Request Body (Polling Delivery):**

```json
{
  "collections": ["sentinel-2-l2a"],
  "bbox": [-122.3, 37.7, -122.1, 37.9],
  "datetime": "2025-01-01/2025-12-31",
  "export": {
    "format": "geoparquet",
    "delivery": {
      "method": "polling"
    }
  }
}

```

**Response (`202 Accepted`):**

```json
{
  "jobID": "exp-a1b2c3d4",
  "type": "process",
  "status": "accepted",
  "message": "Export task has been queued.",
  "created": "2025-06-27T10:30:00Z",
  "progress": 0
}

```

### 3. Checking Task Status (`GET /export/{jobID}`)

**Response (`200 OK` - Running):**

```json
{
  "jobID": "exp-a1b2c3d4",
  "type": "process",
  "status": "running",
  "created": "2025-06-27T10:30:00Z",
  "started": "2025-06-27T10:31:00Z",
  "updated": "2025-06-27T10:35:00Z",
  "estimated_items": 45200,
  "progress": 45
}

```

**Response (`200 OK` - Successful):**

```json
{
  "jobID": "exp-a1b2c3d4",
  "type": "process",
  "status": "successful",
  "created": "2025-06-27T10:30:00Z",
  "started": "2025-06-27T10:31:00Z",
  "finished": "2025-06-27T10:45:00Z",
  "updated": "2025-06-27T10:45:00Z",
  "download_url": "https://data.example.com/downloads/exp-a1b2c3d4.parquet",
  "expires_at": "2025-07-04T00:00:00Z",
  "item_count": 45200,
  "file_size_bytes": 104857600,
  "progress": 100
}

```

**Response (`200 OK` - Failed):**

```json
{
  "jobID": "exp-a1b2c3d4",
  "type": "process",
  "status": "failed",
  "created": "2025-06-27T10:30:00Z",
  "started": "2025-06-27T10:31:00Z",
  "finished": "2025-06-27T10:32:00Z",
  "updated": "2025-06-27T10:32:00Z",
  "error": {
    "code": "InvalidCredentials",
    "description": "Failed to authenticate to S3 bucket. Please verify your credentials and try again."
  }
}

```

### 4. Direct-to-S3 Delivery (Cloud Push)

If the user requests `delivery.method: "s3"`, the server bypasses local storage and streams the generated file directly into the client's private cloud bucket. The user MUST provide a `destination_uri` where the server has been granted write permissions.

**Request (`POST /export`):**

```json
{
  "collections": ["sentinel-2-l2a"],
  "bbox": [-122.3, 37.7, -122.1, 37.9],
  "export": {
    "format": "geoparquet",
    "delivery": {
      "method": "s3",
      "destination_uri": "s3://my-organization-bucket/stac-exports/"
    }
  }
}

```

**Response (`200 OK` - Successful):**
Because the file is pushed directly to the user's bucket, the successful status response does not need to provide a temporary `download_url`. Instead, it confirms the exact final `destination_uri` where the file was written.

```json
{
  "jobID": "exp-a1b2c3d4",
  "type": "process",
  "status": "successful",
  "created": "2025-06-27T10:30:00Z",
  "started": "2025-06-27T10:31:00Z",
  "finished": "2025-06-27T10:45:00Z",
  "destination_uri": "s3://my-organization-bucket/stac-exports/exp-a1b2c3d4.parquet",
  "item_count": 45200,
  "file_size_bytes": 104857600,
  "progress": 100
}

```

### 5. Webhook Delivery Payload (Server-to-Client POST)

If the user requested `delivery.method: "webhook"`, the server will dispatch the following payload to the user's `webhook_url` once the file generation is complete.

**Payload:**

```json
{
  "jobID": "exp-a1b2c3d4",
  "type": "process",
  "status": "successful",
  "finished": "2025-06-27T10:45:00Z",
  "download_url": "https://data.example.com/downloads/exp-a1b2c3d4.parquet",
  "item_count": 45200,
  "file_size_bytes": 104857600
}

```

### Webhook Verification & Security

When using webhook delivery, clients need to verify that the payload genuinely came from the STAC server and not a malicious third party. This section defines the webhook verification mechanism.

#### Optional Webhook Secret

Clients MAY provide an optional `webhook_secret` in the delivery configuration when initiating an export:

```json
{
  "collections": ["sentinel-2-l2a"],
  "bbox": [-122.3, 37.7, -122.1, 37.9],
  "export": {
    "format": "geoparquet",
    "delivery": {
      "method": "webhook",
      "webhook_url": "https://client.example.com/webhooks/export-complete",
      "webhook_secret": "your-secret-key-here"
    }
  }
}
```

#### HMAC SHA-256 Signature

If a `webhook_secret` is provided, the server MUST sign the webhook payload using HMAC SHA-256 and include the signature in the `X-Hub-Signature-256` header:

**Header Format:**
```
X-Hub-Signature-256: sha256=<hex-encoded-hmac-sha256-signature>
```

**Signature Computation:**
1. Serialize the webhook payload to JSON (canonical form, no extra whitespace)
2. Compute HMAC-SHA256 of the JSON payload using the `webhook_secret` as the key
3. Encode the result as hexadecimal
4. Include in the `X-Hub-Signature-256` header

**Example Header:**
```
X-Hub-Signature-256: sha256=abcd1234ef5678...
```

#### Client Verification

Clients receiving a webhook MUST:

1. Extract the `X-Hub-Signature-256` header value
2. Compute the expected HMAC-SHA256 signature using the stored `webhook_secret`
3. Compare the computed signature with the header value using constant-time comparison
4. Only process the payload if signatures match

**Example (Python):**
```python
import hmac
import hashlib
import json

webhook_secret = "your-secret-key-here"
received_signature = request.headers.get("X-Hub-Signature-256", "").replace("sha256=", "")
payload_body = request.get_data()

expected_signature = hmac.new(
    webhook_secret.encode(),
    payload_body,
    hashlib.sha256
).hexdigest()

if not hmac.compare_digest(received_signature, expected_signature):
    return 401  # Unauthorized
```

#### Security Recommendations

1. **Use HTTPS Only:** Webhook URLs MUST use HTTPS to prevent man-in-the-middle attacks
2. **Rotate Secrets:** Periodically rotate `webhook_secret` values
3. **Constant-Time Comparison:** Always use constant-time comparison (e.g., `hmac.compare_digest()`) to prevent timing attacks
4. **Log Verification Failures:** Log failed signature verifications for security auditing
5. **Idempotency:** Implement idempotent webhook handlers to safely handle retries

#### Webhook Retry Policy

If the server receives a non-2xx response from the webhook URL, it SHOULD retry with exponential backoff:
- Initial retry: 1 minute
- Second retry: 5 minutes
- Third retry: 30 minutes
- Maximum retries: 3 attempts

Servers MUST NOT retry indefinitely to prevent overwhelming client systems.

### 6. Truncated Export (Exceeds max_items_limit)

If the server implements Option B (truncation with warning) and a query matches more items than `max_items_limit`:

**Response (`200 OK` - Successful but Truncated):**

```json
{
  "jobID": "exp-a1b2c3d4",
  "type": "process",
  "status": "successful",
  "created": "2025-06-27T10:30:00Z",
  "started": "2025-06-27T10:31:00Z",
  "finished": "2025-06-27T10:50:00Z",
  "updated": "2025-06-27T10:50:00Z",
  "download_url": "https://data.example.com/downloads/exp-a1b2c3d4.parquet",
  "expires_at": "2025-07-04T00:00:00Z",
  "item_count": 500000,
  "estimated_items": 600000,
  "file_size_bytes": 250000000,
  "truncated": true,
  "message": "Export truncated to max_items_limit of 500,000. Query matched 600,000 items.",
  "progress": 100
}

```

---

## OGC API - Processes Alignment

This extension aligns with **OGC API - Processes - Part 1: Core** to ensure interoperability with existing OGC tooling and standards. Key alignments include:

### Status Payload Standardization

The `/export/{jobID}` response schema follows the OGC StatusInfo specification:

- **`jobID`** (required): Unique job identifier (replaces `export_id`)
- **`type`** (required): Always `"process"` for OGC conformance
- **`status`** (required): Uses OGC-standard enum values:
  - `accepted`: Job has been accepted and queued
  - `running`: Job is actively processing
  - `successful`: Job completed successfully
  - `failed`: Job failed
  - `dismissed`: Job was dismissed/cancelled
- **Lifecycle Timestamps**: 
  - `created`: When the job was created
  - `started`: When processing began
  - `finished`: When processing completed
  - `updated`: Last status update timestamp
- **`progress`** (0-100): Standardized progress percentage

### STAC-Specific Extensions

While maintaining OGC conformance, the extension preserves STAC-specific fields:
- `download_url`: For polling delivery method
- `destination_uri`: For cloud push delivery (s3, gcs, azure)
- `expires_at`: Expiration of temporary storage
- `estimated_items` / `item_count`: STAC-specific metadata
- `file_size_bytes`: Export file size

This dual alignment enables:
- **OGC Interoperability**: Clients built for OGC API - Processes can consume this API
- **STAC Specificity**: STAC-specific metadata and delivery options remain available
- **Future Extensibility**: Additional OGC conformance classes can be adopted as needed

---

## Security Considerations

### Cloud Push Credentials & SSRF Risk Mitigation

When implementing cloud push delivery methods (`s3`, `gcs`, `azure`), servers must carefully handle authentication to prevent Server-Side Request Forgery (SSRF) attacks and unauthorized data access.

#### The Risk

If a server relies on its own IAM role or credentials to write to cloud buckets, a malicious user could:
- Provide a `destination_uri` pointing to another organization's bucket
- Overwrite sensitive data in buckets the server has access to
- Exfiltrate data by redirecting exports to attacker-controlled locations

#### Recommended Approaches

**Option 1: Pre-Signed URLs (Recommended)**

Clients provide Pre-Signed URLs (AWS S3) or Signed URLs (GCS) as the `destination_uri`. The server has no need for native cloud IAM permissions—it simply writes to the URL provided.

```json
{
  "export": {
    "format": "geoparquet",
    "delivery": {
      "method": "s3",
      "destination_uri": "https://my-organization-bucket.s3.amazonaws.com/stac-exports/exp-abc123.parquet?X-Amz-Algorithm=AWS4-HMAC-SHA256&X-Amz-Credential=..."
    }
  }
}
```

**Advantages:**
- Server requires no cloud IAM permissions
- Clients maintain full control over access scope and expiration
- Works across multiple cloud providers with their respective signing mechanisms

**Option 2: Optional Credentials Object**

Implementations MAY define an optional `credentials` object in the `DeliveryConfig` to allow clients to provide temporary credentials scoped to a specific bucket or prefix.

```json
{
  "export": {
    "format": "geoparquet",
    "delivery": {
      "method": "s3",
      "destination_uri": "s3://my-organization-bucket/stac-exports/",
      "credentials": {
        "access_key_id": "AKIA...",
        "secret_access_key": "...",
        "session_token": "..."
      }
    }
  }
}
```

**Advantages:**
- Flexible for clients who cannot generate pre-signed URLs
- Credentials can be scoped and time-limited

**Disadvantages:**
- Requires secure transmission (HTTPS only)
- Credentials are exposed in request body; consider encryption at rest
- More complex credential lifecycle management

#### Multi-File Exports & Pre-Signed URL Limitations

**Important Consideration:** Pre-Signed URLs have a critical limitation for large exports:

A single pre-signed URL is strictly bound to one specific object path (e.g., `exp-abc123.parquet`). If an export is very large (e.g., 100+ GB) and the server needs to chunk it into multiple smaller files (e.g., `part-001.parquet`, `part-002.parquet`, `part-003.parquet`), a single pre-signed URL **cannot be used** to write multiple files.

**Recommendation for Multi-File Exports:**

If your implementation anticipates chunking large exports into multiple files, **use the credentials object approach (Option 2)** rather than pre-signed URLs. This allows the server to write multiple files to the same bucket prefix without requiring a separate pre-signed URL for each chunk.

**Example: Multi-File Export with Credentials**

```json
{
  "export": {
    "format": "geoparquet",
    "delivery": {
      "method": "s3",
      "destination_uri": "s3://my-organization-bucket/stac-exports/exp-abc123/",
      "credentials": {
        "access_key_id": "AKIA...",
        "secret_access_key": "...",
        "session_token": "..."
      }
    }
  }
}
```

The server can then write:
- `s3://my-organization-bucket/stac-exports/exp-abc123/part-001.parquet`
- `s3://my-organization-bucket/stac-exports/exp-abc123/part-002.parquet`
- `s3://my-organization-bucket/stac-exports/exp-abc123/part-003.parquet`

**Response indicating multi-file export:**

```json
{
  "jobID": "exp-a1b2c3d4",
  "status": "successful",
  "destination_uri": "s3://my-organization-bucket/stac-exports/exp-abc123/",
  "item_count": 5000000,
  "file_size_bytes": 150000000000,
  "file_count": 3,
  "files": [
    {
      "path": "part-001.parquet",
      "size_bytes": 50000000000
    },
    {
      "path": "part-002.parquet",
      "size_bytes": 50000000000
    },
    {
      "path": "part-003.parquet",
      "size_bytes": 50000000000
    }
  ]
}
```

**Decision Matrix:**

| Scenario | Recommended Approach |
| --- | --- |
| Single-file export, small (<1 GB) | Pre-Signed URL (Option 1) |
| Single-file export, large (>1 GB) | Either option works |
| Multi-file export (chunked) | Credentials object (Option 2) |
| Maximum security (no server IAM) | Pre-Signed URL (Option 1) |
| Maximum flexibility | Credentials object (Option 2) |

#### Implementation Guidelines

1. **Validate destination URIs:** Implement allowlisting or pattern matching to prevent writes to arbitrary locations.
2. **Log all cloud operations:** Record which user initiated exports and where files were written for audit trails.
3. **Enforce HTTPS:** Never accept cloud credentials or pre-signed URLs over unencrypted connections.
4. **Prefer pre-signed URLs:** When possible, encourage clients to use pre-signed URLs to minimize the server's cloud permissions footprint.
5. **Scope credentials tightly:** If using temporary credentials, ensure they are scoped to specific buckets, prefixes, and operations (write-only).
6. **Implement rate limiting:** Prevent abuse by limiting the number of concurrent exports per user or API key.
