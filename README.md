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

**Note on Status Checks for Appended Routes:** If an API implements the optional search-appended initiation routes, it MUST return a `status_url` in the `202 Accepted` response that points to the core status endpoint (e.g., `https://api.example.com/export/{exportId}`). It is NOT required to mirror the status check endpoints across all search paths.

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
To facilitate seamless browser downloads, servers SHOULD support a `redirect=true` query parameter. If this parameter is present and the task is `completed`, the server MUST respond with a `303 See Other` HTTP status, redirecting the client directly to the `download_url`. This allows UI clients to trigger native "Save As..." browser dialogs.

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
  "export_id": "exp-a1b2c3d4",
  "status": "pending",
  "status_url": "https://api.example.com/export/exp-a1b2c3d4",
  "message": "Export task has been queued."
}

```

### 3. Checking Task Status (`GET /export/{exportId}`)

**Response (`200 OK` - Processing):**

```json
{
  "export_id": "exp-a1b2c3d4",
  "status": "processing",
  "status_url": "https://api.example.com/export/exp-a1b2c3d4",
  "estimated_items": 45200,
  "progress_percentage": 45
}

```

**Response (`200 OK` - Completed):**

```json
{
  "export_id": "exp-a1b2c3d4",
  "status": "completed",
  "status_url": "https://api.example.com/export/exp-a1b2c3d4",
  "download_url": "https://data.example.com/downloads/exp-a1b2c3d4.parquet",
  "expires_at": "2026-07-02T00:00:00Z",
  "item_count": 45200,
  "file_size_bytes": 104857600
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

**Response (`200 OK` - Completed):**
Because the file is pushed directly to the user's bucket, the completed status response does not need to provide a temporary `download_url`. Instead, it confirms the exact final `destination_uri` where the file was written.

```json
{
  "export_id": "exp-a1b2c3d4",
  "status": "completed",
  "status_url": "https://api.example.com/export/exp-a1b2c3d4",
  "destination_uri": "s3://my-organization-bucket/stac-exports/exp-a1b2c3d4.parquet",
  "item_count": 45200,
  "file_size_bytes": 104857600
}

```

### 5. Webhook Delivery Payload (Server-to-Client POST)

If the user requested `delivery.method: "webhook"`, the server will dispatch the following payload to the user's `webhook_url` once the file generation is complete.

**Payload:**

```json
{
  "export_id": "exp-a1b2c3d4",
  "status": "completed",
  "download_url": "https://data.example.com/downloads/exp-a1b2c3d4.parquet",
  "item_count": 45200,
  "file_size_bytes": 104857600
}

```

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

#### Implementation Guidelines

1. **Validate destination URIs:** Implement allowlisting or pattern matching to prevent writes to arbitrary locations.
2. **Log all cloud operations:** Record which user initiated exports and where files were written for audit trails.
3. **Enforce HTTPS:** Never accept cloud credentials or pre-signed URLs over unencrypted connections.
4. **Prefer pre-signed URLs:** When possible, encourage clients to use pre-signed URLs to minimize the server's cloud permissions footprint.
5. **Scope credentials tightly:** If using temporary credentials, ensure they are scoped to specific buckets, prefixes, and operations (write-only).
6. **Implement rate limiting:** Prevent abuse by limiting the number of concurrent exports per user or API key.
