# Changelog

All notable changes to the "Async Export Endpoint" STAC API Extension will be documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.0.0/),
and this project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

## [Unreleased]

## [v1.0.0-beta.1] - 2026-06-25

### Added
- Core async export flow with dedicated endpoints for capabilities discovery, export job creation, and job status polling.
- Optional search-appended export initiation routes for global search and catalog-scoped search contexts.
- Delivery method model for polling, webhook callbacks, and direct cloud destinations.
- Polling UX enhancement guidance for redirect-based downloads via redirect=true.
- Response examples for capabilities, queued/processing/completed states, webhook payloads, and direct-to-bucket delivery.
- Initial OpenAPI description file for multi-tenant catalogs extension interoperability.

### Changed
- Conformance metadata updated to async export beta.1 URI.
- Metadata clarifications for conditional conformance and dependency requirements when implementing search-appended and catalog-scoped export routes.
- Documentation guidance for zero-result exports, allowing either immediate bad request handling or completed empty-file behavior.
