# SAP Cloud Integration — Attributes & Flow Steps Reference

A working reference in two parts. **Part I** covers the **attributes** you read and write inside a Cloud Integration iFlow — Headers, Properties, Body, Attachments — plus the predefined Camel and `SAP_*` ones the runtime hands you. **Part II** covers the **flow steps** (the editor palette) — every transformer, call, routing, security, persistence, and validation element you drop onto the canvas — with variants and a scenario for each.

> **Grounding note.** Cloud Integration is **Apache Camel** under the hood. "Headers" and "Properties" are Camel message headers and exchange properties; most predefined attributes are Camel constants (`Camel*`) or SAP additions (`SAP_*`). Camel semantics are stable; the `SAP_*` set and a few runtime-config labels drift between Edge/Cloud builds and tenant versions. **Verify version-sensitive items against current SAP Help and your tenant's behaviour.** No live network was used to build this.

---

## Table of contents

**Part I — Attributes**
1. [The message model: the four containers](#1-the-message-model-the-four-containers)
2. [How to access attributes](#2-how-to-access-attributes)
3. [Predefined Camel headers](#3-predefined-camel-headers)
4. [Predefined SAP_ headers & properties](#4-predefined-sap_-headers--properties)
5. [Pattern-context attributes (Splitter / Aggregator / Loop / Multicast)](#5-pattern-context-attributes-splitter--aggregator--loop--multicast)
6. [Error-handling attributes](#6-error-handling-attributes)
7. [The gotchas that bite](#7-the-gotchas-that-bite)
8. [Scenario cookbook (variations)](#8-scenario-cookbook-variations)

**Part II — Flow Steps (palette)**
9. [How the palette is organised](#9-how-the-palette-is-organised)
10. [Process & participants](#10-process--participants)
11. [Events & flow control](#11-events--flow-control)
12. [Message transformers](#12-message-transformers)
13. [Call activities](#13-call-activities)
14. [Routing](#14-routing)
15. [Security elements](#15-security-elements)
16. [Persistence](#16-persistence)
17. [Validators](#17-validators)
18. [Step-selection cheat sheet](#18-step-selection-cheat-sheet)

---

## 1. The message model: the four containers

Everything in an iFlow rides on a Camel **Exchange**. Inside it you have four places to put data. Picking the wrong one is the root of most "why didn't my value survive / why did it leak to the receiver" bugs.

| Container | Scope / lifetime | Travels to receiver? | Typical use |
|---|---|---|---|
| **Body (payload)** | The message content; one "current" body, replaced by each transformation step | Yes — it *is* what's sent | The actual data being integrated |
| **Header** | Travels *with the message* through steps; can be promoted to a transport header (HTTP header, mail header, etc.) on the outbound call | **Only if explicitly allowed** (see Allowed Headers gotcha) | Values that may need to leave the iFlow as transport metadata; dynamic adapter config |
| **Property (Exchange Property)** | Lives on the Exchange for the whole iFlow run | **Never** — purely iFlow-internal | Internal state, stashed payloads, control flags, anything you don't want leaking |
| **Attachment** | Files/parts attached to the message | Adapter-dependent | MPL attachments for monitoring; multipart payloads |

**The one-line rule:** if a value must potentially leave the iFlow as transport metadata → **Header**. If it's internal bookkeeping → **Property**. When in doubt, Property — it's the safe default because it can't leak and doesn't bloat outbound calls.

---

## 2. How to access attributes

Four mechanisms, used in different places.

### 2.1 Camel Simple expressions

The expression language used in Content Modifier, Router conditions, Filter, dynamic adapter fields, etc.

```text
${header.MyHeader}            <!-- read a header -->
${property.MyProperty}        <!-- read a property -->
${in.body}  or  ${body}       <!-- the current payload -->
${exchangeId}                 <!-- unique Camel exchange id -->
${camelId}                    <!-- runtime/Camel context id -->
${date:now:yyyy-MM-dd'T'HH:mm:ss}          <!-- current timestamp, formatted -->
${date:header.MyDateHeader:yyyyMMdd}       <!-- format a date held in a header -->
${property.CamelSplitIndex}                <!-- a predefined property -->
```

| Token | What it resolves to |
|---|---|
| `${header.X}` / `${in.header.X}` | Value of header `X`. Case-sensitive. |
| `${property.X}` | Value of exchange property `X`. |
| `${body}` / `${in.body}` | Current message body. |
| `${date:now:FORMAT}` | Current time in the given Java `SimpleDateFormat`. |
| `${date:header.X:FORMAT}` | A date value in header `X`, reformatted. |
| `${exchangeId}` | The Exchange's unique id — handy for tracing/log correlation. |
| `${bean:beanName.method}` | Invoke a bean method (advanced). |

### 2.2 Content Modifier

The no-code workhorse for setting attributes. Three tabs, each writing to one container:

- **Message Header** tab → writes **Headers**
- **Exchange Property** tab → writes **Properties**
- **Message Body** tab → replaces the **Body**

Each header/property row has:

| Field | What it does |
|---|---|
| **Name** | The attribute name to create/overwrite. |
| **Source Type** | Where the value comes from: `Constant`, `Header`, `Property`, `XPath`, `Expression` (Camel Simple), `Local Variable`, `External Parameter`, `Number Range`. This selector is the whole point of the Content Modifier — it lets you move values *between* containers (e.g. copy a header into a property). |
| **Source Value** | The literal / reference / expression / XPath itself. |
| **Data Type** | The Java type to cast to, e.g. `java.lang.String`, `java.lang.Integer`. Matters when a downstream step or adapter is type-sensitive (numbers, booleans). |

> XPath source type needs the body to be XML; for JSON, convert first or use a script.

### 2.3 Groovy / JavaScript script

Full programmatic access. The standard Groovy signature:

```groovy
import com.sap.gateway.ip.core.customdev.util.Message
import java.util.HashMap

def Message processData(Message message) {

    // --- BODY ---
    def body = message.getBody(java.lang.String)          // read as String
    message.setBody(body.toUpperCase())                   // replace body

    // --- HEADERS ---
    def headers = message.getHeaders()                    // Map of all headers
    def h = headers.get("MyHeader")                       // read one
    message.setHeader("NewHeader", "value")               // write one

    // --- PROPERTIES ---
    def props = message.getProperties()                   // Map of all properties
    def p = props.get("MyProperty")                       // read one
    message.setProperty("NewProperty", "value")           // write one

    // --- ATTACHMENTS ---
    def attachments = message.getAttachments()            // Map of attachments

    return message
}
```

| Call | What it does |
|---|---|
| `message.getBody(Class)` | Read the body as a given type (`java.lang.String`, `byte[]`, `java.io.Reader`, …). Reading as String consumes streams — read once. |
| `message.setBody(obj)` | Replace the body. |
| `message.getHeaders()` / `getHeader(name, Class)` | Read all / one header. |
| `message.setHeader(name, value)` | Create/overwrite a header. |
| `message.getProperties()` / `getProperty(name)` | Read all / one property. |
| `message.setProperty(name, value)` | Create/overwrite a property. |
| `message.getAttachments()` / `addAttachmentObject(...)` | Read / add attachments. |

### 2.4 XPath / JSONPath

Used in Content Modifier (XPath source type), Router/Filter conditions, and Mapping. XPath needs XML; declare namespaces in the iFlow's runtime configuration if your payload is namespaced, or the expressions silently return empty.

---

## 3. Predefined Camel headers

The runtime sets and reads these — knowing them saves you reinventing behaviour. Set them to *control* an adapter; read them to *react* to one.

| Attribute | Container | What it does | Scenario |
|---|---|---|---|
| `CamelHttpResponseCode` | Header | The HTTP status returned to the sender | Set to `201`/`400`/`202` to control what an HTTPS/SOAP/OData/ProcessDirect sender call returns |
| `CamelHttpResponseText` | Header | The response reason text | Pair with a custom status code |
| `CamelHttpMethod` | Header | HTTP verb on an outbound HTTP/HTTPS receiver call | Drive GET vs POST dynamically |
| `CamelHttpUri` | Header | Full target URI for an HTTP receiver | Fully dynamic endpoint (overrides the configured address) |
| `CamelHttpPath` | Header | Path appended to the configured address | Dynamic path while keeping the base host fixed |
| `CamelHttpQuery` | Header | Query string for the outbound call | Build query params at runtime |
| `Content-Type` | Header | MIME type of the body | Force `application/json` before a receiver that's picky about it |
| `CamelCharsetName` | Header | Character encoding of the body | Fix encoding before writing a file |
| `CamelFileName` | Header | Target file name for File/SFTP receiver; also set by the sender on poll | Dynamic filenames (see cookbook) |
| `CamelFileNameOnly` | Header | File name without path | Read just the name of a polled file |
| `CamelFilePath` | Header | Full path of a polled file | Routing/archiving by source path |
| `CamelFileLength` | Header | Size of a polled file | Skip/branch on empty or oversized files |
| `CamelFileLastModified` | Header | Last-modified timestamp of a polled file | Ordering / freshness checks |

> The exact set of `Camel*` headers honoured depends on the adapter. When a dynamic adapter field offers `${header.X}`, that's your hook — the adapter documents which headers it reads.

---

## 4. Predefined SAP_ headers & properties

SAP's additions — mostly about **monitoring (MPL)**, **correlation**, and **what shows up in Message Monitoring**. Getting these right is the difference between a traceable landscape and a forensic nightmare at 2 a.m.

| Attribute | Container | What it does | Scenario |
|---|---|---|---|
| `SAP_ApplicationID` | Header | Populates the searchable **Application Message ID** in Message Monitoring | Make a message findable by a business key (order no., invoice no.) |
| `SAP_Sender` | Header | Overrides the **Sender** shown in monitoring | Label messages by logical sender when the technical one is generic |
| `SAP_Receiver` | Header | Overrides the **Receiver** shown in monitoring | Same, for receiver-side labelling |
| `SAP_MessageType` | Header | Sets the **message type** attribute in monitoring | Categorise traffic for filtering |
| `SAP_MessageProcessingLogID` | Property | The current **MPL GUID** | Log it / pass it downstream to correlate across systems |
| `SAP_MplCorrelationId` | Property | **Correlation ID** linking related MPLs | Group a parent iFlow and its ProcessDirect children under one trace |
| `SAP_MessageProcessingLogCustomStatus` | Property | A **custom status** string shown on the MPL | Surface a business outcome ("DUPLICATE", "SKIPPED") in monitoring |

> The `SAP_*` set is the most version-sensitive part of this doc — names get added/renamed across releases. Treat the monitoring-facing ones (`SAP_ApplicationID`, `SAP_Sender`, `SAP_Receiver`, `SAP_MessageType`, custom status) as the durable, documented ones, and confirm anything exotic against current SAP Help before relying on it.

---

## 5. Pattern-context attributes (Splitter / Aggregator / Loop / Multicast)

When you're inside an iterating pattern, the runtime exposes context as properties. These are how per-iteration logic knows where it is.

### Splitter

| Attribute | Container | What it does |
|---|---|---|
| `CamelSplitIndex` | Property | **0-based** index of the current split iteration |
| `CamelSplitSize` | Property | Total number of splits — **only set if the splitter could determine size up front** (not in streaming mode) |
| `CamelSplitComplete` | Property | `true` on the **last** iteration only |

```text
${property.CamelSplitIndex}        <!-- e.g. for a 1-based line number: index+1 -->
```

> `CamelSplitSize` being absent in streaming splits is a classic trap — don't build logic that assumes it's always populated. If you need the count, use a non-streaming/general splitter or count beforehand.

### Aggregator / Gather

| Attribute | Container | What it does |
|---|---|---|
| `CamelAggregatedSize` | Property | Number of messages aggregated so far (where exposed) |

Aggregator correlation and completion are configured on the step (correlation expression, completion condition/timeout) rather than read as attributes — but the **correlation key** is typically a header/property you set upstream.

### Looping (Looping Process Call)

| Attribute | Container | What it does |
|---|---|---|
| `CamelLoopIndex` | Property | Current iteration index of a loop (where exposed) |

> Loop control in CPI is usually an **expression condition** evaluated each pass (e.g. `${property.hasMore} = 'true'`) — you set the property yourself inside the loop body. Don't rely on a built-in loop counter being present; manage your own where the pattern doesn't expose one.

### Multicast

Multicast branches each get a **copy** of the exchange. Properties/headers set in one branch don't bleed into siblings; results recombine at the **Gather/Join**. Design state-passing around that isolation.

---

## 6. Error-handling attributes

When a step throws and control passes to an **Exception Subprocess**, the runtime stashes the failure for you to inspect.

| Attribute | Container | What it does |
|---|---|---|
| `CamelExceptionCaught` | Property | The **exception object** that was thrown — the whole Java `Throwable` |
| `SAP_MessageProcessingLogID` | Property | Still available — log it with the error |

Reading it in a script inside the exception subprocess:

```groovy
import com.sap.gateway.ip.core.customdev.util.Message
def Message processData(Message message) {
    def ex = message.getProperty("CamelExceptionCaught")
    def msg   = ex?.getMessage()                    // the error text
    def cls   = ex?.getClass()?.getName()           // the exception class
    // build a fault payload / set a custom status
    message.setProperty("SAP_MessageProcessingLogCustomStatus", "FAILED: " + cls)
    message.setBody("Error: " + msg)
    return message
}
```

In a Content Modifier, the message text is reachable via the expression:

```text
${exception.message}        <!-- the caught exception's message -->
```

> **Body in an exception subprocess** is the payload *as it was when the error hit* — which may be mid-transformation, not your original. If you need the original payload for a fault response, **stash it in a property at the start of the flow** (see cookbook I) so it's intact when the error handler runs.

---

## 7. The gotchas that bite

- **Headers do NOT reach the receiver by default.** A header is only promoted to a transport header (e.g. an outbound HTTP header) if its name is listed in the iFlow's runtime configuration **"Allowed Header(s)"** (comma-separated, wildcards allowed). Forget this and your carefully-set `Authorization` or custom header silently never leaves. Properties *never* leave, full stop.
- **Case sensitivity.** `${header.MyHeader}` ≠ `${header.myheader}`. HTTP transport may normalise header casing on the wire, but inside the exchange the names are exact. Mismatched case = empty value, no error.
- **Header bloat / dropped headers.** Some adapters cap or drop large/numerous headers. Don't shovel big payloads or many control values into headers — that's what properties are for.
- **`CamelSplitSize` may be absent** in streaming splits (see §5). Guard against null.
- **ProcessDirect propagation has nuance.** A ProcessDirect call passes body and headers to the called iFlow; **whether all exchange properties propagate has varied across versions** — don't assume a property set before a ProcessDirect call is readable on the other side. If you need a value to cross reliably, pass it as a header (and remember Allowed Headers if it must then go further), or re-establish it. **Verify current behaviour in your tenant.**
- **Reading a streamed body consumes it.** `getBody(java.lang.String)` on a stream reads it to the end; a later step finds it empty. Read once, or set the body back, or enable the iFlow's body reuse where needed.
- **XPath needs namespaces declared.** Namespaced XML + undeclared prefix in runtime config = silent empty result.
- **Content Modifier Data Type matters.** A value typed `java.lang.String` won't compare as a number downstream. Set the type when the consumer cares (numeric ranges, booleans).

---

## 8. Scenario cookbook (variations)

Concrete patterns, each naming the attribute(s) that do the work.

### A. Make a message searchable by a business key in monitoring
Set a header `SAP_ApplicationID` to the business identifier. It then appears as the **Application Message ID** in Message Monitoring and is searchable.
```text
Content Modifier → Message Header
  Name: SAP_ApplicationID
  Source Type: XPath (or Expression)
  Source Value: //Order/OrderNumber
```

### B. Return a custom HTTP status to the sender
Before the integration flow ends, set the response code header.
```text
Content Modifier → Message Header
  Name: CamelHttpResponseCode
  Source Type: Constant
  Source Value: 202
  Data Type: java.lang.Integer
```
Variation: set it dynamically from a property your logic computed (`Source Type: Property`).

### C. Dynamic SFTP/file name with timestamp + business key
Set `CamelFileName` from an expression combining a date and a header.
```text
Content Modifier → Message Header
  Name: CamelFileName
  Source Type: Expression
  Source Value: INV_${header.SAP_ApplicationID}_${date:now:yyyyMMdd_HHmmss}.xml
```
Variation: append `${property.CamelSplitIndex}` when writing one file per split.

### D. Per-iteration logic inside a splitter
Inside the split branch, read the split context to number lines or tag the last one.
```text
Line number (1-based):   ${property.CamelSplitIndex} + 1   (compute in script)
Is last item:            ${property.CamelSplitComplete}    → route differently on the final split
```

### E. Branch on a value without losing it
Routers consume nothing — but if you transform the body before the decision, stash the deciding value in a **property** first so it survives the transformation.
```text
1. Content Modifier → Exchange Property  routeKey = //Doc/Type   (XPath)
2. (transform body freely)
3. Router condition: ${property.routeKey} = 'INVOICE'
```

### F. Build a fault response from a caught exception
In the exception subprocess, read `CamelExceptionCaught` and shape a clean error to return.
```groovy
def ex = message.getProperty("CamelExceptionCaught")
message.setHeader("CamelHttpResponseCode", 500)
message.setBody('{"error":"' + (ex?.getMessage() ?: 'unknown') + '"}')
```

### G. Forward one specific header to the receiver
Set the header, then **allow** it through.
```text
1. Content Modifier → Message Header  X-Correlation-Id = ${exchangeId}
2. iFlow Runtime Configuration → Allowed Header(s): X-Correlation-Id
```
Without step 2 the header never leaves the iFlow.

### H. Carry a correlation ID across a ProcessDirect chain
Because property propagation across ProcessDirect is version-dependent, pass the correlation value as a **header** and allow it; have each downstream iFlow read it back into a property/MPL.
```text
Parent:  set header  X-Corr = ${property.SAP_MplCorrelationId}  + allow it
Child:   read ${header.X-Corr} → set property / log to MPL
```
(Confirm in your tenant whether properties already propagate; if they do, you can skip the header hop.)

### I. Preserve the original payload while transforming
Stash the inbound body in a property at the very start, so error handlers (or later steps) can recover it intact.
```text
Content Modifier (first step) → Exchange Property
  Name: originalPayload
  Source Type: Expression
  Source Value: ${in.body}
```
Later / in the exception subprocess: `message.getProperty("originalPayload")`.

### J. Force content type / encoding before a picky receiver
```text
Content Modifier → Message Header
  Name: Content-Type      Source: Constant   application/json
  Name: CamelCharsetName  Source: Constant   UTF-8
```

---

# Part II — Flow Steps (palette)

## 9. How the palette is organised

Steps in the iFlow editor are grouped into a handful of palette categories. The sections below follow that grouping. A step is dropped onto the **integration process** (or a local process / exception subprocess) and wired into the message path.

> **Scope note.** This lists the **modelling steps** on the canvas. **Adapters** (HTTPS, SFTP, OData, SOAP, IDoc, AMQP, ProcessDirect, Mail, Kafka, JMS, SuccessFactors, Ariba, …) are configured on the **sender/receiver channels**, not as palette steps, so they're out of scope here — that's a separate reference. The palette evolves between releases; if a step you expect is missing, it's likely newer than this doc — **check the editor.**

The categories: **Message Transformers · Call · Routing · Security · Persistence · Validator**, plus the structural **Process/Participants** and **Events**.

---

## 10. Process & participants

The structural containers — the pools and the start/end of a process.

| Step | What it does | Scenario / note |
|---|---|---|
| **Sender** | The participant that triggers the iFlow; carries an inbound adapter (channel) | An app POSTing to an HTTPS endpoint; an SFTP poll |
| **Receiver** | A downstream participant the iFlow calls; carries an outbound adapter | The backend OData service, the target SFTP |
| **Integration Process** | The main pool holding the message path | Every iFlow has one |
| **Local Integration Process** | A reusable sub-process called via Process Call / Looping Process Call | Modularise; create a transaction or error-handling scope |
| **Exception Subprocess** | Catches errors raised in its parent process | Build fault responses, set custom MPL status, alerting (see Part I §6) |

> A **Local Integration Process** is also how you scope a **transaction** or a tighter exception boundary — wrap the risky steps, call it, and handle failure locally without aborting the whole flow.

---

## 11. Events & flow control

How a process starts and the ways it can end.

| Step | What it does | Scenario / note |
|---|---|---|
| **Start Message** | Entry point of a process, fed by a sender | Standard request-driven start |
| **End Message** | Normal completion; returns the message to the caller (sync) or ends it (async) | End of the happy path |
| **Start Timer / Timer Start Event** | Starts the iFlow **on a schedule** with no sender | Nightly batch poll; periodic data push — there's no inbound message, you build one |
| **Error Start Event** | Entry point of an **Exception Subprocess** | Where the caught exception lands |
| **Terminate Message / End Event** | Stops processing at this point | Deliberately halt after a business condition |
| **Error End Event** | Ends the process **in error** — marks the MPL as Failed | Force a failure status when a business rule says "this must fail" |
| **Escalation End Event** | Ends a branch with an escalation (used in some exception patterns) | Signal a non-fatal escalation |

---

## 12. Message transformers

The steps that change the message — its content, format, or encoding.

| Step | Variants / sub-types | What it does | Scenario |
|---|---|---|---|
| **Content Modifier** | — | Set Headers / Properties / Body (Part I §2.2) | Stash a value, set a response code, build a small payload |
| **Message Mapping** | — | Graphical field-to-field mapping with functions, value mapping, user-defined functions | Map a source schema to a target schema |
| **Operation Mapping** | — | Wraps message mappings (PI/PO migration artifact) | Reuse an imported PI operation mapping |
| **XSLT Mapping** | — | Transform XML via an XSLT stylesheet | Complex structural transforms beyond graphical mapping |
| **ID Mapping** | — | Map/track message IDs between systems | Cross-system ID correlation |
| **Converter** | CSV↔XML, JSON↔XML, XML↔EDI, (and more) | Convert between data formats | Flat-file CSV from a bank → XML for mapping; JSON API ↔ XML backend |
| **Filter** | — | Extract a node-set from XML via **XPath** and make it the new body | Strip an envelope down to the business node |
| **Script** | Groovy, JavaScript | Arbitrary code over the message (Part I §2.3) | Anything declarative steps can't express — custom logic, dynamic headers, exception shaping |
| **Encoder** | Base64 Encode, GZIP Compress, ZIP Compress, MIME Multipart Encode | Encode/compress/package the body | Base64 a binary for a JSON field; gzip before upload |
| **Decoder** | Base64 Decode, GZIP Decompress, ZIP Decompress, MIME Multipart Decode | The inverse | Decode an incoming Base64 blob; unzip a polled archive |
| **Message Digest** | — | Compute a hash digest of the message (for signing/integrity) | Pre-compute a digest used by a downstream signature |

> Mapping vs Script vs XSLT, quick call: **graphical mapping** for clear field-to-field with reuse; **XSLT** for heavy structural reshaping; **Groovy** for logic, conditionals, and anything touching headers/properties. Don't reach for Groovy when a mapping would be clearer to the next person who maintains it.

---

## 13. Call activities

Steps that call *out* (to a receiver) or call *in* (to a local process). This is the category people most often pick wrong.

| Step | Sync? | What it does | Scenario |
|---|---|---|---|
| **Request Reply** | Synchronous | Send to a receiver and **wait for the response**, which replaces the body | Call an OData/REST service and use its response |
| **Send** | Asynchronous | Fire-and-forget to a receiver; **no response** expected | Drop a message on a queue (JMS/AMQP); async notify |
| **Content Enricher** | Synchronous | Call a receiver and **combine** its response with the original message (Combine or Enrich strategy) | Look up master data and merge it into the in-flight payload without losing the original |
| **Poll Enrich** | Synchronous | **Poll** a source (e.g. file/SFTP) to fetch and enrich | Pull a reference file mid-flow |
| **Process Call** | — | Invoke a **Local Integration Process** once | Modularise; create an exception/transaction scope |
| **Looping Process Call** | — | Repeatedly invoke a local process **while a condition holds** | Pagination — keep calling an API until `hasMore=false`; chunked processing |

> **Request Reply vs Content Enricher** is the classic mix-up: Request Reply **replaces** the body with the response; Content Enricher **keeps** your original and combines the response into it. If a lookup must not clobber your payload, you want Content Enricher.

> **Looping Process Call** is the right tool for **API pagination** — set a "has more pages" property inside the loop body and let the condition drive it. Don't fake loops with recursion or multiple copied steps.

---

## 14. Routing

Steps that direct or fan out the message path.

| Step | Variants | What it does | Scenario |
|---|---|---|---|
| **Router** | — | Content-based routing: send the message down **one** branch based on a condition (XPath/expression), with a **default** route | Route by document type, country, amount threshold |
| **Multicast** | Parallel, Sequential | Send a **copy** to multiple branches | Send the same order to two backends; produce two formats |
| **Join** | — | Marks where multicast branches **converge** | Pair with Gather after a multicast |
| **Gather** | Combine, Merge strategies | **Combine** messages from multicast/splitter branches into one | Reassemble split results; merge two enrichment responses |
| **Splitter** | General, Iterating, IDoc, EDI, PKCS#7/CMS | Break one message into **many** (by XPath, expression, or format) | Process a batch line-by-line; split an IDoc bundle; split an EDI interchange |
| **Aggregator** | — | **Collect related messages over time** and combine them, using a correlation key + completion condition/timeout | Batch up individual messages arriving over an hour into one file; recombine async split results |

> **Parallel vs Sequential Multicast:** parallel fans out concurrently (faster, but branches are isolated and order isn't guaranteed); sequential runs branches in order (predictable, slower). Remember branches don't share property/header writes (Part I §5).

> **Splitter vs Aggregator** are inverses: splitter is **one→many in one run**; aggregator is **many runs→one**, stateful, and needs a data store. Aggregator is the heavier, trickier one — get the correlation key and completion condition right or messages either never complete or complete too early.

---

## 15. Security elements

Message-level crypto, distinct from transport (TLS) security on the adapters.

| Step | Variants | What it does | Scenario |
|---|---|---|---|
| **Encryptor** | PKCS#7/CMS, PGP | Encrypt the message body for a recipient | Send a PGP-encrypted file to a partner's SFTP |
| **Decryptor** | PKCS#7/CMS, PGP | Decrypt an inbound encrypted body | Decrypt a partner's PGP payload on arrival |
| **Signer** | Simple Signer, PKCS#7/CMS, XML Digital Signer, PGP | Digitally **sign** the message for integrity/non-repudiation | Sign an outbound document a partner will verify |
| **Verifier** | PKCS#7/CMS, XML Signature, PGP | **Verify** an inbound signature | Reject a tampered or unsigned partner message |

> Keys/certs live in the tenant **Keystore** (and PGP keyrings); the step references aliases. Message-level (PGP/CMS) protects the payload end-to-end through intermediaries; transport-level (TLS) only protects the hop. Partner B2B/EDI flows usually need the message-level steps, not just TLS.

---

## 16. Persistence

Steps that store state — for staging, decoupling, dedupe, or audit.

| Step | Operations / variants | What it does | Scenario |
|---|---|---|---|
| **Data Store Operations** | Write, Get, Select, Delete | Read/write entries in a **Data Store** (transient or persistent), keyed by ID | Decouple sender from receiver (write now, process later); stage for retry; async hand-off |
| **Write Variables** | Global, Local | Persist named **variables** across iFlow runs | Remember a high-water mark / last-run timestamp for delta polling |
| **Persist Message** | — | Save a **snapshot** of the message at this point (visible in monitoring, with retention) | Audit/troubleshoot — capture the payload mid-flow without affecting processing |
| **Idempotent Process Call** | — | Use an **idempotent repository** so a message with an already-seen key is processed **once** | Deduplicate — drop replays of the same order ID; exactly-once semantics |

> **Data Store vs Persist Message:** Data Store is *operational* (you write, then later get/select/delete to drive processing); Persist Message is *observational* (a monitoring snapshot you don't read back into the flow). Don't use Persist Message as a queue.

> **Write Variables** is the standard pattern for **delta/incremental polling** — store the last successful timestamp/ID, read it next run, query only newer records.

---

## 17. Validators

| Step | Variants | What it does | Scenario |
|---|---|---|---|
| **XML Validator** | — | Validate the body against an **XSD/WSDL/DTD** schema; fail on non-conformance | Reject malformed inbound XML before it corrupts a mapping |
| **EDI Validator / Extractor** | — | Validate/extract **EDI** interchanges against rules | B2B: enforce partner EDI conformance, raise functional acks on failure |

> Validate **early** — at the boundary, right after the sender — so a bad payload fails fast with a clear error rather than deep in a mapping with a cryptic one.

---

## 18. Step-selection cheat sheet

The decisions that trip people up, in one place:

- **Call a service and use its answer** → Request Reply. **Call it but keep my payload** → Content Enricher. **Don't need an answer** → Send.
- **One message → many** → Splitter. **Many → one, over time** → Aggregator. **Same message → several places at once** → Multicast (+ Gather to recombine).
- **Reusable sub-logic / error scope** → Local Integration Process via Process Call. **Repeat until done (pagination)** → Looping Process Call.
- **Change format** → Converter. **Change structure** → Mapping or XSLT. **Change a header/property/small body** → Content Modifier. **Logic it can't express** → Script.
- **Stage/decouple/retry** → Data Store. **Remember across runs** → Write Variables. **Dedupe** → Idempotent Process Call. **Audit snapshot** → Persist Message.
- **Protect the payload through intermediaries** → Encryptor/Signer (message-level), not just TLS.
- **Reject bad input early** → Validator at the boundary.

---

*Reference compiled for SAP Cloud Integration (Integration Suite). Camel attribute semantics are stable; the SAP_* set, ProcessDirect property propagation, runtime-config labels, and the exact palette/step variants are version-dependent — confirm against current SAP Help and your tenant before relying on any item in production. Adapters (sender/receiver channels) are out of scope here and warrant their own reference.*
