---
name: pixiebrix-manage-database-records
description: Read, write and clean up records in a PixieBrix team database, respecting its plan quota, its schema, and the pagination exception on the records endpoint.
api: PixieBrix Developer API
base_url: https://app.pixiebrix.com/api/
generated: '2026-08-26'
method: generated
source: openapi/pixiebrix-openapi.yml
operations:
  - listDatabases
  - retrieveDatabase
  - retrieveDatabaseSchema
  - updateDatabaseSchema
  - listRecords
  - retrieveRecordDetail
  - updateRecordDetail
  - createRecord
  - destroyRecordDetail
  - clearRecord
  - listDatabaseRecordsArchives
  - createDatabaseExportJob
  - retrieveDatabaseExportJob
---

# Work with PixieBrix database records

PixieBrix databases are the hosted key-value store mods read and write. Record
volume is **plan-gated**: Free 1 database / 2,000 records, Team 5 / 10,000,
Business 10 / 100,000, Enterprise custom.

## Steps

1. **Find the database.** `listDatabases` —
   `GET /api/organizations/{organization_pk}/databases/`, then `retrieveDatabase`
   for detail.

2. **Read the schema before writing.** `retrieveDatabaseSchema` —
   `GET /api/organizations/{organization_pk}/databases/{database_pk}/schema/`.
   Writes that do not match are rejected as 400 with field-keyed messages.

3. **Read records.** `listRecords` — `GET /api/databases/{database_pk}/records/`.
   **Pagination exception, and it will bite you:** this endpoint does *not* return
   `X-Total-Count` and does *not* emit a `Link rel="last"`, because both need a full
   count that performs poorly on large databases. Iterate by following
   `rel="next"` until it is absent. Never compute a page count up front here.
   Single record: `retrieveRecordDetail` —
   `GET /api/databases/{database_pk}/records/{key}/`.

4. **Write a record.** Prefer `updateRecordDetail` —
   `PUT /api/databases/{database_pk}/records/{key}/`. **PUT to an explicit key is
   idempotent**, which matters because the API publishes no idempotency-key
   mechanism; `createRecord` (`POST /api/databases/{database_pk}/records/`) is not
   replay-safe. Use the keyed PUT for anything an agent might retry.

5. **Delete carefully.** `destroyRecordDetail` —
   `DELETE /api/databases/{database_pk}/records/{key}/` removes one record.
   `clearRecord` — `DELETE /api/databases/{database_pk}/records/` clears the
   collection. Neither has a published recovery window.

6. **Recover, if you can.** `listDatabaseRecordsArchives` —
   `GET /api/organizations/{organization_pk}/databases/{database_pk}/record-archives/`
   lists archived snapshots newest-first, each with a downloadable file of the
   contents at archive time. There is **no restore-from-archive operation** — the
   snapshot is a download you re-import by hand, and no retention period is
   published. Treat every delete as permanent.

7. **Export in bulk.** `createDatabaseExportJob` (`POST /api/databases/records/jobs/`)
   then poll `retrieveDatabaseExportJob` (`GET /api/databases/records/jobs/{id}/`).
   Export endpoints can return CSV — send `Accept: "text/csv; version=2.0"`.

## Rules that apply here

- Quota exhaustion is a plan limit, not a rate limit — raising it means changing plan.
- 429 means the per-token minute throttle; back off, no `Retry-After` is sent.
- No CORS: call this from a backend, not from browser JavaScript.
