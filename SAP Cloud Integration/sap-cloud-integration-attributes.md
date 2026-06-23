# SAP Cloud Integration — Attributes Reference

A working reference to the **attributes** you read and write inside a Cloud Integration iFlow — Headers, Properties, Body, Attachments — plus the predefined Camel and `SAP_*` ones the runtime hands you, how to access each, and the scenarios where each actually earns its place.

> **Grounding note.** Cloud Integration is **Apache Camel** under the hood. "Headers" and "Properties" are Camel message headers and exchange properties; most predefined attributes are Camel constants (`Camel*`) or SAP additions (`SAP_*`). Camel semantics are stable; the `SAP_*` set and a few runtime-config labels drift between Edge/Cloud builds and tenant versions. **Verify version-sensitive items against current SAP Help and your tenant's behaviour.** No live network was used to build this.

---

## Table of contents

1. [The message model: the four containers](#1-the-message-model-the-four-containers)
2. [How to access attributes](#2-how-to-access-attributes)
3. [Predefined Camel headers](#3-predefined-camel-headers)
4. [Predefined SAP_ headers & properties](#4-predefined-sap_-headers--properties)
5. [Pattern-context attributes (Splitter / Aggregator / Loop / Multicast)](#5-pattern-context-attributes-splitter--aggregator--loop--multicast)
6. [Error-handling attributes](#6-error-handling-attributes)
7. [The gotchas that bite](#7-the-gotchas-that-bite)
8. [Scenario cookbook (variations)](#8-scenario-cookbook-variations)

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

*Reference compiled for SAP Cloud Integration (Integration Suite). Camel attribute semantics are stable; the SAP_* set, ProcessDirect property propagation, and runtime-config labels are version-dependent — confirm against current SAP Help and your tenant before relying on any item in production.*
