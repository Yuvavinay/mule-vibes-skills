# Non-HTTP MUnit skill

Loaded by the [dispatcher](./SKILL.md) for APIs with no HTTP Listener: scheduler, Anypoint MQ,
JMS, VM, file, SFTP, Kafka, Salesforce and other sources, batch jobs, and flows with no source.
If you arrive without the dispatcher's handoff block, run the dispatcher first.

> ⛔ **Never run tests.** Never execute `mvn test` or trigger a test run at any point. If the
> user asks to run, print the command and stop.

Every test runs the flow directly with `flow-ref`. Flow sources stay off in MUnit, so no
scheduler fires and no message is consumed. Only the connector operations (the backend calls)
are mocked; the flow logic, and every flow it calls through `flow-ref`, runs for real. There is
no contract and no status code: **the DataWeave is the source of truth** for the input, the mock
responses and the expected results, and validation is **one Assert expression** per test. The
`set-event` payload is loaded with `MunitTools::getResourceAsString`. No `test-config.xml` is
needed or created.

The dispatcher's hard rules apply: the project receives MUnit files only (plus `pom.xml` only
if MUnit is missing), and all scratch work goes to `WORK_DIR`. In a mixed API, the HTTP skill
has already run: re-read any suite it wrote and append to it; never overwrite it or repeat its
scenarios. Print one emoji progress line per step.

---

## Step 1 — Set the scope and read the unit

📂 Setting scope...

| Request | Scope | Suite written |
|---|---|---|
| `<file>.xml` | `FILE`: every processor in every flow and sub-flow of that file | `<file>-test-suite.xml` |
| `errors` | `ERRORS`: every error-handler branch the API's flows route errors to | `error-handler-test-suite.xml` |
| `all` | `FILE` for every file that declares a `flow` or `sub-flow` (except files whose flows run only inside error handlers), then `ERRORS` | All of the above |

Resolve `<file>` against `WORK_DIR/index.txt`: relative path first, then basename. If none
matches, list the closest names; if several match, ask. List `src/test/munit/` once; if a suite
you are about to write exists, read it and add only the scenarios it does not cover yet.

🔍 Reading flows and entry points...

Use the index for flows, callers and handlers; do not search the project again. Then read:

1. **The target file in full.** Expand every `flow-ref` to a flow or sub-flow in the **same
   file**: it is part of the unit. Follow every `flow-ref` to **another file** as well, and read
   the called flow in full, then the flows it calls, recursively: it runs for real in the test,
   so its connector operations must be mocked and its DataWeave must not fail.
2. **Entry flows.** Each flow or sub-flow of the file that no other flow in the same file calls is
   an entry flow; tests call it with `flow-ref`. Same-file flows it calls run inside its tests.
3. **The source** of each entry flow, if it has one. It tells you which `attributes` the flow can
   read:

   | Source | Key `attributes` fields |
   |---|---|
   | `anypoint-mq:subscriber` | `ackToken`, `properties` (map of message properties) |
   | `file:listener`, `sftp:listener`, `ftp:on-new-or-updated-file` | `fileName`, `path`, `size`, `timestamp` |
   | `jms:listener` | `headers.correlationId`, `headers.replyTo`, `properties` (map) |
   | `vm:listener` | No standard attributes; message body only |
   | `kafka:message-listener` | `key`, `offset`, `partition`, `topic` |
   | `salesforce:modified-object-listener`, `salesforce:new-object-listener` | `recordId`, `changeType`, `entityName` |
   | `scheduler` | No attributes (no `in/` file needed) |
   | `wsc:consumer` | SOAP-specific envelope attributes |

4. **Callers in other files**, for entry flows without a source: only the steps that set the
   `vars` or the `payload` the entry flow reads. The test recreates them in `set-event`.

> ⚠️ **Every connector operation on every path MUST be mocked** — including `os:retrieve`,
> `os:contains`, `db:select`, `salesforce:query` and every other operation that returns a value
> or can raise an error. An un-mocked connector operation will attempt a real connection and fail.

Record every **backend call** on the path, with its `doc:id`, `doc:name` and the fields the next
processor reads from its result: `http:request`, `db:*`, `salesforce:*`, `os:*`,
`anypoint-mq:*`, `jms:*`, `vm:*`, `email:*`, `file:*`, `sftp:*`, `ftp:*`, `wsc:consume`,
`java:*` and any other connector operation, in the unit and in every flow it calls through
`flow-ref`, in any file. A `flow-ref` is never a backend call: the flow it calls runs, and the
connector operations inside it are mocked. Note: `vm:consume` used synchronously inside a flow
(not as a source) is a backend call and must be mocked.

## Step 2 — Analyse the DataWeave

⚙️ Analysing DataWeave scripts...

Read every DWL file the path uses (`resource="…dwl"`), including those of flows called in other
files, and every inline script in `ee:transform`, `set-variable`, `set-payload` and `choice`
`when` expressions. Save the notes as `WORK_DIR/dwl/<file>.txt`, and reuse them when another
file in the same run uses the same script. For each script, note:

- **Inputs:** each `payload.x`, `vars.y`, `attributes.z` and `p('key')` it reads, with the type
  and format it expects.
- **What breaks it:** coercions (`as Number`, `as Date {format: …}`); string functions (`lower`,
  `upper`, `replace`, `splitBy`, `trim`) and `++` on a field that could be null; index access
  (`payload[0].x` needs an array); `sizeOf`, `map`, `filter` and `groupBy` on a value that must
  be an array.
- **Property keys built from data:** for `p('prefix.' ++ payload.x)`, search the property files
  for `prefix.` once, and use a value of `x` whose key exists.
- **Output:** the fields it produces, and their values for a given input. These are the values
  the tests assert.

The notes decide every fixture:

| A field the scripts read is … | It goes in |
|---|---|
| In the incoming message body | `in/` file → `munit:payload` |
| Set by the source (queue properties, file name, …) | `munit:attributes` in `set-event` |
| Set by a caller before the entry flow | `munit:variables` in `set-event` |
| Returned by a backend call | `out/` file → that call's mock |
| Computed inside the unit | Nowhere: the flow computes it |

Shape each `out/` response from what the **next** script or `choice` reads, not from what the flow
finally outputs. When a field selects a branch, set it to the value that selects the branch the
test is for. Every fixture must hold every input its scripts read, in the right type and format,
so that no DataWeave fails in a test.

**Watermark flows.** When a flow uses Object Store to track a watermark (reads a timestamp with
`os:retrieve`, uses it to filter, then stores the new mark with `os:store`):

- Mock `os:retrieve` to return a synthetic past timestamp for "subsequent run" tests.
- Mock `os:store` as a void operation (no `then-return`).
- For the "first run" scenario, mock `os:retrieve` to raise `OS:KEY_NOT_FOUND`, and assert that
  the flow handles the missing watermark gracefully (falls back to a default or initial value).

## Step 3 — Plan the tests

🧪 Planning tests...

List every processor of the unit, including the expanded same-file flows, and save the list as
`WORK_DIR/coverage-<file>.txt` with the test that covers each row. Loggers are not listed. Flows
called in other files run in these tests but are counted in their own file's coverage.

| Construct | Tests |
|---|---|
| Straight path | 1 success test that runs every unconditional processor |
| `choice` | 1 test per `when`, plus 1 for the default route, even when `otherwise` is not written |
| `raise-error` | 1 test per raise site, driven by input or mock results that meet its condition |
| Backend call whose failure is handled differently | 1 test with the mock returning an error |
| `foreach`, `parallel-foreach` | An empty collection, and a non-empty one that runs the body and each `choice` inside it |
| `scatter-gather` | All routes succeed; each route failing, when the outcome differs |
| `until-successful` | Success, and retries exhausted |
| `try` | Success, and each of its on-error branches (they belong to the flow) |
| `async` | The work inside the scope, with a 10-second sleep after the `flow-ref` |
| Batch job | Each `batch:step` (records that meet its `acceptExpression`), each `batch:aggregator`, and `batch:on-complete`, with a 10-second sleep after the `flow-ref` |

> ⚠️ **`parallel-foreach` result ordering is non-deterministic.** Never assert on a specific
> element position or use `equalTo` on the full output array. Use `haveSize(n)` to verify the
> count, or `eachItem(matcher)` to check every element independently.

> ℹ️ **`choice` with no `otherwise`.** When none of the `when` conditions match, the payload
> and vars pass through unchanged from the last set operation before the `choice`. The default-
> route test asserts that payload and vars equal those pre-`choice` values.

Merge scenarios that run exactly the same processors with the same outcome, but never merge two
routes of one `choice`. A processor that no test can reach is reported with the reason, never
dropped. Print `→ <file>: <n> tests, <covered>/<total> processors`.

- **Raise-error tests** prove the raise and nothing more; handler output is tested only in
  `ERRORS` scope. Drive the raise for real (never mock `raise-error`). If the error leaves the
  flow, catch it with the test-local `try` (Step 4) and assert the caught error type and
  description; if the flow's own handler continues, assert a value that handler sets. If the
  handler makes backend calls, mock them.
- **Batch:** a step with `acceptPolicy="ONLY_FAILURES"` needs a record that failed in an earlier
  step, so make that step's mock return an error. With `maxFailedRecords="0"` (the default) the
  job stops at the first failure; if that makes a step unreachable, report it.

**`ERRORS` scope**

1. **Map flows to handlers.** An error raised in flow F is caught by F's own `<error-handler>`
   (inline or `ref`); when F has none, by the global `defaultErrorHandler-ref`. A sub-flow has no
   handler of its own; it uses its calling flow's.
2. **One test per branch** that the API's flows reach. Use the flow that reaches the branch with
   the fewest mocks.
3. **Trigger the error for real.** Drive a `raise-error` whose type matches the branch, or make the
   first backend call on the path return an error of that type. For a catch-all branch, use the
   error the connector really raises (`ANYPOINT-MQ:CONNECTIVITY`, `DB:CONNECTIVITY`,
   `OS:KEY_NOT_FOUND`, …).
4. **Assert what the branch builds**, in one Assert expression. `on-error-continue` returns
   normally from the `flow-ref`: assert the payload and vars. `on-error-propagate` raises out of
   the `flow-ref`: catch it with the test-local `try`, then assert the error type and the payload
   and vars the branch set. Mock every backend call inside the handler (a DLQ publish, a
   notification).

Test name: `<handler-name>-<error-type>-test`, for example
`main-error-handler-mq-connectivity-test`.

## Step 4 — Write the files

✍️ Writing fixtures and test suites...

### Fixtures

| Folder | Holds | Name |
|---|---|---|
| `src/test/resources/in/` | The incoming message body (message, file content, record list) | `<flow>-<scenario>-input.<ext>` |
| `src/test/resources/out/` | Mock results of backend calls, shaped by Step 2 | `<flow>-<call>-<scenario>-response.<ext>` |

Use the real format, extension and `mediaType` (`.json`, `.xml`, `.csv`, `.txt`). A
scheduler-driven flow usually has no `in/` file. Replace any real-looking personal data with
obvious synthetic values. Create only the files the tests use.

### Suite

**Naming (MuleSoft conventions)**

| Item | Rule | Example |
|---|---|---|
| Suite file | `src/test/munit/<config-file>-test-suite.xml` — the name of the **Mule XML configuration file**, never a flow name | `order-sync.xml` → `order-sync-test-suite.xml` |
| `munit:config name` | The suite file name **including the `.xml` extension** | `name="order-sync-test-suite.xml"` |
| Test `name` | `<flow-name>-<scenario>-test`, fully lower-kebab-case, unique in the suite | `order-sync-flow-no-new-orders-test` |
| Test `description` | `<flow> \| <source, or "direct"> \| <scenario> \| expects <result>` | `order-sync-flow \| scheduler \| no new orders \| expects no publish` |
| `doc:name` | What the step does | `Mock Select new orders`, `Run order-sync-flow`, `Assert order count` |

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

Characters such as `:` `\` `/` become `-` in names only; every `flow-ref` and mock value keeps
the production name exactly.

**Shell** (no `test-config.xml` import)

```xml
<?xml version="1.0" encoding="UTF-8"?>
<mule xmlns="http://www.mulesoft.org/schema/mule/core"
      xmlns:munit="http://www.mulesoft.org/schema/mule/munit"
      xmlns:munit-tools="http://www.mulesoft.org/schema/mule/munit-tools"
      xmlns:doc="http://www.mulesoft.org/schema/mule/documentation"
      xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
      xsi:schemaLocation="http://www.mulesoft.org/schema/mule/core http://www.mulesoft.org/schema/mule/core/current/mule.xsd
                          http://www.mulesoft.org/schema/mule/munit http://www.mulesoft.org/schema/mule/munit/current/mule-munit.xsd
                          http://www.mulesoft.org/schema/mule/munit-tools http://www.mulesoft.org/schema/mule/munit-tools/current/mule-munit-tools.xsd">

    <munit:config name="<suite-file-name>.xml"/>

    <!-- tests -->
</mule>
```

**Test**

```xml
<munit:test name="<flow>-<scenario>-test"
            description="<flow> | <source or direct> | <scenario> | expects <result>"
            timeout="30000" timeoutUnit="MILLISECONDS">
    <munit:behavior>
        <munit-tools:mock-when processor="<connector>:<operation>" doc:name="Mock <production doc:name>">
            <munit-tools:with-attributes>
                <munit-tools:with-attribute attributeName="doc:id" whereValue="<production doc:id>"/>
            </munit-tools:with-attributes>
            <munit-tools:then-return>
                <munit-tools:payload value="#[readUrl('classpath://out/<flow>-<call>-<scenario>-response.json', 'application/json')]"
                                     mediaType="application/json"/>
            </munit-tools:then-return>
        </munit-tools:mock-when>
    </munit:behavior>
    <munit:execution>
        <munit:set-event doc:name="Set payload and vars">
            <munit:payload value="#[MunitTools::getResourceAsString('in/<flow>-<scenario>-input.json')]"
                           mediaType="application/json" encoding="UTF-8"/>
            <munit:variables>
                <munit:variable key="<varName>" value='#["<value>"]' mediaType="application/java"/>
            </munit:variables>
            <munit:attributes value="#[{ <attrKey>: '<attrValue>' }]" mediaType="application/java"/>
        </munit:set-event>
        <flow-ref name="<exact production flow name>" doc:name="Run <flow>"/>
    </munit:execution>
    <munit:validation>
        <munit-tools:assert doc:name="Assert <scenario> result">
            <munit-tools:that><![CDATA[#[import * from dw::test::Asserts
---
{ result: vars.<outputVar> }
    must equalTo({ result: "<expectedValue>" })]]]></munit-tools:that>
        </munit-tools:assert>
    </munit:validation>
</munit:test>
```

For async/batch tests that include a sleep, set `timeout="60000" timeoutUnit="MILLISECONDS"`.

**Payload.** The `set-event` payload is always
`#[MunitTools::getResourceAsString('in/<file>')]` with the fixture's `mediaType` and
`encoding="UTF-8"`; the path is relative to `src/test/resources`, with no `classpath://`. A
scheduler-triggered flow has no `in/` file: omit `munit:payload`. Omit `munit:variables` and
`munit:attributes` when the flow does not read them.

**Assert expression.** Validation holds one `munit-tools:assert` and nothing else: no
`assert-that`, no `verify-call` of any kind — including async scopes and batch jobs. Put the
output fields that prove the route into one object and compare it with `must equalTo({ … })`,
taking the expected values from Step 2 (DataWeave notes). When a field has no fixed value (a
generated id, a timestamp), use the list form instead, with `$` standing for the value before
`must`:

```text
{ processed: vars.processedCount, status: vars.jobStatus } must [
    $.processed must beGreaterThan(0),
    $.status must equalTo("COMPLETED")
]
```

Matchers from `dw::test::Asserts`: `equalTo(x)`, `notBeNull()`, `beNull()`, `beEmpty()`,
`haveSize(n)`, `haveKey("k")`, `contain("x")`, `beOneOf([…])`, `beGreaterThan(n)`,
`startWith("x")`, `eachItem(matcher)`.

**Building blocks**

```xml
<!-- DB: select (returns array of objects) -->
<munit-tools:mock-when processor="db:select" doc:name="Mock <production doc:name>">
    <munit-tools:with-attributes>
        <munit-tools:with-attribute attributeName="doc:id" whereValue="<production doc:id>"/>
    </munit-tools:with-attributes>
    <munit-tools:then-return>
        <munit-tools:payload value="#[readUrl('classpath://out/<flow>-<call>-<scenario>-response.json', 'application/json')]"
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
        <munit-tools:payload value="#[readUrl('classpath://out/<flow>-<call>-<scenario>-response.json', 'application/json')]"
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
        <munit-tools:payload value="#[readUrl('classpath://out/<flow>-<call>-<scenario>-response.json', 'application/json')]"
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
    <munit-tools:variables><munit-tools:variable key="x" value="#[readUrl('classpath://out/…json', 'application/json')]" mediaType="application/json"/></munit-tools:variables>
</munit-tools:then-return>

<!-- Connector with no doc:id: match by config-ref -->
<munit-tools:with-attributes>
    <munit-tools:with-attribute attributeName="config-ref" whereValue="<connector config name>"/>
</munit-tools:with-attributes>

<!-- Raise-error test: catch the propagated error with a test-local try -->
<try doc:name="Catch expected error">
    <flow-ref name="<exact production flow name>" doc:name="Run <flow>"/>
    <error-handler>
        <on-error-continue type="<NAMESPACE>:<TYPE>" doc:name="Catch <error type>">
            <set-variable variableName="caughtError" value="#[error]" doc:name="Capture error"/>
        </on-error-continue>
    </error-handler>
</try>
<!-- then assert: -->
<munit-tools:assert doc:name="Assert error type raised">
    <munit-tools:that><![CDATA[#[import * from dw::test::Asserts
---
vars.caughtError.errorType.identifier
    must equalTo("<TYPE>")]]]></munit-tools:that>
</munit-tools:assert>

<!-- Expected value held in a property: assert against the key, not a copied value -->
<munit-tools:assert doc:name="Assert value from property">
    <munit-tools:that><![CDATA[#[import * from dw::test::Asserts
---
(vars.resultCode as String) must equalTo(p('result.success.code'))]]]></munit-tools:that>
</munit-tools:assert>

<!-- async scope or batch job on the path: last step of execution, with extended timeout.
     After the sleep, assert on a variable or payload value set inside the async/batch scope.
     Never use verify-call — not even for async/batch work. -->
<munit-tools:sleep time="10" timeUnit="SECONDS" doc:name="Wait for async/batch to finish"/>
```

**Flows with no source of their own.** A sub-flow (or a flow with no source) is called only
through `flow-ref`. Test it directly: no source to enable; in `munit:execution`, a
`munit:set-event` with the payload (`MunitTools::getResourceAsString`, as above) and the
`munit:variables` its callers set, then `<flow-ref name="<exact flow name>"/>`;
in `munit:validation`, one Assert expression on `payload` and `vars`.

**Rules**

- Never enable flow sources (no `munit:enable-flow-sources`): non-HTTP flows run by `flow-ref`
  only; their source never fires in a test.
- Mock every connector operation on the path by `doc:id` (`config-ref` when a `doc:id` is
  missing), including those inside flows called from other files. Never mock a `flow-ref`.
- Never mock loggers, and never mock `ee:transform`, `raise-error`, `choice` or other scopes,
  error handlers, or the flow under test.
- Validation is one `munit-tools:assert` (Assert expression). **Never use `assert-that`,
  `assert-equals`, or `verify-call` under any circumstances — not even for `async` scopes or
  batch jobs.** For async/batch, assert on a variable or payload value the scope sets (after
  the sleep); if two routes produce the same output, document the reason in `coverage-<file>.txt`.
- Every test proves its route: its Assert expression checks a value that only that route produces.
  When two routes produce the same output, say so in `coverage-<file>.txt` instead of asserting
  on the same field again.
- Assert specific values (named fields), never `notBeNull()` alone.
- No mocks needed: omit `<munit:behavior>` entirely (an empty one fails validation). Test
  elements need no `doc:id`.
- Child order: `munit:test` → `behavior`, `execution`, `validation`;
  `set-event` → `payload`, `variables`, `attributes`;
  `then-return` → `payload`, `attributes`, `error`, `variables`.

## Step 5 — Check and report

✅ Verifying output...

Re-read every file you wrote and confirm:

- [ ] Only MUnit files were written to the project (`pom.xml` only if MUnit was missing).
- [ ] No `test-config.xml` was created or modified.
- [ ] Every row of each `coverage-<file>.txt` names a test, or is reported with its reason.
- [ ] Every mocked `doc:id` exists in the production XML, every connector operation on each path
      is mocked, and no `flow-ref` is mocked.
- [ ] No `munit:enable-flow-sources` element appears anywhere in the suite.
- [ ] Every `set-event` payload uses `MunitTools::getResourceAsString('in/…')` with `mediaType`
      and `encoding="UTF-8"`. Scheduler flows omit `munit:payload`.
- [ ] Each test has `timeout` set; async/batch tests use `60000 MILLISECONDS`.
- [ ] Each test's validation is a single `munit-tools:assert`; no `assert-that`; no `verify-call`
      of any kind (not even for async/batch — assert on a value the scope sets instead).
- [ ] Every `in/` and `out/` file referenced exists, parses, and holds every field the scripts
      read.
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
