# WSO2 Micro Integrator templates for Boomi DataHub (MDH)

[![License: Apache 2.0](https://img.shields.io/badge/License-Apache_2.0-blue.svg)](LICENSE)

**The capabilities of the Boomi DataHub connector, on WSO2 Micro Integrator.**

Boomi Integration ships a Data Hub connector that gives a process low-code access to a Data
Hub repository. If your integration layer is WSO2 rather than Boomi, that connector isn't
available to you. This project closes that gap: eleven reusable Synapse **sequence templates**
covering every operation the connector exposes, calling the Boomi DataHub Repository API
directly over HTTPS, plus a sample REST API and shared config/fault sequences that show how to
wire them up.

Each template encapsulates the one thing that's tedious and error-prone to repeat: the exact
endpoint URL, HTTP method, content type, auth header, and the optional-query-param handling.
Everything else (building request bodies, transforming responses, error policy) stays with the
caller, so the templates compose cleanly into your own proxies/APIs/sequences.

## Goal: connector parity

The aim is that anything you can do with the Boomi Data Hub connector inside a Boomi
Integration process, you can do here in an MI sequence. Boomi documents the connector as
exposing five operations. All five are covered:

| Boomi Data Hub connector operation | This project |
|---|---|
| **Update Golden Records** - create, update and end-date golden records; quarantine source entities; optionally send to staging | `mdh-update-golden-records` (commit) and `mdh-stage-golden-records` (staging area) |
| **Query Golden Records** - retrieve active golden records | `mdh-query-golden-records` |
| **Query Quarantine Entries** - retrieve quarantined records | `mdh-query-quarantine-entries` |
| **Fetch Channel Updates** - fetch batches of source record update requests, with automatic or manual acknowledgement | `mdh-fetch-channel-updates-ack` (automatic) or `mdh-fetch-channel-updates` + `mdh-acknowledge-channel-updates` (manual) |
| **Match Entities** - match results for a batch of entities from a contributing source | `mdh-match-entities` |

The connector's two tracked properties have equivalents. *Mdm Current Delivery Id* - the
acknowledged batch id under manual acknowledgement - is exposed as the `MDH_ACK_ID` property.
*Query Total Count* is not lifted into a property, but Boomi returns `resultCount` and
`totalCount` as attributes on the query response, which the templates pass through intact:

```xml
<property name="totalCount" expression="$body/*/@totalCount"/>
```

**Three templates go beyond the connector**, wrapping Repository API operations it does not
expose: `mdh-get-golden-record` (fetch one record by id), `mdh-get-quarantine-entry` (fetch one
quarantine entry by source and entity id) and `mdh-get-batch-update-status` (poll an
asynchronous Update or Stage batch).

> Parity is judged against Boomi's published connector documentation. Connector operation sets
> change between releases - if you are relying on this claim, check it against the version your
> team runs.

## What this is for

Boomi Data Hub holds the trusted version of the records that several systems all think they
own - customers, products, suppliers, sites. Salesforce has "Bob Smith", the webshop has
"robert smith", the support desk has "B. Smith". Data Hub decides those are one person, keeps
one golden record, and tracks which system contributed what.

That creates two-way traffic. Source systems contribute entities inward; Data Hub propagates
the resulting changes back outward to every system that subscribes to them. Something has to
sit in the middle and move that traffic. In a Boomi shop that's a process built with the Data
Hub connector. Here it's an MI sequence built from these templates.

### If you already run WSO2, you don't need Boomi Integration as well

The common assumption when Data Hub is bought is that Boomi Integration must come with it,
because the connector is how you talk to a repository. But the connector is a convenience
wrapper over the Repository API, not a privileged path into it - contributing batches, fetching
channel updates, querying golden records, pulling quarantine and staging for review are all
plain HTTPS calls. Since the parity table above covers every connector operation, an
integration engine you already operate can do the same work.

What that avoids is not mainly a licence line - it's a second operational footprint: another
runtime to deploy, monitor and patch; another skill set, since Boomi process building and
Synapse mediation are different disciplines; another delivery pipeline; and split
observability, with half your data flows traced in one console and half in another during an
incident. For moving master data between a handful of systems, that's a large standing
overhead for a small amount of work.

There is also one thing the connector cannot do at all: **it does not support JWT.** It is
basic auth only, and basic grants administrator-level permissions on every Repository API
request. JWT authorises against the caller's platform role instead, and works only when calling
the Repository API directly - as these templates do. If admin-level credentials sitting inside
an integration won't pass your security review, this is the route. See
[Auth: basic vs JWT](#auth-basic-vs-jwt).

> **Two honest limits.** Whether Data Hub can be licensed without Integration is a commercial
> question for Boomi and varies by agreement - the claim here is technical, not contractual.
> And none of this replaces Data Hub itself: models, sources, match rules and channels are
> still configured in the Data Hub UI. What's removed is the need for a second *integration*
> platform, not the MDM configuration work.

## Operations → endpoints

All paths are relative to the Hub base, e.g. `https://c01-usa-east.hub.boomi.com/mdm`.

| Template | Method | Path | Request body |
|---|---|---|---|
| `mdh-get-golden-record` | GET | `/universes/{u}/records/{recordId}` | - |
| `mdh-get-quarantine-entry` | GET | `/universes/{u}/quarantine/sources/{sourceId}/entities/{entityId}` | - |
| `mdh-update-golden-records` | POST | `/universes/{u}/records` | `<batch src="...">` |
| `mdh-stage-golden-records` | POST | `/universes/{u}/staging/{stagingCode}` | `<batch src="...">` |
| `mdh-get-batch-update-status` | GET | `/universes/{u}/records/updates/{batchId}` | - |
| `mdh-query-golden-records` | POST | `/universes/{u}/records/query` | `<RecordQueryRequest>` |
| `mdh-query-quarantine-entries` | POST | `/universes/{u}/quarantine/query` | `<QuarantineQueryRequest>` |
| `mdh-fetch-channel-updates` | POST | `/universes/{u}/sources/{sourceId}/updates` then `/updates/{updateId}` | - (empty) |
| `mdh-fetch-channel-updates-ack` | POST | fetch + acknowledge one batch, return it | - (empty) |
| `mdh-acknowledge-channel-updates` | POST | `/universes/{u}/sources/{sourceId}/updates/{updateId}/acknowledge` | - (empty) |
| `mdh-match-entities` | POST | `/universes/{u}/match` | `<batch src="...">` |

A few things don't map where you'd expect in Boomi's own API grouping, so a heads-up:

- **Update Golden Records** is a *Batch* operation (`POST .../records`), not a "Records" one.
  Staging the same batch instead of committing it straight to golden records is a separate
  path (`POST .../staging/{stagingCode}`) - that's `mdh-stage-golden-records`, and it's the
  In the connector these are one operation with a Staging Area ID; here they are two
  templates.
- **Get Quarantine Entry** is fetched *by source + source entity ID*. There is no
  "get by transactionId" GET; to look entries up by cause/date/transaction, use
  `mdh-query-quarantine-entries`.
- **Consuming channel updates** - there are two acknowledge strategies, matching the
  connector's automatic and manual acknowledgement modes:
  - *Auto (fetch + acknowledge in one call).* `mdh-fetch-channel-updates-ack` fetches the next
    batch, backs up the payload, acknowledges that batch (so Boomi won't redeliver it), and
    returns the original batch to the caller. Simplest for consumers; at-most-once (the batch is
    acknowledged before you get it).
  - *Manual (acknowledge after processing).* Use `mdh-fetch-channel-updates` to fetch, process the
    batch downstream, then `mdh-acknowledge-channel-updates` to acknowledge it by id - at-least-once,
    since nothing is acknowledged until you've safely handled it.

  `mdh-fetch-channel-updates-ack` reuses both `mdh-fetch-channel-updates` and
  `mdh-acknowledge-channel-updates`, so all three must be deployed together.

## Files

These are standalone Synapse artifacts - copy them into your own MI project's artifact folders
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
├── CONTRIBUTING.md
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

**Hub base URL** - set once in `boomi-mdh-init.xml` (`MDH_HUB_BASE_URL`). The default is
`https://c01-usa-east.hub.boomi.com/mdm`; change it to the Hub Cloud that hosts your repository.
You can confirm your repository's host from the Repositories UI in Boomi - regions include:

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

**Secrets (Secure Vault)** - create two entries so credentials never sit in the XML:

```
boomi_mdh_username   Hub Repository API username
boomi_mdh_token      Hub Repository API token
```

Both are on the repository's **Configure** tab in the Hub UI. `boomi-mdh-init` reads them and
builds `Authorization: Basic base64(username:token)`.

> **Security note:** the built credential lives in the `MDH_AUTHORIZATION` synapse property and is
> sent as a transport header, so it can surface if you enable DEBUG/wire logging. Keep wire-level
> logging off in production, and prefer JWT/bearer where your Hub supports it. 

**universeId** - the universe (domain) UUID. In the sample API it's passed per request as
`?universeId=...` (different domains = different UUIDs). From the Hub UI it's the ID in the
Repositories URL.

## Auth: basic vs JWT

Every template takes a single `authorization` param - the full header value - so it works with
either scheme:

- **Basic** (default here): `Basic <base64(user:token)>`. Do **not** send `repositoryId`.
- **JWT/Bearer**: set `authorization` to `Bearer <jwt>` and **also** pass `repositoryId`
  (the Repository API requires it with JWT only). Every template already accepts `repositoryId`
  and appends it to the query string only when non-empty.

JWT is a capability the Boomi connector does not have: it scopes access to the caller's platform
role, where basic auth is administrator-level on every request. Note that JWTs are minted by the Boomi *platform* API, not the Hub, and expire after a
few minutes - mint per request or cache with a safe margin.

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

## Sample API - quick tests

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

# Stage Golden Records (same batch, staged for review instead of committed)
# stagingCode = the Staging Area ID configured on the source in the Hub UI.
curl -X POST "http://localhost:8290/boomi-mdh/staging/STAGE_AREA_1?universeId=$u&async=true" \
  -H 'Content-Type: application/xml' \
  -d '<batch src="CRM_SYSTEM"><contact><id>1</id><name>bob</name><email>bob@gmail.com</email></contact></batch>'

# Batch update status: poll an async Update/Stage batch.
# batchId = the last path segment of the resource URL returned by the 202 above.
curl "http://localhost:8290/boomi-mdh/batch-updates/$batchId?universeId=$u&includeEntities=true"

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
- **`REST_URL_POSTFIX` is cleared** in every template that makes an outbound call. Without this,
  MI appends the inbound API resource's trailing path to the outbound Boomi URL and you get
  404s - the classic gotcha when calling a backend from inside an API resource.
- **Fetch = acknowledge + fetch, and the first call omits `updateId`.** The trailing
  `{updateId}` segment acknowledges a *previous* batch, so the first fetch must have **no**
  segment (`POST .../updates`) - there's nothing to acknowledge yet. Do **not** send
  `.../updates/0`: Boomi reads `0` as a real batch id and rejects it with
  *"There is no update with id '0'…"*. On each subsequent poll, pass the `id` attribute from the
  `<batch>` you just received (`POST .../updates/{id}`) to acknowledge it and get the next batch.
  Until you acknowledge a batch, repeated fetches keep returning that **same** batch. A `204`
  means nothing is pending. The template makes `updateId` optional and builds the path segment
  accordingly; the sample API exposes both `/updates` and `/updates/{updateId}` resources.
- **Fetch + acknowledge (`mdh-fetch-channel-updates-ack`)** is the one-shot consume: it fetches
  the next batch, backs the payload up into a property, calls the explicit acknowledge endpoint
  (`POST .../updates/{id}/acknowledge`) for that batch, then restores the backed-up payload as the
  response so the caller receives the original batch - now acknowledged and never redelivered.
  The backup is essential: the acknowledge is a second HTTP call whose response would otherwise
  overwrite the batch in the message. It exposes the acknowledged id as property `MDH_ACK_ID` -
  the equivalent of the connector's *Mdm Current Delivery Id* tracked property; an empty
  response body means nothing was pending. Because it acknowledges *before* returning, it's
  at-most-once. For at-least-once, use `mdh-fetch-channel-updates` to fetch, process the batch,
  then `mdh-acknowledge-channel-updates` to acknowledge it only once it's safely handled. Note
  `acknowledge` differs from acknowledging-via-next-fetch: it acks the batch you name *without*
  pulling the next one, which is what lets you return the same batch you just fetched.
- **If the acknowledge fails, you are told.** When the acknowledge call returns a non-2xx status,
  `mdh-fetch-channel-updates-ack` does **not** hand back the batch as though it succeeded: it sets
  `MDH_ACK_FAILED=true`, returns an `<error>` body instead of the batch, and sets the response
  status to `502`. The batch remains unacknowledged in Boomi and will be redelivered on the next
  fetch, so callers should treat this as "nothing consumed" rather than retrying downstream work.
- **Update and stage are async by default** - a `202` returns the batch-update resource URL (its
  last segment is the `batchId`). Poll processing with `mdh-get-batch-update-status`
  (`GET /universes/{u}/records/updates/{batchId}`, add `?includeEntities=true` for per-entity
  detail). Set `async=false` to block until the batch completes. `mdh-stage-golden-records` also
  accepts `returnEntities`.
- **Match takes an optional `ambiguousMatchSampleSize`**, which caps how many sample matches are
  returned per ambiguous entity. Quarantine queries accept `includeData` to return the quarantined
  payload alongside the metadata.
- **Boomi 4xx/5xx pass straight through.** The `call` mediator treats a backend error status as a
  normal response, so the Boomi status code and `<error>` body are returned to the caller as-is.
  `boomi-mdh-fault` only fires on connection/timeout failures (it returns `502`).
- **End-dating goes through Update**, via `op="DELETE"` on an entity in the `<batch>` - the
  same way the connector does it. The Repository API also has dedicated end-date endpoints
  (`.../records/{id}/enddate`, `.../records/enddate`); they are outside connector parity and so
  outside this project's default scope, but they wrap exactly like the existing templates.

## Scope

In scope: everything the Boomi Data Hub connector does, plus the three read operations listed
under [Goal: connector parity](#goal-connector-parity).

Out of scope by default: Repository API operations the connector does not expose - record
purge, restore, unlink, history and metadata; the propagate/outbound half of the Channels
group; and the Transactions and Universes groups. These are deliberate omissions, not a
backlog. If you need one, it wraps exactly like the existing templates - see
[CONTRIBUTING.md](CONTRIBUTING.md).

Reference: Boomi DataHub Repository API - https://developer.boomi.com/docs/datahub-api-reference

## License

Licensed under the Apache License, Version 2.0 (SPDX: `Apache-2.0`). See the [LICENSE](LICENSE)
and [NOTICE](NOTICE) files for the full text and attributions. Unless required by applicable law
or agreed to in writing, the software is distributed on an "AS IS" BASIS, WITHOUT WARRANTIES OR
CONDITIONS OF ANY KIND.

Contributions are accepted under the same license: per Apache-2.0 §5, any contribution you
intentionally submit for inclusion is licensed under Apache-2.0 without additional terms. See
[CONTRIBUTING.md](CONTRIBUTING.md).

## Trademarks

This project is an independent integration aid and is not affiliated with, endorsed by, or
sponsored by any of the products it interoperates with. Boomi and Boomi DataHub (Master Data
Hub) are trademarks of Boomi, LP; WSO2 and WSO2 Micro Integrator are trademarks of WSO2 LLC.
All other trademarks are the property of their respective owners.