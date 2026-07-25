# Contributing

Thanks for considering a contribution. This project is a small library of WSO2 Micro
Integrator artifacts wrapping the Boomi DataHub Repository API, and its value depends on
every template behaving the same way - so most of this document is about conventions.

By contributing you agree that your contribution is licensed under the Apache License 2.0,
per [§5 of the licence](LICENSE): any contribution intentionally submitted for inclusion is
licensed under Apache-2.0 without additional terms. No separate CLA is required.

## Ways to contribute

- **Report a bug.** Include the artifact, your MI version, what you called, what Boomi
  returned, and what you expected. Redact credentials, repository IDs and record data.
- **Close a parity gap.** The project's goal is to cover everything the Boomi Data Hub
  connector does (see **Goal: connector parity** in the README). If you find a connector
  behaviour - an operation, an option, a tracked property - that these templates don't
  reproduce, that is the highest-value contribution there is. Say which connector version
  you compared against.
- **Improve the docs.** Corrections to endpoint paths, parameter names and Boomi behaviour
  are welcome, especially where you have verified them against a live repository.
- **Report a documentation/behaviour mismatch.** If the README describes something the code
  doesn't do, that is a bug.

### Operations outside connector parity

The Repository API is larger than the connector: record purge, restore, unlink, history and
metadata, the propagate/outbound half of Channels, and the Transactions and Universes groups
are all unwrapped. That is deliberate scope, not a backlog.

Pull requests wrapping them are still welcome - they follow the same conventions and cost
nothing to carry - but please open an issue first describing the use case. Anything accepted
beyond parity is documented as such in the README's **Scope** section, so users can tell the
core promise from the extras.

For anything larger than a fix, open an issue first so we can agree the shape before you
write it.

## Before you start

You will need:

- **WSO2 Micro Integrator 4.x** (artifacts use the MI 4.x project layout).
- **A Boomi DataHub repository** you can test against, with a Hub Repository API username
  and token from the repository's **Configure** tab.
- `xmllint` (`libxml2-utils`) to run the CI checks locally.

There is no build step - these are plain Synapse artifacts. Copy them into an MI project,
add them to the composite exporter, build the CApp and deploy, as described in the README.

## Template conventions

Every template in `templates/` follows the same shape. A pull request that departs from it
will be asked to change, because the consistency is what makes the library predictable and
mechanically checkable.

**Naming and structure**

1. One artifact per file. The `<template name="...">` **must** equal the filename without
   `.xml` - CI enforces this.
2. Name templates `mdh-<verb>-<noun>`, matching Boomi's operation name where one exists
   (`mdh-query-golden-records`, `mdh-acknowledge-channel-updates`).
3. Start every file with the Apache licence header, then a comment block giving: the Boomi
   endpoint, the request-body shape if any, notable behaviour (async, paging, caps), and each
   parameter marked `(req)` or `(opt)`. CI enforces the licence header.

**Parameters**

4. Take `hubBaseUrl`, `universeId`, `authorization` and `repositoryId` on every template that
   calls Boomi, in that order, so callers are interchangeable.
5. `authorization` is the complete header value (`Basic …` or `Bearer …`). Templates never
   build credentials themselves.
6. Set optional `uri.var.*` properties only when the parameter is non-empty:

   ```xml
   <filter xpath="string-length(normalize-space($func:repositoryId)) &gt; 0">
       <then><property name="uri.var.repositoryId" expression="$func:repositoryId"/></then>
   </filter>
   ```

**URI templates**

7. Use RFC 6570 **form expansion** for optional query parameters -
   `{?uri.var.repositoryId,uri.var.limit}` - so unset parameters are omitted rather than sent
   blank. Sending `repositoryId=` breaks basic auth.
8. Use **reserved expansion** for the base URL - `{+uri.var.hubBaseUrl}` - so `https://` is
   not percent-encoded. Path IDs use plain expansion.

**Message handling**

9. **Clear `REST_URL_POSTFIX`** in every template that makes an outbound `<call>`:

   ```xml
   <property name="REST_URL_POSTFIX" scope="axis2" action="remove"/>
   ```

   Without it, MI appends the inbound API resource's trailing path to the outbound Boomi URL
   and you get unexplained 404s. CI enforces this.
10. Set `NO_ENTITY_BODY` for GET and DELETE; set `messageType` and `ContentType` for calls
    that send a body.
11. Templates operate on the current message and leave the Boomi response as the payload.
    Building request bodies and transforming responses stays with the caller.
12. If a template makes more than one HTTP call, back up the payload into a property with
    `<enrich>` and restore it afterwards - see `mdh-fetch-channel-updates-ack`. A second call
    will otherwise overwrite the first response.

**Error handling**

13. Keep endpoint suspension disabled (`initialDuration=-1`, `retriesBeforeSuspension=0`).
    These endpoints are shared across every caller; suspending one would break unrelated
    flows.
14. Let Boomi 4xx/5xx pass through as normal responses. Do not translate them into faults.
    `boomi-mdh-fault` handles connection and timeout failures only.
15. If a template swallows a failure that the caller must know about, surface it explicitly -
    set a property, return an `<error>` body and set a non-2xx status, as
    `mdh-fetch-channel-updates-ack` does with `MDH_ACK_FAILED`.

**Never**

16. Never hardcode credentials, tokens, account IDs, repository IDs or universe UUIDs.
17. Never log the `Authorization` header or `MDH_AUTHORIZATION`.

## Adding a new operation

1. Find the operation in the
   [Boomi DataHub Repository API reference](https://developer.boomi.com/docs/datahub-api-reference)
   and note its exact path, method, request body and response shape.
2. Copy the closest existing template - a GET like `mdh-get-golden-record`, or a
   POST-with-body like `mdh-query-golden-records` - and change the path, method and comment
   block.
3. Add a resource to `api/boomi-mdh-api.xml` if the operation makes sense on the sample API.
4. Update the README: the operations table, the file tree, a `curl` example under **Sample
   API - quick tests**, and a bullet under **Behaviours worth knowing** if the operation has
   non-obvious semantics (async, paging, caps, ordering).
5. Run the checks below, then open the pull request.

Documentation is part of the change, not a follow-up. A template that isn't in the README
effectively doesn't exist.

## Checks

CI runs XML well-formedness on every push and pull request. Run it locally first:

```bash
for f in $(find api sequences templates -name '*.xml'); do xmllint --noout "$f" || echo "FAIL $f"; done
```

Then confirm by hand:

- `<template name>` / `<sequence name>` matches the filename.
- Every template with a `<call>` clears `REST_URL_POSTFIX`.
- The Apache licence header is present.
- No literal credentials.

**Please test against a real Boomi repository before submitting.** Well-formed XML proves
very little here - most defects in this library are wrong paths, wrong parameter names, or
misunderstood Boomi semantics, none of which a linter can catch. Say in your pull request
what you tested and what you couldn't; "untested against a live Hub" is useful information,
not a reason to withhold the contribution.

## Pull requests

- One logical change per pull request.
- Write commit subjects in the imperative, under ~72 characters: `Add mdh-restore-record
  template`, `Fix repositoryId omitted on quarantine query`.
- In the description, say what changed, why, how you tested it, and note any behaviour you
  inferred rather than verified.
- Keep unrelated reformatting out of the diff.

Expect review comments about the conventions above. They are not stylistic preferences -
each one exists because getting it wrong produces a silent failure that is hard to diagnose.

## Code of conduct

Be respectful and assume good faith. Discussion stays on the technical merits. Maintainers
may edit, lock or remove contributions that are abusive or off-topic, and may block repeat
offenders. If someone's conduct is a problem, raise it privately with the maintainer rather
than in the thread.