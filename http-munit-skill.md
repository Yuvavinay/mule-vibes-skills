# HTTP MUnit skill

Loaded by the [dispatcher](./SKILL.md) for APIs driven by an HTTP Listener, with or without an
APIkit router. If you arrive without the dispatcher's handoff block, run the dispatcher first.

> ⛔ **Never run tests.** Never execute `mvn test` or trigger a test run at any point. If the
> user asks to run, print the command and stop.

Every test sends a real HTTP request to the API's own listener through `testHttpRequestConfig`.
The listener, the router, the flows under test and every flow they call through `flow-ref` run
for real; only the backend calls (Step 2) are mocked, never a `flow-ref`. Each test loads its
request body with `MunitTools::getResourceAsString` and validates with one **Assert expression**.

The skill serves two kinds of API, given as `API_TYPE` in the dispatcher's handoff block:

| `API_TYPE` | The API has | The request is built from | Each test enables |
|---|---|---|---|
| `HTTP_APIKIT` | An `http:listener` and an `apikit:router` | The RAML contract (Step 4) and the DataWeave (Step 3) | The main flow and the operation flow it calls |
| `HTTP_LISTENER` | An `http:listener` and no router | The listener and the DataWeave only (Steps 3 and 4); no contract is read | The listener flow |

The type is set per listener flow: in an APIkit project, a listener flow without a router (a
health check, a callback) is tested the `HTTP_LISTENER` way.

The dispatcher's hard rules apply: the project receives MUnit files only (plus `pom.xml` only
if MUnit is missing), and all scratch work and caches go to `WORK_DIR`. Print one emoji
progress line per step.

---

## Step 1 — Set the scope

📂 Setting scope...

| Request | Scope | Suite written | Covers |
|---|---|---|---|
| `<file>.xml` | `FILE` | `<file>-test-suite.xml` | Every processor in every flow and sub-flow of that file. On `HTTP_LISTENER`, a file with a listener flow: every processor in every flow the listener calls, in any file (Step 2, call tree) |
| `apikit` | `APIKIT` (`HTTP_APIKIT` only) | `apikit-error-handler-test-suite.xml` | Every APIkit error the API handles |
| `errors` | `ERRORS` | `error-handler-test-suite.xml` | Every error-handler branch that the API's flows route errors to |
| `all` | `APIKIT` (`HTTP_APIKIT` only), then `FILE` for each file, then `ERRORS` | All of the above | Everything |

- **Resolve `<file>`:** match a relative path first, then a basename, against the files in
  `WORK_DIR/index.txt`. If none matches, list the closest names; if several match, ask.
- **Files in `all`:** every file that declares a `flow` or `sub-flow`, except files whose flows
  run only inside error handlers (covered by `APIKIT` and `ERRORS`). On `HTTP_APIKIT`, also skip
  the file that holds the main flow when its flows only route (the main flow, and operation flows
  that hold nothing but a `flow-ref`): they run inside every test. On `HTTP_LISTENER`, write one
  suite per file that holds a listener flow, covering the listener's whole call tree; a file
  whose flows all sit in some listener's call tree gets no suite of its own, and a flow that no
  listener reaches is tested directly (Step 8, flows with no endpoint of their own).
- **Existing suites:** list `src/test/munit/` once. If a suite you are about to write exists, or
  (for `APIKIT` and `ERRORS`) another suite already tests the same handler, read it and add only
  the scenarios it does not cover yet.

## Step 2 — Read the unit under test

🔍 Reading flows and building call tree...

Use `WORK_DIR/index.txt` to find flows, callers and handlers; do not search the project again.

| Scope | Read |
|---|---|
| `FILE` | The target file in full. Expand every `flow-ref` to a flow or sub-flow in the **same file**: it is part of the unit. Follow every `flow-ref` to **another file** as well, and read the called flow in full, then the flows it calls, recursively: it runs for real in the test, so its connector operations must be mocked and its DataWeave must not fail. |
| `APIKIT` | The listener flow, and the error handler that applies to it (Step 6). |
| `ERRORS` | Each handler that the API's flows use (Step 7), and each path that reaches it. |

Then read the listener flow: the `http:listener` element (`path`, `allowedMethods`, `config-ref`,
`response` and `error-response` status expressions). On `HTTP_APIKIT`, read the steps before the
router: they run for real in every test, so note which headers they read. On `HTTP_LISTENER`, the
whole listener flow is part of the unit, and a `choice` on `attributes.method`,
`attributes.requestPath` or `attributes.relativePath` marks one endpoint per route.

**Call tree (`HTTP_LISTENER`).** The unit is the listener flow and every flow and sub-flow it
reaches, in any file, and the listener's suite covers every processor in all of them. Build the
tree from `WORK_DIR/index.txt`: take each `flow-ref` in the listener flow, find the `flow` or
`sub-flow` it names, read that flow in full, and repeat for its own `flow-ref`s, including those
inside `choice`, `foreach`, `try`, `async`, `scatter-gather` and `until-successful`. Visit each
flow once, even when several flows call it or it calls itself.

- A `flow-ref` whose `name` is an expression (`#[vars.target]`) reaches every flow the
  expression can name: take the values from the scripts and property files that set it, or
  report it as unresolved.
- A flow reached through a connector (`vm:publish`, `anypoint-mq:publish`, an `http:request` to
  the app itself) is not called: that operation is a backend call and is mocked, and the flow
  that receives it is tested with its own source.

Save the tree as `WORK_DIR/calltree-<listener flow>.txt`: each flow with its file, the flow that
calls it, and the calling `flow-ref`'s `doc:id`.

Record every **backend call** on the path, with its `doc:id`, `doc:name` and the fields the next
processor reads from its result: `http:request`, `db:*`, `salesforce:*`, `os:*`,
`anypoint-mq:*`, `jms:*`, `vm:*`, `email:*`, `file:*`, `sftp:*`, `ftp:*`, `wsc:consume`,
`java:*` and any other connector operation, in the unit and in every flow it calls through
`flow-ref`, in any file. A `flow-ref` is never a backend call: the flow it calls runs, and the
connector operations inside it are mocked. Modules that only check or transform data in memory
(`validation:*`, `json:*`, `xml-module:*`) are not backend calls either: they run for real, and
the request must pass them.

> ⚠️ **Every connector operation on every path MUST be mocked** — including `os:retrieve`,
> `os:contains`, `db:select`, `salesforce:query` and every other operation that returns a value
> or can raise an error. An un-mocked connector operation will attempt a real connection and fail.

## Step 3 — Analyse the DataWeave on each path

⚙️ Analysing DataWeave scripts...

Read every DWL file the path uses (`resource="…dwl"`), including those of flows called in other
files, and every inline script in `ee:transform`, `set-variable`, `set-payload` and `choice`
`when` expressions. These notes shape every `in/` and `out/` file. For each script, note:

- **Inputs:** each `payload.x`, `vars.y`, `attributes.headers|queryParams|uriParams.z`,
  `attributes.method|requestPath|relativePath` and `p('key')` it reads, and its
  `input payload <mime>` directive, if any.
- **What breaks it:** coercions (`as Number`, `as Date {format: …}`); string functions (`lower`,
  `upper`, `replace`, `splitBy`, `trim`) and `++` on a field that could be null; index access
  (`payload[0].x` needs an array); `sizeOf`, `map` and `filter` on a value that must be an array.
- **Property keys built from data:** for `p('prefix.' ++ payload.x)`, search the property files
  for `prefix.` once, and use a value of `x` whose key exists.
- **Output:** the shape the next processor or `choice` reads.

The notes decide every fixture, on both kinds of API:

| A field the scripts read is … | It goes in |
|---|---|
| In the request body | `in/` file → `set-event` payload → `http:body` |
| A request header, query parameter or URI parameter | `http:headers`, `http:query-params`, `http:uri-params` |
| Returned by a backend call | `out/` file → that call's mock |
| A property, `p('key')` | Nowhere: MUnit loads the property files; pick request values whose keys exist |
| Computed inside the unit | Nowhere: the flow computes it |

Shape each `out/` response from what the **next** script or `choice` reads, not from what the API
finally returns, and give it the `mediaType` that script expects. When a field selects a branch,
set it to the value that selects the branch the test is for. Every fixture must hold every input
its scripts read, with the right type and format, so that no DataWeave fails in a test.

## Step 4 — Build the request (RAML only)

?? Reading RAML contract and building request ledger...

### `HTTP_LISTENER`: from the listener and the DataWeave

No contract is read: the listener and the Step 3 notes are the whole specification. Build one
entry per endpoint (each route of a `choice` on the method or path is one endpoint):

| Item | Source |
|---|---|
| Method | The listener's `allowedMethods`; else the method the route's `choice` tests; else `POST` when the scripts read `payload`, `GET` when they do not |
| Path | The listener `path`, each `{x}` filled with a value that passes the scripts. For a path ending in `/*`, the `relativePath` or `requestPath` the route's `choice` matches |
| Headers | Every `attributes.headers.x` the scripts read, plus `Content-Type` from the script's `input payload` directive or the format it parses (else `application/json`) |
| Query and URI parameters | Every `attributes.queryParams.x` and `attributes.uriParams.x` the scripts read |
| Body | Every `payload.x` the scripts on the route read, in the type and format that pass them: the `in/` file |
| Success status | The listener's `response` `statusCode` with the vars the route sets; 200 when it has none |

Send nothing the scripts do not read, and leave out nothing they do. Save each entry as
`WORK_DIR/listener/ledger/<name>.txt` (`<name>` as in step 5 below), with the path, size and time
of the flow and DWL files it came from; reuse it while none of them has changed.

### `HTTP_APIKIT`: from the RAML contract

> ℹ️ **RAML only.** Only RAML 0.8 and RAML 1.0 contracts are supported. If the project uses an
> OAS spec, block all `FILE` and `ERRORS` tests for that router, report the reason, and continue
> with what can be tested.

**Endpoint.** APIkit names operation flows `<method>:\<resource>[:<media-type>]:<config>`.
Backslashes are path separators and `(x)` is the URI parameter `{x}`:
`post:\orders\(id)\items:application\json:orders-config` is `POST /orders/{id}/items`. An
`apikit:flow-mapping` overrides the name. Keep the exact name of the main flow (the flow with the
`http:listener` and the `apikit:router`) and of each operation flow a test calls: every test
enables both (Step 8). When the project has several routers (two API versions, for example), the
main flow is the one whose `apikit:router` `config-ref` is the config at the end of the operation
flow's name.

**The request must pass both the router and the flow.** The contract decides what the router
accepts; the Step 3 notes decide what the flow's scripts read. Start from the contract's example,
then add each field the scripts read that the example lacks, in a form the contract allows.

**Contract.** Only `FILE` and `ERRORS` need it, because the real router validates the request;
`APIKIT` mocks the router. Each `apikit:config` has its own contract: an operation flow's name ends
with its config, and that config's contract gives the flow's ledger. Read the config's `api` (or
`raml`) attribute; for `api="${key}"`, look up that one key in the property files. The value says
where the contract is:

| Contract | `api` value | Found in | `<cache>` |
|---|---|---|---|
| **Dependency**: an Exchange asset declared in `pom.xml` | `resource::<groupId>:<assetId>:<version>:<classifier>:<type>:<mainFile>` (RAML, plain or fat archive) | The local Maven repository | `WORK_DIR/contract/<assetId>-<version>/` |
| **Inline**: kept in the project | A relative path such as `orders-api.raml` or `api/v1/orders.raml` | `src/main/resources/api/<path>`, else `src/main/resources/<path>`, else a search of `src/main/resources` for the file name (ask if several match) | `WORK_DIR/contract/inline-<path under src/main/resources without extension, / as ->/` |
| Neither (a URL, an absolute path, or non-RAML) | — | Not supported: report it and stop the tests that need it | — |

Never build an APIkit request the listener way when its contract cannot be found: the router
would reject it. Block those tests and report the reason.

The contract is prepared **once** per source and cached in `WORK_DIR`. Every later endpoint,
file, scope and run reads the cache; none of them extracts, copies or resolves the contract again.

```text
<cache>/
  source.txt   where it came from: path, size and last-modified time of each source file
  spec/        the contract as the router sees it: the extracted archive or a copy of the inline
               RAML root, with its libraries in spec/exchange_modules/ so that includes resolve
  index.txt    line numbers of every resource, method and top-level section
  shared.txt   the headers and query parameters each trait, resource type and security scheme adds
  ledger/      one file per endpoint
```

1. **Check the cache.** If `<cache>/source.txt` exists and every file it lists still has the same
   path, size and time, the cache is current: skip to 5. Otherwise delete `<cache>` (inside
   `WORK_DIR` only) and build it with 2–4.
2. **Find the source.** *Dependency:* the local repository is `<localRepository>` in
   `<HOME>/.m2/settings.xml` (read only that element, as the file can hold credentials; resolve
   `${user.home}`), else `-Dmaven.repo.local` in `MAVEN_OPTS`, else `<HOME>/.m2/repository`. The
   artifact is
   `<repo>/<groupId, . as />/<assetId>/<version>/<assetId>-<version>-<classifier>.<type>`.
   If it is missing, stop the tests that need it, print the path, and ask the user to build the
   project once (`mvn -DskipTests package`). Never download it yourself or use another contract.
   *Inline:* the API root, which is the folder that holds the main file or, when an `!include`
   leaves that folder with `../`, the nearest parent that holds every included file.
3. **Fill `spec/`**, never the project. *Dependency:* list the archive's entries first
   (`unzip -l`, `tar -tf`, or `[IO.Compression.ZipFile]::OpenRead('<zip>').Entries` in
   PowerShell) and refuse it if an entry is absolute or contains `..`; then extract it into
   `spec/`. A fat archive already holds `exchange_modules/`. Otherwise each `dependencies` entry
   in the extracted `exchange.json` is a library: extract its archive from
   `<repo>/<groupId, . as />/<assetId>/<version>/` (`<assetId>-<version>-raml-fragment.zip`)
   into `spec/exchange_modules/<groupId>/<assetId>/<version>/` with the same check, and repeat
   for its own dependencies. *Inline:* copy the API root into `spec/`; its `exchange_modules/`
   comes with it, and a library missing there is extracted from the Maven repository as above.
   Never run anything from the cache.
4. **Index once.** Run one search over the main RAML file and save it, with line numbers, as
   `index.txt`:

   ```text
   ^\s*/[^:]*:|^\s*(get|post|put|patch|delete|head|options):|^(uses|traits|resourceTypes|securitySchemes|securedBy|types|schemas|mediaType):
   ```

   A resource or section whose value is `!include <file>` lives in that file: index that file
   too, relative to the file that points to it. Then resolve every RAML trait, resource type and
   security scheme, following `uses:` aliases, includes and the libraries in
   `spec/exchange_modules/`, to the headers and query parameters it adds. Save them in
   `shared.txt`, keeping `<<parameters>>` as placeholders. Write `source.txt` last, so that an
   interrupted build is never taken for a current cache.
5. **Ledger per endpoint.** If `ledger/<name>.txt` exists, reuse it. Otherwise find the
   endpoint's resource and method in `index.txt`, read only those blocks of the spec (a nested
   RAML resource: each segment under its parent), take what the traits, resource types and
   security schemes add from `shared.txt` (filling in `<<parameters>>`), and write the ledger.
   `<name>` is the method and path with `-` for `/`, plus the media type when the operation flow
   names one: `POST /orders/{id}/items` → `POST-orders-{id}-items`.

Print the cache outcome: `→ Step 4: contract reused from cache (2 ledgers reused, 1 new)`.

**Ledger** — one per endpoint that reaches the unit:

| Item | Source |
|---|---|
| Headers | The method's `headers:`, plus every trait in `is:` on the resource and the method, the resource's `type:`, and `securedBy`, each taken from `shared.txt` with its `<<parameters>>` filled in |
| Required? | RAML 1.0: required unless `required: false` or the name ends in `?`. RAML 0.8: optional unless `required: true` |
| Values | The declaration's `example`, else `default`, else its first `enum` value, else a synthetic value that fits the type (`Bearer munit-test-token`, a fixed UUID). Never copy a real token, secret or `secure::` value |
| Query and URI parameters | `queryParameters:` and `uriParameters:` on the method and resource. Send the required ones; send nothing undeclared when the `apikit:config` sets `queryParamsStrictValidation` or `headersStrictValidation` |
| Body | The type under `body: <media-type>: type:` (required properties, enums, patterns) and its `example`/`examples` (follow `!include`; a named example may wrap its content in `value:`) |
| Success status | The 2xx key under `responses:` |

An unresolved trait, library or include blocks that endpoint: report it, and never send a
reduced header set instead.

## Step 5 — Plan the tests (`FILE` scope)

🧪 Planning tests...

List every processor of the unit and save the list as `WORK_DIR/coverage-<file>.txt`: one row
per processor, with its flow, file and `doc:id`, and the test that covers it. Loggers are not
listed.

| `API_TYPE` | The unit's processors |
|---|---|
| `HTTP_APIKIT` | Every flow and sub-flow of the file, with same-file sub-flows expanded. Flows in other files run in these tests but are counted in their own file's coverage |
| `HTTP_LISTENER` | A file with a listener flow: every flow in the listener's call tree (Step 2), in every file. Any other file: its own flows and the flows they call, reached by requests through the listener. The error handlers those flows use are listed too, each row marked `ERRORS` (they are tested in `ERRORS` scope) |

The table below applies to every flow in the unit: a `choice` in a sub-flow three calls deep
needs its own tests, like one in the listener flow.

| Construct | Tests |
|---|---|
| Straight path | 1 success test that runs every unconditional processor |
| `choice` | 1 test per `when`, plus 1 for the default route, even when `otherwise` is not written |
| `raise-error` | 1 test per raise site, driven by request data or mock results that meet its condition |
| Backend call whose failure is handled differently | 1 test with the mock returning an error |
| `foreach`, `parallel-foreach` | An empty collection, and a non-empty one that runs the body and each `choice` inside it |
| `scatter-gather` | All routes succeed; each route failing, when the outcome differs |
| `until-successful` | Success, and retries exhausted |
| `try` | Success, and each of its on-error branches (they belong to the flow) |
| `async`, batch job | The work inside it, with a 10-second sleep after the request (see Step 8) |

> ⚠️ **`parallel-foreach` result ordering is non-deterministic.** Never assert on a specific
> element position or use `equalTo` on the full output array. Use `haveSize(n)` to verify the
> count, or `eachItem(matcher)` to check every element independently.

> ℹ️ **`choice` with no `otherwise`.** When none of the `when` conditions match, the payload
> and vars pass through unchanged from the last set operation before the `choice`. The default-
> route test asserts that payload and vars equal those pre-`choice` values.

Merge scenarios that run exactly the same processors with the same outcome, but never merge two
routes of one `choice`. A processor that no test can reach is reported with the reason, never
dropped. Print `→ <file>: <n> tests, <covered>/<total> processors`, and on `HTTP_LISTENER` add
`across <k> flows in <m> files`.

**Framing each request:** on `HTTP_APIKIT`, start from the contract example and add the fields the
scripts read (Step 4); on `HTTP_LISTENER`, start from the listener ledger. Check it has every
required property. A branch variant changes only the field that selects the branch. Pick mock
results so that each `choice` takes the route the test is for, and so that the response carries a
value only that route produces: the Assert expression checks that value.

**Raise-error tests** prove the raise and nothing more; the rest of what the handler builds is
tested only in `ERRORS` scope. Drive the raise for real (never mock `raise-error`). Read the
handler branch that catches the raise's type and assert the status it returns; when the response
body carries the error description, also assert it equals the raise's `description`. If the
handler that catches it makes backend calls, mock them so the test stays offline.

## Step 6 — Plan the tests (`APIKIT` scope, `HTTP_APIKIT` only)

🛡️ Planning APIkit error tests...

An `HTTP_LISTENER` API has no router, so there is no `APIKIT` scope: `all` skips it, and errors
the listener flow raises are tested in `ERRORS` scope.

1. **Find the handler** that applies to the listener flow: its own `<error-handler>`, an
   `<error-handler ref="…">`, or the global `defaultErrorHandler-ref`.
2. **Tests.** One test per `APIKIT:*` type that the handler's `on-error-*` branches declare (a
   branch with several types gets one test per type), plus one for its catch-all branch (`ANY` or
   no `type`), injecting `MULE:EXPRESSION`.
3. **No handler declares APIkit types**, or there is no handler at all: still write the suite,
   with one test for each of `APIKIT:BAD_REQUEST`, `APIKIT:NOT_FOUND`,
   `APIKIT:METHOD_NOT_ALLOWED`, `APIKIT:NOT_ACCEPTABLE`, `APIKIT:UNSUPPORTED_MEDIA_TYPE` and
   `APIKIT:NOT_IMPLEMENTED`. If a handler catches them (a catch-all branch), expect what that
   branch builds. If nothing catches them, expect the status from the listener's `error-response`
   `statusCode` with no vars set: 500 for `#[vars.httpStatus default 500]`, and 500 when there
   is no expression.
4. **Request.** Each test mocks the router to raise the error, so the contract is not needed. Use
   one operation from the router (a bodyless `GET` if there is one), enable the main flow and that
   operation's flow, and send only the headers the listener flow reads before the router, with
   synthetic values.
5. **Assertion.** One Assert expression on the status and the body fields that the branch builds
   (read the branch and the response DataWeave it uses).

Test name: `apikit-<type>-test`, for example `apikit-bad-request-test`.

## Step 7 — Plan the tests (`ERRORS` scope)

🚨 Planning error handler tests...

1. **Map flows to handlers.** An error raised in flow F is caught by F's own `<error-handler>`
   (inline or `ref`); when F has none, by the global `defaultErrorHandler-ref`. A sub-flow has no
   handler of its own; it uses its calling flow's.
2. **One test per branch** that the API's flows reach, skipping `APIKIT:*` branches (they belong
   to `APIKIT`; on `HTTP_LISTENER` nothing raises them, so report them as unreachable). Use the
   flow that reaches the branch with the fewest mocks.
3. **Trigger the error for real.** Drive a `raise-error` whose type matches the branch, or make the
   first backend call on the path return an error of that type. For a catch-all branch, use the
   error the connector really raises (`HTTP:INTERNAL_SERVER_ERROR`, `DB:CONNECTIVITY`, …). For a
   branch type that nothing raises, make a backend call on a path that reaches the handler return
   that type.
4. **Assertion.** One Assert expression on the status the branch sets (`on-error-continue` returns
   through the listener's `response`, and `on-error-propagate` through its `error-response`) and
   the body fields the branch's DataWeave builds. Mock every backend call inside the handler.

Test name: `<handler-name>-<error-type>-test`, for example
`main-error-handler-app-input-error-test`.
Reuse the `in/` request files written for `FILE` scope where they fit.

## Step 8 — Write the files

✍️ Writing fixtures and test suites...

### `src/test/resources/test-config.xml`

One file per project. If it already declares `testHttpRequestConfig`, leave it unchanged; if the
file exists without it, add only that element.

```xml
<?xml version="1.0" encoding="UTF-8"?>
<mule xmlns="http://www.mulesoft.org/schema/mule/core"
      xmlns:http="http://www.mulesoft.org/schema/mule/http"
      xmlns:doc="http://www.mulesoft.org/schema/mule/documentation"
      xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
      xsi:schemaLocation="http://www.mulesoft.org/schema/mule/core http://www.mulesoft.org/schema/mule/core/current/mule.xsd
                          http://www.mulesoft.org/schema/mule/http http://www.mulesoft.org/schema/mule/http/current/mule-http.xsd">

    <http:request-config name="testHttpRequestConfig" doc:name="HTTP Request configuration" doc:id="<new UUID>" basePath="<BASE_PATH>">
        <http:request-connection host="<HOST>" port="<PORT>"/>
    </http:request-config>
</mule>
```

| Value | Rule |
|---|---|
| `<PORT>` | The same property the listener's `http:listener-config` uses for `port` (for example `${http.port}`), or its literal |
| `<HOST>` | A property whose value is `localhost` (for example `${http.host}`); search the property files for `localhost` once. If there is none, `localhost`. Never `0.0.0.0` |
| `<BASE_PATH>` | The listener-config `basePath`, if any, plus the static part of the listener `path` without `/*`. For `path="${api.path}"`, look up that one key |

**Several listeners** on different ports, configs or paths (two API versions, a health endpoint):
declare one request config per listener flow, `testHttpRequestConfig` for the first and
`testHttpRequestConfig-<listener flow>` for each other one, and point every test at the config of
the listener it calls. Leave existing configs unchanged.

Add nothing else: MUnit already loads the application's configuration and properties.

### Fixtures

| Folder | Holds | Name |
|---|---|---|
| `src/test/resources/in/` | Request bodies | `<flow>-<scenario>-request.json` |
| `src/test/resources/out/` | Mock results of backend calls, shaped by what the next script reads | `<flow>-<call>-<scenario>-response.json` |

Use the real media type and extension. Bodyless methods have no `in/` file. Replace any
real-looking personal data with obvious synthetic values. Create only the files the tests use.

### Suite

**Naming (MuleSoft conventions)**

| Item | Rule | Example |
|---|---|---|
| Suite file | `src/test/munit/<config-file>-test-suite.xml` — the name of the **Mule XML configuration file**, never a flow name | `orders.xml` → `orders-test-suite.xml` |
| `munit:config name` | The suite file name **including the `.xml` extension** | `name="orders-test-suite.xml"` |
| Test `name` | `<flow-name>-<scenario>-test`, fully lower-kebab-case, unique in the suite | `orders-flow-missing-customer-test` |
| Test `description` | `<flow> \| <METHOD /path> \| <scenario> \| expects <result>` | `orders-flow \| POST /orders \| missing customer \| expects raise-error` |
| `doc:name` | What the step does | `Mock Get customer`, `POST /orders`, `Assert status 201` |

> ⚠️ **Suite name = config XML file name, never a flow name.**
> If the config file is `payment-advice.xml`, the suite MUST be
> `payment-advice-test-suite.xml`. A suite named
> `getPaymentAdviceByIdImplFlow-test-suite.xml` is **wrong**.

> ⚠️ **Test names are always fully lower-kebab-case.**
> camelCase or PascalCase flow names are converted:
> `getPaymentAdviceByIdImplFlow` → `get-payment-advice-by-id-impl-flow`.
> So the correct test name is
> `get-payment-advice-by-id-impl-flow-sap-unavailable-test`, not
> `getPaymentAdviceByIdImplFlow-sap-unavailable-test`.

`<flow-name>` in the test `name` is the implementation flow, not the APIkit operation flow.
Characters such as `:` `\` `/` become `-` in names only; every `flow-ref`,
`enable-flow-source` and mock value keeps the production name exactly.

**Shell**

```xml
<?xml version="1.0" encoding="UTF-8"?>
<mule xmlns="http://www.mulesoft.org/schema/mule/core"
      xmlns:munit="http://www.mulesoft.org/schema/mule/munit"
      xmlns:munit-tools="http://www.mulesoft.org/schema/mule/munit-tools"
      xmlns:http="http://www.mulesoft.org/schema/mule/http"
      xmlns:doc="http://www.mulesoft.org/schema/mule/documentation"
      xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
      xsi:schemaLocation="http://www.mulesoft.org/schema/mule/core http://www.mulesoft.org/schema/mule/core/current/mule.xsd
                          http://www.mulesoft.org/schema/mule/munit http://www.mulesoft.org/schema/mule/munit/current/mule-munit.xsd
                          http://www.mulesoft.org/schema/mule/munit-tools http://www.mulesoft.org/schema/mule/munit-tools/current/mule-munit-tools.xsd
                          http://www.mulesoft.org/schema/mule/http http://www.mulesoft.org/schema/mule/http/current/mule-http.xsd">

    <munit:config name="<suite-file-name>.xml"/>
    <import file="test-config.xml" doc:name="Import test-config.xml"/>

    <!-- tests -->
</mule>
```

**HTTP test**

```xml
<munit:test name="<flow>-<scenario>-test"
            description="<flow> | <METHOD> <path> | <scenario> | expects <result>">
    <munit:enable-flow-sources>
        <munit:enable-flow-source value="<main flow name>"/>
        <munit:enable-flow-source value="<operation flow name, e.g. post:\orders:application\json:orders-config>"/>
    </munit:enable-flow-sources>
    <munit:behavior>
        <munit-tools:mock-when processor="http:request" doc:name="Mock <production doc:name>">
            <munit-tools:with-attributes>
                <munit-tools:with-attribute attributeName="doc:id" whereValue="<production doc:id>"/>
            </munit-tools:with-attributes>
            <munit-tools:then-return>
                <munit-tools:payload value="#[MunitTools::getResourceAsString('out/<flow>-<call>-<scenario>-response.json')]"
                                     mediaType="application/json"/>
            </munit-tools:then-return>
        </munit-tools:mock-when>
    </munit:behavior>
    <munit:execution>
        <munit:set-event doc:name="Set request body">
            <munit:payload value="#[MunitTools::getResourceAsString('in/<flow>-<scenario>-request.json')]"
                           mediaType="application/json" encoding="UTF-8"/>
        </munit:set-event>
        <http:request config-ref="testHttpRequestConfig" method="<METHOD>" path="<path>" doc:name="<METHOD> <path>">
            <http:body><![CDATA[#[payload]]]></http:body>
            <http:headers><![CDATA[#[{
    "Content-Type": "application/json",
    "<header>": "<value>"
}]]]></http:headers>
            <http:response-validator>
                <http:success-status-code-validator values="100..599"/>
            </http:response-validator>
        </http:request>
    </munit:execution>
    <munit:validation>
        <munit-tools:assert doc:name="Assert status <code> and response">
            <munit-tools:that><![CDATA[#[import * from dw::test::Asserts
---
{ status: attributes.statusCode, <field>: payload.<field> }
    must equalTo({ status: <code>, <field>: "<value>" })]]]></munit-tools:that>
        </munit-tools:assert>
    </munit:validation>
</munit:test>
```

> ⚠️ **MUnit 3.x: `timeout` and `timeoutUnit` are NOT valid attributes on `munit:test`.**
> They existed in MUnit 2.x but were removed from the `munit:test` XSD schema in MUnit 3.x.
> The schema validator rejects them at parse time, preventing the entire suite from loading.
> Never add `timeout` or `timeoutUnit` to `munit:test` — omit them entirely for all tests
> including async/batch.

**Enabled flows.** On an APIkit API, enable the main flow (the one with the `http:listener` and the
`apikit:router`) and the operation flow the request reaches, both with their exact production
names. On `HTTP_LISTENER`, enable the listener flow only.

**Request body (set-event) and mock responses.** Always use `MunitTools::getResourceAsString`
for both — never `readUrl('classpath://...')`. The path is relative to `src/test/resources`
with no `classpath://` prefix:

- `set-event` payload: `#[MunitTools::getResourceAsString('in/<flow>-<scenario>-request.json')]`
- `mock-when` `then-return` payload: `#[MunitTools::getResourceAsString('out/<flow>-<call>-<scenario>-response.json')]`

A bodyless method has no `set-event` and no `http:body`.

**Assert expression.** Validation holds one `munit-tools:assert` and nothing else: no
`assert-that`, no `verify-call` (the only exception is `async` and batch work, see Rules). Put the
status and the response fields that prove the route into one object and compare it with
`must equalTo({ … })`, taking the expected values from Step 3 and the response DataWeave. When a
field has no fixed value (a generated id, a timestamp), use the list form instead, with `$`
standing for the value before `must`:

```text
{ status: attributes.statusCode, body: payload } must [
    $.status must equalTo(201),
    $.body.id must notBeNull(),
    $.body.status must equalTo("CREATED")
]
```

Matchers from `dw::test::Asserts`: `equalTo(x)`, `notBeNull()`, `beNull()`, `beEmpty()`,
`haveSize(n)`, `haveKey("k")`, `contain("x")`, `beOneOf([…])`, `beGreaterThan(n)`,
`startWith("x")`, `eachItem(matcher)`.

**Building blocks**

```xml
<!-- HTTP backend call returns an error -->
<munit-tools:then-return><munit-tools:error typeId="HTTP:INTERNAL_SERVER_ERROR"/></munit-tools:then-return>

<!-- Flow reads attributes of the call (statusCode, headers): add after payload -->
<munit-tools:attributes value="#[{ statusCode: 200, headers: {} }]"/>

<!-- DB: select (returns array of objects) -->
<munit-tools:mock-when processor="db:select" doc:name="Mock <production doc:name>">
    <munit-tools:with-attributes>
        <munit-tools:with-attribute attributeName="doc:id" whereValue="<production doc:id>"/>
    </munit-tools:with-attributes>
    <munit-tools:then-return>
        <munit-tools:payload value="#[MunitTools::getResourceAsString('out/<flow>-<call>-<scenario>-response.json')]"
                             mediaType="application/json"/>
    </munit-tools:then-return>
</munit-tools:mock-when>

<!-- DB: insert / update / delete (void or affected-row count — omit then-return for void) -->
<munit-tools:mock-when processor="db:insert" doc:name="Mock <production doc:name>">
    <munit-tools:with-attributes>
        <munit-tools:with-attribute attributeName="doc:id" whereValue="<production doc:id>"/>
    </munit-tools:with-attributes>
</munit-tools:mock-when>

<!-- DB returns an error -->
<munit-tools:then-return><munit-tools:error typeId="DB:CONNECTIVITY"/></munit-tools:then-return>

<!-- Object Store: retrieve (returns a stored value) -->
<munit-tools:mock-when processor="os:retrieve" doc:name="Mock OS Retrieve">
    <munit-tools:with-attributes>
        <munit-tools:with-attribute attributeName="doc:id" whereValue="<production doc:id>"/>
    </munit-tools:with-attributes>
    <munit-tools:then-return>
        <munit-tools:payload value="#[MunitTools::getResourceAsString('out/<flow>-<call>-<scenario>-response.json')]"
                             mediaType="application/json"/>
    </munit-tools:then-return>
</munit-tools:mock-when>

<!-- Object Store: contains (returns Boolean) -->
<munit-tools:mock-when processor="os:contains" doc:name="Mock OS Contains">
    <munit-tools:with-attributes>
        <munit-tools:with-attribute attributeName="doc:id" whereValue="<production doc:id>"/>
    </munit-tools:with-attributes>
    <munit-tools:then-return>
        <munit-tools:payload value="#[true]" mediaType="application/java"/>
    </munit-tools:then-return>
</munit-tools:mock-when>

<!-- Object Store: store / remove / clear (void — no then-return) -->
<munit-tools:mock-when processor="os:store" doc:name="Mock OS Store">
    <munit-tools:with-attributes>
        <munit-tools:with-attribute attributeName="doc:id" whereValue="<production doc:id>"/>
    </munit-tools:with-attributes>
</munit-tools:mock-when>

<!-- Object Store key not found error -->
<munit-tools:then-return><munit-tools:error typeId="OS:KEY_NOT_FOUND"/></munit-tools:then-return>

<!-- Salesforce: query (returns SObject list) -->
<munit-tools:mock-when processor="salesforce:query" doc:name="Mock SF Query">
    <munit-tools:with-attributes>
        <munit-tools:with-attribute attributeName="doc:id" whereValue="<production doc:id>"/>
    </munit-tools:with-attributes>
    <munit-tools:then-return>
        <munit-tools:payload value="#[MunitTools::getResourceAsString('out/<flow>-<call>-<scenario>-response.json')]"
                             mediaType="application/json"/>
    </munit-tools:then-return>
</munit-tools:mock-when>

<!-- Salesforce: create / update (returns SaveResult with id) -->
<munit-tools:mock-when processor="salesforce:create" doc:name="Mock SF Create">
    <munit-tools:with-attributes>
        <munit-tools:with-attribute attributeName="doc:id" whereValue="<production doc:id>"/>
    </munit-tools:with-attributes>
    <munit-tools:then-return>
        <munit-tools:payload value='#[[{ "id": "0010000000TEST001", "success": true }]]'
                             mediaType="application/json"/>
    </munit-tools:then-return>
</munit-tools:mock-when>

<!-- Operation with target="x" puts its result in vars.x -->
<munit-tools:then-return>
    <munit-tools:variables><munit-tools:variable key="x" value="#[MunitTools::getResourceAsString('out/<flow>-<call>-<scenario>-response.json')]" mediaType="application/json"/></munit-tools:variables>
</munit-tools:then-return>

<!-- Connector with no doc:id: match by config-ref -->
<munit-tools:with-attributes>
    <munit-tools:with-attribute attributeName="config-ref" whereValue="<connector config name>"/>
</munit-tools:with-attributes>

<!-- APIKIT scope: router raises BAD_REQUEST -->
<munit-tools:mock-when processor="apikit:router" doc:name="Mock APIkit router - APIKIT:BAD_REQUEST">
    <munit-tools:with-attributes>
        <munit-tools:with-attribute attributeName="config-ref" whereValue="<apikit config name>"/>
    </munit-tools:with-attributes>
    <munit-tools:then-return><munit-tools:error typeId="APIKIT:BAD_REQUEST"/></munit-tools:then-return>
</munit-tools:mock-when>

<!-- APIKIT scope: router raises NOT_FOUND -->
<munit-tools:then-return><munit-tools:error typeId="APIKIT:NOT_FOUND"/></munit-tools:then-return>

<!-- APIKIT scope: router raises METHOD_NOT_ALLOWED -->
<munit-tools:then-return><munit-tools:error typeId="APIKIT:METHOD_NOT_ALLOWED"/></munit-tools:then-return>

<!-- APIKIT scope: router raises NOT_ACCEPTABLE -->
<munit-tools:then-return><munit-tools:error typeId="APIKIT:NOT_ACCEPTABLE"/></munit-tools:then-return>

<!-- APIKIT scope: router raises UNSUPPORTED_MEDIA_TYPE -->
<munit-tools:then-return><munit-tools:error typeId="APIKIT:UNSUPPORTED_MEDIA_TYPE"/></munit-tools:then-return>

<!-- APIKIT scope: router raises NOT_IMPLEMENTED -->
<munit-tools:then-return><munit-tools:error typeId="APIKIT:NOT_IMPLEMENTED"/></munit-tools:then-return>

<!-- Raise-error test validation -->
<munit-tools:assert doc:name="Assert <raise doc:name> raised">
    <munit-tools:that><![CDATA[#[import * from dw::test::Asserts
---
{ status: attributes.statusCode, message: payload.<description field> }
    must equalTo({ status: <code>, message: "<raise-error description>" })]]]></munit-tools:that>
</munit-tools:assert>

<!-- Expected value held in a property: assert against the key, not a copied value -->
<munit-tools:assert doc:name="Assert status from property">
    <munit-tools:that><![CDATA[#[import * from dw::test::Asserts
---
(attributes.statusCode as String) must equalTo(p('error.badrequest.statusCode'))]]]></munit-tools:that>
</munit-tools:assert>

<!-- async scope or batch job on the path: last step of execution, with extended timeout -->
<munit-tools:sleep time="10" timeUnit="SECONDS" doc:name="Wait for async/batch to finish"/>
```

**Flows with no endpoint of their own.** A file whose flows are called only from other files
(shared sub-flows, for example) cannot be reached by a request without its callers. Test each of
its flows directly: no `enable-flow-sources`; in `munit:execution`, a `munit:set-event` with the
payload (`MunitTools::getResourceAsString`, as above) and the `munit:variables` its callers set,
then `<flow-ref name="<exact flow name>"/>`; in `munit:validation`, one Assert expression on
`payload` and `vars`. On `HTTP_LISTENER`, this applies only to flows outside every listener's
call tree; flows inside it are covered through the listener's requests.

**Rules**

- Enable the main flow and the operation flow on `HTTP_APIKIT` (the listener flow on
  `HTTP_LISTENER`), and only in tests that send a request.
- Mock every connector operation on the path by `doc:id` (`config-ref` when a `doc:id` is
  missing), including those inside flows called from other files. Never mock a `flow-ref`.
- Never mock loggers, and never mock `ee:transform`, `raise-error`, `choice` or other scopes,
  error handlers, or the flow under test.
- Validation is one `munit-tools:assert` (Assert expression). Never use `assert-that` or
  `assert-equals`, and use `verify-call` only for work inside an `async` scope or batch job
  (one `verify-call atLeast="1"` on a call inside it, after the sleep).
- Every test proves its route: its Assert expression checks the status and a response value that
  only that route produces. When two routes return the same response, say so in
  `coverage-<file>.txt` instead of adding a `verify-call`.
- Assert specific values (status and named fields), never `notBeNull()` alone.
- No mocks needed: omit `<munit:behavior>` (an empty one fails validation). Test elements need no
  `doc:id`.
- Child order: `munit:test` → `enable-flow-sources`, `behavior`, `execution`, `validation`;
  `http:request` → `body`, `headers`, `uri-params`, `query-params`, `response-validator`;
  `then-return` → `payload`, `attributes`, `error`, `variables`.
- **Never add `munit-tools:with-payload` as a direct child of `munit-tools:mock-when`.**
  The `mock-when` XSD in MUnit 3.x only allows `with-attributes`, `then-return`, and `then-call`
  as children. `with-payload` at that level causes a schema validation failure at deployment.
  Mock isolation is already guaranteed per-test: each `munit:test` registers its own mocks in
  isolation, so differentiating mocks by payload content inside `mock-when` is neither needed
  nor supported.

## Step 9 — Check and report

✅ Verifying output...

Re-read every file you wrote and confirm:

- [ ] Only MUnit files were written to the project (`pom.xml` only if MUnit was missing).
- [ ] `HTTP_APIKIT`: the RAML contract was read from `<cache>/spec/`, libraries are in
      `spec/exchange_modules/`, and `source.txt` was written after the rest of the cache.
      `HTTP_LISTENER`: no contract was read.
- [ ] Every row of each `coverage-<file>.txt` names a test, or is reported with its reason.
- [ ] Every mocked `doc:id` exists in the production XML, every connector operation on each path
      is mocked, and no `flow-ref` is mocked.
- [ ] Every request test enables the correct flows with production names.
- [ ] Every `set-event` payload uses `MunitTools::getResourceAsString('in/…')` with `mediaType`
      and `encoding="UTF-8"`. Never use `readUrl('classpath://in/…')`.
- [ ] Every `mock-when` `then-return` payload uses `MunitTools::getResourceAsString('out/…')`
      with `mediaType`. Never use `readUrl('classpath://out/…', '…')`.
- [ ] No `timeout` or `timeoutUnit` attributes appear on any `munit:test` element (they are invalid in MUnit 3.x and cause a parse-time deployment failure).
- [ ] Each test's validation is a single `munit-tools:assert`; no `assert-that`; `verify-call`
      only for async/batch.
- [ ] Every `in/` and `out/` file referenced exists, parses, and holds every field the scripts read.
- [ ] Suite file name = config XML file name (never a flow name); `munit:config name` includes
      `.xml`; test names are fully lower-kebab-case.
- [ ] XML is well formed (`xmllint --noout <file>`, or `[xml](Get-Content <file>)` in PowerShell).

To run (only when the user asks):
`mvn test -Dmunit.test=<suite>.xml -Dmule.env=<env>`  
Add the secure-properties key if the API uses one, and `-Danypoint.platform.gatekeeper=disabled`
when the API declares `api-gateway:autodiscovery`. Never edit `pom.xml` to make a run pass.
**Do not execute this command.** Print it and stop.

---

📊 **MUnit Generation Summary**

| Suite | Tests | Processors | Fixtures | Notes |
|---|---|---|---|---|
| `<config-file>-test-suite.xml` | `<n>` | `<covered>/<total>` | `in/<n>, out/<n>` | `<blockers or —>` |
| `error-handler-test-suite.xml` | `<n>` | `<covered>/<total>` | `in/<n>, out/<n>` | `<blockers or —>` |

⚠️ Blockers: `<list>` or none  
ℹ️ Tests were **NOT run**. To run: `mvn test -Dmunit.test=<suite>.xml -Dmule.env=<env>`
