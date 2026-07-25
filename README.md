# WSO2 Micro Integrator templates for Boomi DataHub (MDH)

[![License: Apache 2.0](https://img.shields.io/badge/License-Apache_2.0-blue.svg)](LICENSE)

## What this is for
 
Boomi Data Hub holds the trusted version of the records that several systems all think they own customers, products, suppliers, sites. Salesforce has "Bob Smith", the webshop
has "robert smith", the support desk has "B. Smith". Data Hub decides those are one person, keeps one golden record, and tracks which system contributed what.

That creates two-way traffic. Source systems contribute entities inward; Data Hub propagates the resulting changes back outward to every system that subscribes to them. Something has to sit in the middle and move that traffic. These templates are for building that something on WSO2 Micro Integrator.

### If you already run WSO2, you don't need Boomi Integration as well
 
The common assumption when Data Hub is bought is that Boomi Integration must come with it, 
because Data Hub is a Boomi product. Technically that isn't so. The Repository API is a
complete data-movement surface: contributing batches, fetching channel updates, querying
golden records, pulling quarantine and staging for review are all plain HTTP calls. The
Boomi Data Hub connector is a convenience wrapper over that API, not a privileged path
into it. An integration engine you already operate can do the same work.

> Two honest limits. Whether Data Hub can be licensed without Integration is a commercial question for Boomi and varies by agreement — the claim here is technical, not contractual.

Eleven reusable Synapse **sequence templates** wrapping the Boomi DataHub Repository API,
plus a sample REST API and shared config/fault sequences that show how to wire them up.

Each template encapsulates the one thing that's tedious and error-prone to repeat: the exact
endpoint URL, HTTP method, content type, auth header, and the optional-query-param handling.
Everything else (building request bodies, transforming responses, error policy) stays with the
caller, so the templates compose cleanly into your own proxies/APIs/sequences.

## Operations → endpoints

All paths are relative to the Hub base, e.g. `https://c01-usa-east.hub.boomi.com/mdm`.

| Template | Method | Path | Request body |
|---|---|---|---|
| `mdh-get-golden-record` | GET | `/universes/{u}/records/{recordId}` | — |
| `mdh-get-quarantine-entry` | GET | `/universes/{u}/quarantine/sources/{sourceId}/entities/{entityId}` | — |
| `mdh-update-golden-records` | POST | `/universes/{u}/records` | `<batch src="...">` |
| `mdh-stage-golden-records` | POST | `/universes/{u}/staging/{stagingCode}` | `<batch src="...">` |
| `mdh-get-batch-update-status` | GET | `/universes/{u}/records/updates/{batchId}` | — |
| `mdh-query-golden-records` | POST | `/universes/{u}/records/query` | `<RecordQueryRequest>` |
| `mdh-query-quarantine-entries` | POST | `/universes/{u}/quarantine/query` | `<QuarantineQueryRequest>` |
| `mdh-fetch-channel-updates` | POST | `/universes/{u}/sources/{sourceId}/updates` then `/updates/{updateId}` | — (empty) |
| `mdh-fetch-channel-updates-ack` | POST | fetch + acknowledge one batch, return it | — (empty) |
| `mdh-acknowledge-channel-updates` | POST | `/universes/{u}/sources/{sourceId}/updates/{updateId}/acknowledge` | — (empty) |
| `mdh-match-entities` | POST | `/universes/{u}/match` | `<batch src="...">` |

Two names don't map where you'd expect in Boomi's own API grouping, so a heads-up:

- **Update Golden Records** is a *Batch* operation (`POST .../records`), not a "Records" one. The
  Boomi *Staging* resource is a different thing — it manages the staged-entity lifecycle
  (query/commit/delete staged entities), not the contribute-a-batch operation.
- **Get Quarantine Entry** is fetched *by source + source entity ID*. There is no
  "get by transactionId" GET; to look entries up by cause/date/transaction, use
  `mdh-query-quarantine-entries`.
- **Consuming channel updates** — there are two acknowledge strategies:
  - *Auto (fetch + acknowledge in one call).* `mdh-fetch-channel-updates-ack` fetches the next
    batch, backs up the payload, acknowledges that batch (so Boomi won't redeliver it), and
    returns the original batch to the caller. Simplest for consumers; at-most-once (the batch is
    acknowledged before you get it).
  - *Manual (acknowledge after processing).* Use `mdh-fetch-channel-updates` to fetch, process the
    batch downstream, then `mdh-acknowledge-channel-updates` to acknowledge it by id — at-least-once,
    since nothing is acknowledged until you've safely handled it.
  `mdh-fetch-channel-updates-ack` reuses both `mdh-fetch-channel-updates` and
  `mdh-acknowledge-channel-updates`, so all three must be deployed together.

## Files

These are standalone Synapse artifacts — copy them into your own MI project's artifact folders
(see Deploy). The repository layout is flat:

```
wso2-mi-boomi-datahub/
├── templates/
│   ├── mdh-get-golden-record.xml
│   ├── mdh-get-quarantine-entry.xml
│   ├── mdh-update-golden-records.xml
│   ├── mdh-stage-golden-records.xml             # stage a batch (Update w/ Staging Area ID)
│   ├── mdh-get-batch-update-status.xml          # poll async Update/Stage batch status
│   ├── mdh-query-golden-records.xml
│   ├── mdh-query-quarantine-entries.xml
│   ├── mdh-fetch-channel-updates.xml            # single fetch (you drive ack + fetch)
│   ├── mdh-fetch-channel-updates-ack.xml        # fetch + acknowledge one batch, return it
│   ├── mdh-acknowledge-channel-updates.xml      # acknowledge a batch by id (manual-ack flow)
│   └── mdh-match-entities.xml
├── sequences/
│   ├── boomi-mdh-init.xml      # resolves Hub base URL + Authorization from the vault
│   └── boomi-mdh-fault.xml     # returns 502 + <error> on transport/timeout failures
├── api/
│   └── boomi-mdh-api.xml       # sample REST API delegating to the templates
├── .github/workflows/
│   └── xml-lint.yml            # CI: XML well-formedness check
├── .gitignore
├── LICENSE                     # Apache License 2.0
├── NOTICE
└── README.md
```

## Deploy

Drop the artifacts into your MI project (MI 4.x layout):

```
src/main/wso2mi/artifacts/templates/    <- the 11 mdh-*.xml
src/main/wso2mi/artifacts/sequences/    <- boomi-mdh-init.xml, boomi-mdh-fault.xml
src/main/wso2mi/artifacts/apis/         <- boomi-mdh-api.xml
```

Add them to the composite exporter, build the CApp, deploy. If you only want the templates,
you can skip the API and `boomi-mdh-init`/`boomi-mdh-fault` and call the templates from your
own flows.

## Configure

**Hub base URL** — set once in `boomi-mdh-init.xml` (`MDH_HUB_BASE_URL`). The default is
`https://c01-usa-east.hub.boomi.com/mdm`; change it to the Hub Cloud that hosts your repository.
You can confirm your repository's host from the Repositories UI in Boomi — regions include:

```
USA East Hub Cloud      https://c01-usa-east.hub.boomi.com/mdm
USA East Hub Cloud 02   https://c02-usa-east.hub.boomi.com/mdm
Canada Hub Cloud 01     https://c01-ca.hub.boomi.com/mdm
GBR Hub Cloud           https://c01-gbr.hub.boomi.com/mdm
GB Hub Cloud            https://c01-gbr.hub.gb.boomi.com/mdm
DEU Hub Cloud 01        https://c01-deu.hub.boomi.com/mdm
Singapore Hub Cloud 01  https://c01-sg.hub.boomi.com/mdm
Japan Hub Cloud 01      https://c01-jp.hub.boomi.com/mdm
Australia Hub Cloud     https://c01-aus.hub.boomi.com/mdm
```

**Secrets (Secure Vault)** create two entries so credentials never sit in the XML:

```
boomi_mdh_username   Hub Repository API username
boomi_mdh_token      Hub Repository API token
```

`boomi-mdh-init` reads them and builds `Authorization: Basic base64(username:token)`.

> **Security note:** the built credential lives in the `MDH_AUTHORIZATION` synapse property and is
> sent as a transport header, so it can surface if you enable DEBUG/wire logging. Keep wire-level
> logging off in production, and prefer JWT/bearer where your Hub supports it.

**universeId** — the universe (domain) UUID. In the sample API it's passed per request as
`?universeId=...` (different domains = different UUIDs). From the Hub UI it's the ID in the
Repositories URL.

## Auth: basic vs JWT

Every template takes a single `authorization` param — the full header value — so it works with
either scheme:

- **Basic** (default here): `Basic <base64(user:token)>`. Do **not** send `repositoryId`.
- **JWT/Bearer**: set `authorization` to `Bearer <jwt>` and **also** pass `repositoryId`
  (the Repository API requires it with JWT only). Every template already accepts `repositoryId`
  and appends it to the query string only when non-empty.

## Calling a template directly

The templates operate on the current message, so for the POST-with-body operations, build the
payload first, then call:

```xml
<payloadFactory media-type="xml">
    <format>
        <RecordQueryRequest limit="200" includeSourceLinks="true">
            <filter op="AND">
                <fieldValue>
                    <fieldId>EMAIL</fieldId>
                    <operator>EQUALS</operator>
                    <value>$1</value>
                </fieldValue>
            </filter>
        </RecordQueryRequest>
    </format>
    <args><arg expression="$ctx:searchEmail"/></args>
</payloadFactory>

<call-template target="mdh-query-golden-records">
    <with-param name="hubBaseUrl"    value="{get-property('MDH_HUB_BASE_URL')}"/>
    <with-param name="universeId"    value="851a6a64-6a88-4916-a5b7-d6a974d54318"/>
    <with-param name="authorization" value="{get-property('MDH_AUTHORIZATION')}"/>
    <with-param name="repositoryId"  value=""/>
    <with-param name="includeReferenceTitle" value=""/>
</call-template>
<!-- the Boomi response is now the message payload; transform or <respond/> -->
```

After the template returns, the Boomi response is in the message and the backend HTTP status is
in `$axis2:HTTP_SC`.

## Sample API — quick tests

Assuming universe `u=851a6a64-6a88-4916-a5b7-d6a974d54318` and basic auth configured:

```bash
# Get Golden Record
curl "http://localhost:8290/boomi-mdh/golden-records/0fc4eb6d-f12c-4cdd-ba37-5475caed4887?universeId=$u"

# Get Quarantine Entry (source + entity)
curl "http://localhost:8290/boomi-mdh/quarantine/sources/CRM_SYSTEM/entities/CUST_12345?universeId=$u"

# Query Golden Records
curl -X POST "http://localhost:8290/boomi-mdh/golden-records/query?universeId=$u" \
  -H 'Content-Type: application/xml' \
  -d '<RecordQueryRequest limit="50"><filter op="AND"><fieldValue><fieldId>EMAIL</fieldId><operator>EQUALS</operator><value>a@b.com</value></fieldValue></filter></RecordQueryRequest>'

# Query Quarantine Entries
curl -X POST "http://localhost:8290/boomi-mdh/quarantine/query?universeId=$u" \
  -H 'Content-Type: application/xml' \
  -d '<QuarantineQueryRequest limit="50" type="ACTIVE"><filter op="AND"><cause>POSSIBLE_DUPLICATE</cause></filter></QuarantineQueryRequest>'

# Match Entities (de-dup check, no staging)
curl -X POST "http://localhost:8290/boomi-mdh/match?universeId=$u" \
  -H 'Content-Type: application/xml' \
  -d '<batch src="CRM_SYSTEM"><contact><id>1</id><name>bobby</name><email>bob@gmail.com</email></contact></batch>'

# Update Golden Records (contribute a batch)
curl -X POST "http://localhost:8290/boomi-mdh/golden-records?universeId=$u&async=true" \
  -H 'Content-Type: application/xml' \
  -d '<batch src="CRM_SYSTEM"><contact><id>1</id><name>bob</name><email>bob@gmail.com</email></contact></batch>'

# Fetch Channel Updates - FIRST fetch: no trailing segment (do NOT use /0)
curl -X POST "http://localhost:8290/boomi-mdh/sources/CRM_SYSTEM/updates?universeId=$u"
# ...then acknowledge that batch and fetch the next by putting its id in the path
# (batchId = the "id" attribute of the <batch> returned by the first fetch):
curl -X POST "http://localhost:8290/boomi-mdh/sources/CRM_SYSTEM/updates/$batchId?universeId=$u"

# Fetch + Acknowledge one batch: fetch the next batch, acknowledge it, and return it.
# Empty response = nothing pending. This is the simplest way for a consumer to drain a channel.
curl -X POST "http://localhost:8290/boomi-mdh/sources/CRM_SYSTEM/next-update?universeId=$u"

# Manual acknowledge: fetch with the single call above, process the batch, THEN acknowledge it
# by id (at-least-once). batchId = the "id" attribute of the <batch> you fetched and handled.
curl -X POST "http://localhost:8290/boomi-mdh/sources/CRM_SYSTEM/updates/$batchId/acknowledge?universeId=$u"
```

## Behaviours worth knowing

- **Optional query params are omitted, not blanked.** Each template sets `uri.var.*` only when
  the param is non-empty, and the URI template uses RFC 6570 form expansion
  (`{?uri.var.repositoryId,...}`), so unset params never appear in the URL. This is what keeps
  `repositoryId` off basic-auth requests.
- **The base URL uses reserved expansion** (`{+uri.var.hubBaseUrl}`) so `https://…` is not
  percent-encoded. Path IDs use plain expansion. If a `sourceId`/`entityId` contains reserved
  characters (`/`, `%`, spaces, …), URL-encode it before passing so it lands correctly in the
  path.
- **`REST_URL_POSTFIX` is cleared** in every template. Without this, MI appends the inbound API
  resource's trailing path to the outbound Boomi URL and you get 404s — the classic gotcha when
  calling a backend from inside an API resource.
- **Fetch = acknowledge + fetch, and the first call omits `updateId`.** The trailing
  `{updateId}` segment acknowledges a *previous* batch, so the first fetch must have **no**
  segment (`POST .../updates`) — there's nothing to acknowledge yet. Do **not** send
  `.../updates/0`: Boomi reads `0` as a real batch id and rejects it with
  *"There is no update with id '0'…"*. On each subsequent poll, pass the `id` attribute from the
  `<batch>` you just received (`POST .../updates/{id}`) to acknowledge it and get the next batch.
  Until you acknowledge a batch, repeated fetches keep returning that **same** batch. A `204`
  means nothing is pending. The template makes `updateId` optional and builds the path segment
  accordingly; the sample API exposes both `/updates` and `/updates/{updateId}` resources.
- **Fetch + acknowledge (`mdh-fetch-channel-updates-ack`)** is the one-shot consume: it fetches
  the next batch, backs the payload up into a property, calls the explicit acknowledge endpoint
  (`POST .../updates/{id}/acknowledge`) for that batch, then restores the backed-up payload as the
  response so the caller receives the original batch — now acknowledged and never redelivered.
  The backup is essential: the acknowledge is a second HTTP call whose response would otherwise
  overwrite the batch in the message. It exposes the acknowledged id as property `MDH_ACK_ID`; an
  empty response body means nothing was pending. Because it acknowledges *before* returning, it's
  at-most-once. For at-least-once, use `mdh-fetch-channel-updates` to fetch, process the batch,
  then `mdh-acknowledge-channel-updates` to acknowledge it only once it's safely handled. Note
  `acknowledge` differs from acknowledging-via-next-fetch: it acks the batch you name *without*
  pulling the next one, which is what lets you return the same batch you just fetched.
- **Update is async by default** — a `202` returns the batch-update resource URL (its last
  segment is the `batchId`). Poll processing with
  `GET /universes/{u}/records/updates/{batchId}` (add `?includeEntities=true` for per-entity
  detail). Set `async=false` to block until the batch completes.
- **Boomi 4xx/5xx pass straight through.** The `call` mediator treats a backend error status as a
  normal response, so the Boomi status code and `<error>` body are returned to the caller as-is.
  `boomi-mdh-fault` only fires on connection/timeout failures (it returns `502`).
- **End-dating.** `mdh-update-golden-records` can end-date via `op="DELETE"` on an entity, but
  Boomi also has dedicated end-date operations (`.../records/{id}/enddate` and
  `.../records/enddate`). Add them the same way if you need them.

Reference: Boomi DataHub Repository API — https://developer.boomi.com/docs/datahub-api-reference

## License

Licensed under the Apache License, Version 2.0 (SPDX: `Apache-2.0`). See the [LICENSE](LICENSE)
and [NOTICE](NOTICE) files for the full text and attributions. Unless required by applicable law
or agreed to in writing, the software is distributed on an "AS IS" BASIS, WITHOUT WARRANTIES OR
CONDITIONS OF ANY KIND.

Contributions are accepted under the same license: per Apache-2.0 §5, any contribution you
intentionally submit for inclusion is licensed under Apache-2.0 without additional terms.

## Trademarks

This project is an independent integration aid and is not affiliated with, endorsed by, or
sponsored by any of the products it interoperates with. Boomi and Boomi DataHub (Master Data
Hub) are trademarks of Boomi, LP; WSO2 and WSO2 Micro Integrator are trademarks of WSO2 LLC.
All other trademarks are the property of their respective owners.

