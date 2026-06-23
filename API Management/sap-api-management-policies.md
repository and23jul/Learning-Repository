# SAP API Management — Policy Reference

A working reference to the policies available in SAP API Management (part of Integration Suite), the XML syntax of each, and what every element inside the syntax does.

> **Grounding note.** SAP API Management runs on **Apigee Edge** technology. The policy types, the XML element names, and their semantics are inherited almost 1:1 from Apigee Edge. That's useful: where this doc is thin, the Apigee Edge policy docs apply directly. It's also a caveat — policy *availability*, a few default attributes, and especially the root-element XML namespace drift between SAP releases. **Verify anything version-sensitive against the policy editor's own generated XML or current SAP Help.** No live network was used to build this.

---

## Table of contents

1. [Mental model: how policies attach](#1-mental-model-how-policies-attach)
2. [Elements common to every policy](#2-elements-common-to-every-policy)
3. [Flow variables (the `ref` mechanism)](#3-flow-variables-the-ref-mechanism)
4. [Traffic Management policies](#4-traffic-management-policies)
5. [Security policies](#5-security-policies)
6. [Mediation policies](#6-mediation-policies)
7. [Extension policies](#7-extension-policies)
8. [Practical gotchas](#8-practical-gotchas)

---

## 1. Mental model: how policies attach

A policy is a single XML file. It does **nothing** on its own — it executes only when *attached as a step* to a **flow** inside an API proxy. The four flow attachment points:

| Flow segment | Runs on | Typical policies |
|---|---|---|
| **ProxyEndpoint → PreFlow / Flows / PostFlow** (request) | inbound from the app | Verify API Key, OAuth, Spike Arrest, Quota, threat protection |
| **TargetEndpoint → PreFlow / Flows / PostFlow** (request) | just before hitting backend | Assign Message, Service Callout, Basic Auth (encode) |
| **TargetEndpoint** (response) | after backend replies | XML↔JSON, Extract Variables, Response Cache populate |
| **ProxyEndpoint** (response) | just before replying to app | Assign Message (clean headers), Message Logging |
| **FaultRules / DefaultFaultRule** | on error | Raise Fault, Assign Message |

Steps execute **top to bottom** in the order attached. Order matters — e.g. Spike Arrest before Quota, threat protection before any parsing.

---

## 2. Elements common to every policy

Every policy's **root element** carries the same set of attributes, so they're documented once here and omitted from the per-policy notes below.

```xml
<PolicyType async="false" continueOnError="false" enabled="true" name="My-Policy-1" xmlns="http://www.sap.com/apimgmt">
    <DisplayName>My Policy 1</DisplayName>
    <!-- policy-specific elements -->
</PolicyType>
```

| Attribute / element | What it does |
|---|---|
| `name` | Unique policy ID within the proxy. This is what you reference when attaching the policy as a `<Step>` in a flow. No spaces. |
| `async` | Legacy threading flag. Effectively always `false`; leave it. |
| `continueOnError` | `false` (default) → on policy failure the flow jumps to fault handling. `true` → swallow the error and keep going. Set `true` deliberately, e.g. a logging policy you don't want to break the call. |
| `enabled` | `true` to run, `false` to keep the policy in the bundle but skip it. |
| `xmlns` | The SAP API Management policy namespace. **Release-sensitive** — the editor may generate it with or without; copy whatever your tenant produces. |
| `<DisplayName>` | Human-readable label shown in the UI. Cosmetic only. |

A common element you'll also see across many policies:

| Element | What it does |
|---|---|
| `<IgnoreUnresolvedVariables>` | `true` → if a referenced flow variable doesn't exist, treat it as empty instead of throwing. `false` → unresolved variable raises a fault. Big behavioural lever; set it consciously. |

---

## 3. Flow variables (the `ref` mechanism)

Almost every element accepts a literal value **or** a `ref` to a flow variable. This is the spine of the whole system.

```xml
<APIKey ref="request.queryparam.apikey"/>     <!-- read value from a flow variable -->
<Header name="X-Trace">{messageid}</Header>    <!-- {curly} = inline variable substitution -->
```

Useful built-in variables you'll reference constantly:

| Variable | Holds |
|---|---|
| `request.header.NAME` / `response.header.NAME` | a request/response header |
| `request.queryparam.NAME` | a query parameter |
| `request.formparam.NAME` | a form field |
| `request.verb`, `request.path`, `request.uri` | HTTP verb / path / full URI |
| `request.content`, `response.content` | the message body |
| `message.content` | body of whichever message is in context |
| `client.ip`, `proxy.pathsuffix` | caller IP, path after the basepath |
| `apiproxy.name`, `environment.name`, `organization.name` | runtime identifiers |
| `system.timestamp`, `messageid` | epoch ms, unique transaction id |
| `apikey`, `developer.id`, `developer.email`, `app.name` | populated *after* Verify API Key / OAuth runs |

---

## 4. Traffic Management policies

Protect the backend and enforce commercial limits.

### 4.1 Quota

Enforces a **count of calls allowed over a time interval** (business limit, e.g. 10,000 calls/month per app).

```xml
<Quota name="Quota-1" type="calendar">
    <Allow count="10000" countRef="request.header.allowed_quota"/>
    <Interval ref="request.header.quota_count">1</Interval>
    <TimeUnit ref="request.header.quota_timeout">month</TimeUnit>
    <StartTime>2024-01-01 00:00:00</StartTime>
    <Distributed>true</Distributed>
    <Synchronous>true</Synchronous>
    <AsynchronousConfiguration>
        <SyncIntervalInSeconds>20</SyncIntervalInSeconds>
        <SyncMessageCount>5</SyncMessageCount>
    </AsynchronousConfiguration>
    <Identifier ref="client_id"/>
    <MessageWeight ref="request.header.weight"/>
</Quota>
```

| Element / attribute | What it does |
|---|---|
| `type` | Counter reset behaviour: `calendar` (fixed window from `StartTime`), `rollingwindow` (sliding), `flexi` (window starts on first request). Determines when the count resets to zero. |
| `<Allow count>` | The hard limit. `countRef` overrides `count` from a variable (e.g. plan-based quota per app). |
| `<Interval>` | How many `TimeUnit`s make up one window. `1` + `month` = monthly. |
| `<TimeUnit>` | `second` / `minute` / `hour` / `day` / `month`. |
| `<StartTime>` | Anchor for `calendar` type — when the first window begins. |
| `<Distributed>` | `true` → one shared counter across all message processors (correct for real limits). `false` → per-node counter (multiplies your effective limit by node count — usually a bug). |
| `<Synchronous>` | `true` → counter updated atomically on every call (accurate, slightly slower). `false` → use async config below for throughput. |
| `<AsynchronousConfiguration>` | When async: flush the local counter to the central store every `SyncIntervalInSeconds` **or** every `SyncMessageCount` calls, whichever first. Trades accuracy for speed. |
| `<Identifier>` | What the quota is counted *per*. Usually `client_id` (per app). Omit → single global counter for the whole proxy. |
| `<MessageWeight>` | Lets one call consume more than 1 from the quota (e.g. a bulk endpoint costs 5). |

### 4.2 Spike Arrest

Smooths **traffic rate** — protects the backend from sudden bursts. Not a business quota; a shock absorber.

```xml
<SpikeArrest name="Spike-Arrest-1">
    <Rate>30ps</Rate>
    <Identifier ref="client_id"/>
    <MessageWeight ref="request.header.weight"/>
</SpikeArrest>
```

| Element | What it does |
|---|---|
| `<Rate>` | The cap, as `Nps` (per second) or `Npm` (per minute). **Key behaviour:** it's smoothed, not bucketed — `30ps` ≈ 1 request every ~33 ms, not 30 in a clump at t=0. Two requests 10 ms apart → the second is rejected even though you're "under 30". |
| `<Identifier>` | Apply the rate per app/client (`client_id`) vs globally. |
| `<MessageWeight>` | Weight a call as multiple hits against the rate. |

> Quota vs Spike Arrest vs Concurrent Rate Limit, in one line each: **Quota** = how many over a long window (business), **Spike Arrest** = how fast (per second smoothing), **Concurrent Rate Limit** = how many *in flight at once* to the backend.

### 4.3 Concurrent Rate Limit

Caps **simultaneous open connections** to a target — protects a backend that dies under parallelism, not volume.

```xml
<ConcurrentRatelimit name="Concurrent-Ratelimit-1">
    <AllowConnections count="100" countRef="request.header.count"/>
    <Distributed>true</Distributed>
    <TargetIdentifier name="default">
        <UseConfiguredConnectionLimit>true</UseConfiguredConnectionLimit>
    </TargetIdentifier>
    <Strict>false</Strict>
</ConcurrentRatelimit>
```

| Element | What it does |
|---|---|
| `<AllowConnections count>` | Max concurrent connections; `countRef` to drive it from a variable. |
| `<Distributed>` | Shared count across message processors vs per-node. |
| `<TargetIdentifier>` | Names the TargetEndpoint this limit guards. **Note:** this policy must be attached to the *TargetEndpoint* flow, not the proxy flow. |
| `<Strict>` | `true` → enforce the limit exactly. `false` → allow a small overshoot for throughput. |

### 4.4 Response Cache

Caches a backend response so repeat requests skip the target entirely — latency and load win.

```xml
<ResponseCache name="Response-Cache-1">
    <CacheKey>
        <Prefix/>
        <KeyFragment ref="request.uri" type="string"/>
    </CacheKey>
    <Scope>Exclusive</Scope>
    <ExpirySettings>
        <TimeoutInSec ref="">300</TimeoutInSec>
    </ExpirySettings>
    <SkipCacheLookup>request.header.bypass-cache = "true"</SkipCacheLookup>
    <SkipCachePopulation/>
    <CacheResource>myCache</CacheResource>
    <UseAcceptHeader>false</UseAcceptHeader>
    <UseResponseCacheHeaders>false</UseResponseCacheHeaders>
</ResponseCache>
```

| Element | What it does |
|---|---|
| `<CacheKey>` | Builds the lookup key. `<Prefix>` namespaces it; each `<KeyFragment>` adds a literal or `ref` value. Whatever varies the response (URI, a header, query params) must be in the key or you'll serve stale/wrong data to the wrong caller. |
| `<Scope>` | Prefix scoping: `Exclusive` (this proxy only) up to `Global`. Controls cache-key collisions across proxies. |
| `<ExpirySettings><TimeoutInSec>` | TTL in seconds. Also supports `<ExpiryDate>` / `<TimeOfDay>` for absolute expiry. |
| `<SkipCacheLookup>` | A condition; when true the policy **ignores** the cache and goes to backend (e.g. a force-refresh header). |
| `<SkipCachePopulation>` | A condition; when true the response is **not stored** (e.g. don't cache 5xx). |
| `<CacheResource>` | Named cache store to use. |
| `<UseAcceptHeader>` | `true` → fold the `Accept` header into the key (separate cache per content type). |
| `<UseResponseCacheHeaders>` | `true` → honour backend `Cache-Control`/`Expires` when deciding TTL. |

> **Attach this policy twice:** once in the request flow (lookup) and once in the response flow (populate). Same `name`, two steps.

### 4.5 Reset Quota

Programmatically **adjusts the remaining count** on an existing Quota policy without waiting for the window to roll over.

```xml
<ResetQuota name="Reset-Quota-1">
    <Quota name="Quota-1">
        <Identifier ref="request.header.clientId"/>
        <Allow>90</Allow>
    </Quota>
</ResetQuota>
```

| Element | What it does |
|---|---|
| `<Quota name>` | The **name of the target Quota policy** whose counter you're resetting. |
| `<Identifier>` | Which counter (which client) to reset — must match the Quota's identifier. |
| `<Allow>` | The amount to **reduce the used count by** (i.e. credit back). It restores capacity, it doesn't set an absolute. |

---

## 5. Security policies

Authentication, authorization, and payload-level attack protection.

### 5.1 Verify API Key

Validates an inbound API key and populates app/developer/product variables on success.

```xml
<VerifyAPIKey name="Verify-API-Key-1">
    <APIKey ref="request.queryparam.apikey"/>
</VerifyAPIKey>
```

| Element | What it does |
|---|---|
| `<APIKey ref>` | Where to read the key from (query param, header, etc.). On success it sets `verifyapikey.{policy}.client_id`, `developer.email`, `app.name`, product entitlements — which downstream Quota/OAuth policies rely on. |

### 5.2 OAuth v2.0

The OAuth workhorse — generates and verifies tokens. One policy, many operations.

```xml
<OAuthV2 name="OAuth-v20-1">
    <Operation>VerifyAccessToken</Operation>
    <AccessToken>request.header.authorization</AccessToken>
    <Scope>read write</Scope>
    <ExpiresIn>3600000</ExpiresIn>
    <RefreshTokenExpiresIn>86400000</RefreshTokenExpiresIn>
    <GrantType>request.formparam.grant_type</GrantType>
    <GenerateResponse enabled="true"/>
    <Tokens/>
</OAuthV2>
```

| Element | What it does |
|---|---|
| `<Operation>` | The action. Common values: `GenerateAccessToken`, `RefreshAccessToken`, `VerifyAccessToken`, `InvalidateToken`, `GenerateAuthorizationCode`. This single element switches the policy's entire behaviour. |
| `<AccessToken>` | For `VerifyAccessToken`: where to read the bearer token from. Defaults to the `Authorization` header. |
| `<Scope>` | Space-separated scopes required (verify) or granted (generate). Scope mismatch → 403. |
| `<ExpiresIn>` | Access-token lifetime in **milliseconds**. |
| `<RefreshTokenExpiresIn>` | Refresh-token lifetime in ms. |
| `<GrantType>` | Where to read the grant type from, for generate operations. |
| `<GenerateResponse>` | `true` → policy writes the standard OAuth JSON response itself. `false` → token data lands in flow variables for you to handle. |
| `<Tokens>` | Used by `InvalidateToken` to point at the token(s) to revoke. |

### 5.3 VerifyJWT / GenerateJWT / DecodeJWT

Validate, mint, or just read a JWT. Three sibling policies; `VerifyJWT` shown.

```xml
<VerifyJWT name="Verify-JWT-1">
    <Algorithm>RS256</Algorithm>
    <Source>request.header.authorization</Source>
    <PublicKey>
        <JWKS uri="https://issuer.example.com/.well-known/jwks.json"/>
    </PublicKey>
    <Issuer>https://issuer.example.com</Issuer>
    <Audience>my-api</Audience>
    <Subject>subject-value</Subject>
    <AdditionalClaims>
        <Claim name="role">admin</Claim>
    </AdditionalClaims>
</VerifyJWT>
```

| Element | What it does |
|---|---|
| `<Algorithm>` | Signing algorithm to enforce — `RS256`/`HS256`/etc. **Pin this**; accepting `none` or letting the token pick is a classic JWT vuln. |
| `<Source>` | Where the JWT comes from. |
| `<PublicKey><JWKS uri>` | For RS*: fetch the verification key from a JWKS endpoint (rotates automatically). Alternatives: `<Value>` (inline PEM) or `<SecretKey>` for HS*. |
| `<Issuer>` | Required `iss` claim value — rejects tokens from other issuers. |
| `<Audience>` | Required `aud` claim — this API must be a named audience. |
| `<Subject>` | Required `sub` claim, if you pin it. |
| `<AdditionalClaims>` | Extra claims that must be present and match — your custom authorization gates. |

`GenerateJWT` swaps in `<PrivateKey>`, `<ExpiresIn>`, and `<AdditionalClaims>` you want to *write*. `DecodeJWT` just needs `<Source>` — it parses without verifying (use only for inspection, never for trust decisions).

### 5.4 Basic Authentication

Encodes credentials into, or decodes them out of, an `Authorization: Basic` header. Typically used to authenticate **to the backend**.

```xml
<BasicAuthentication name="Basic-Authentication-1">
    <Operation>Encode</Operation>
    <User ref="request.queryparam.username"/>
    <Password ref="request.queryparam.password"/>
    <AssignTo createNew="false">request.header.Authorization</AssignTo>
    <Source>request.header.Authorization</Source>
    <IgnoreUnresolvedVariables>false</IgnoreUnresolvedVariables>
</BasicAuthentication>
```

| Element | What it does |
|---|---|
| `<Operation>` | `Encode` (build the header from user+password) or `Decode` (split a header back into variables). |
| `<User>` / `<Password>` | Credential sources for Encode (literal or `ref`). |
| `<AssignTo>` | Encode: where to write the resulting header. `createNew="true"` forces a new variable rather than overwriting. |
| `<Source>` | Decode: the header to read and split. |

### 5.5 JSON Threat Protection

Rejects malicious/oversized JSON bodies before anything parses them.

```xml
<JSONThreatProtection name="JSON-Threat-Protection-1">
    <Source>request</Source>
    <ContainerDepth>10</ContainerDepth>
    <ArrayElementCount>20</ArrayElementCount>
    <ObjectEntryCount>15</ObjectEntryCount>
    <ObjectEntryNameLength>50</ObjectEntryNameLength>
    <StringValueLength>500</StringValueLength>
</JSONThreatProtection>
```

| Element | What it does |
|---|---|
| `<Source>` | Which message to inspect (`request`/`response`/named variable). |
| `<ContainerDepth>` | Max nesting depth of objects/arrays — caps deeply nested "billion laughs"-style payloads. |
| `<ArrayElementCount>` | Max entries in any array. |
| `<ObjectEntryCount>` | Max key/value pairs in any object. |
| `<ObjectEntryNameLength>` | Max length of any key name. |
| `<StringValueLength>` | Max length of any string value. |

Any limit breached → policy faults and the call never reaches your parser/backend.

### 5.6 XML Threat Protection

The XML equivalent — caps structure and value sizes to stop XML bombs and oversized payloads.

```xml
<XMLThreatProtection name="XML-Threat-Protection-1">
    <Source>request</Source>
    <NameLimits>
        <Element>10</Element>
        <Attribute>10</Attribute>
        <NamespacePrefix>10</NamespacePrefix>
    </NameLimits>
    <StructureLimits>
        <NodeDepth>5</NodeDepth>
        <AttributeCountPerElement>2</AttributeCountPerElement>
        <NamespaceCountPerElement>3</NamespaceCountPerElement>
        <ChildCount includeComment="true" includeElement="true"
                    includeProcessingInstruction="true" includeText="true">3</ChildCount>
    </StructureLimits>
    <ValueLimits>
        <Text>15</Text>
        <Attribute>10</Attribute>
        <NamespaceURI>10</NamespaceURI>
    </ValueLimits>
</XMLThreatProtection>
```

| Element group | What it does |
|---|---|
| `<NameLimits>` | Max character length of element names, attribute names, namespace prefixes — blocks abuse via absurdly long identifiers. |
| `<StructureLimits>` | Shape caps: `NodeDepth` (nesting), `AttributeCountPerElement`, `NamespaceCountPerElement`, and `ChildCount` (with toggles for what counts as a child). The core anti-XML-bomb controls. |
| `<ValueLimits>` | Max length of text content, attribute values, namespace URIs, comments, PI data. |

### 5.7 Regular Expression Protection

Pattern-matches request parts against regexes to catch SQL injection / XSS / scripting attempts.

```xml
<RegularExpressionProtection name="Regular-Expression-Protection-1">
    <Source>request</Source>
    <IgnoreUnresolvedVariables>false</IgnoreUnresolvedVariables>
    <QueryParam name="q">
        <Pattern>[\s]*(?i)(union)(.*?)(select)[\s]*</Pattern>
    </QueryParam>
    <Header name="X-Input">
        <Pattern>(?i)<script>(.*?)</script></Pattern>
    </Header>
    <JSONPayload>
        <JSONPath>
            <Expression>$.input</Expression>
            <Pattern>[\s]*(?i)(drop)(.*?)(table)[\s]*</Pattern>
        </JSONPath>
    </JSONPayload>
</RegularExpressionProtection>
```

| Element | What it does |
|---|---|
| `<Source>` | Message to scan. |
| `<QueryParam>` / `<Header>` / `<FormParam>` / `<URIPath>` | Target a named part of the request; one or more `<Pattern>` regexes that, if matched, fault the call. |
| `<XMLPayload>` / `<JSONPayload>` | Drill into the body via `<XPath>` / `<JSONPath>` `<Expression>`, then apply `<Pattern>` to the extracted value. |
| `<Pattern>` | The regex. **A match = reject.** These are blocklists; keep them tight to avoid false positives on legit traffic. |

### 5.8 Access Control

Allow/deny by **source IP** — an allowlist/blocklist at the edge.

```xml
<AccessControl name="Access-Control-1">
    <IPRules noRuleMatchAction="ALLOW">
        <MatchRule action="DENY">
            <SourceAddress mask="24">198.51.100.0</SourceAddress>
        </MatchRule>
    </IPRules>
</AccessControl>
```

| Element / attribute | What it does |
|---|---|
| `<IPRules noRuleMatchAction>` | Default action when no rule matches — `ALLOW` (blocklist mode) or `DENY` (allowlist mode). |
| `<MatchRule action>` | `ALLOW` or `DENY` for IPs matching this rule. |
| `<SourceAddress mask>` | IP (or CIDR via `mask`) to match. `mask="24"` = a /24 range. |

> Behind a load balancer, `client.ip` may be the LB. Drive matching off `X-Forwarded-For` instead, or your rules match the wrong address.

### 5.9 SAML Assertion (Generate / Validate)

`ValidateSAMLAssertion` checks an inbound SOAP message's signed SAML assertion; `GenerateSAMLAssertion` signs and attaches one outbound. SOAP/WS-Security territory.

```xml
<ValidateSAMLAssertion name="Validate-SAML-1">
    <Source>request</Source>
    <TrustStore>my-truststore</TrustStore>
    <RemoveAssertion>false</RemoveAssertion>
</ValidateSAMLAssertion>
```

| Element | What it does |
|---|---|
| `<Source>` | Message carrying the assertion. |
| `<TrustStore>` | Keystore of trusted signing certs used to validate the signature. |
| `<RemoveAssertion>` | `true` → strip the assertion from the message before forwarding to backend. |

---

## 6. Mediation policies

Transform, route, and shape messages. This is where most real work happens.

### 6.1 Assign Message

The Swiss-army knife — build, copy, set, or strip headers / params / body / verb on a message.

```xml
<AssignMessage name="Assign-Message-1">
    <Add>
        <Headers>
            <Header name="X-Source">api-mgmt</Header>
        </Headers>
    </Add>
    <Copy source="request">
        <Headers>
            <Header name="Authorization"/>
        </Headers>
        <Payload>true</Payload>
    </Copy>
    <Remove>
        <Headers>
            <Header name="apikey"/>
        </Headers>
    </Remove>
    <Set>
        <Headers>
            <Header name="Content-Type">application/json</Header>
        </Headers>
        <Payload contentType="application/json">{"status":"ok"}</Payload>
        <Verb>POST</Verb>
        <Path>/v2/orders</Path>
        <StatusCode>200</StatusCode>
        <ReasonPhrase>OK</ReasonPhrase>
    </Set>
    <AssignVariable>
        <Name>my.var</Name>
        <Ref>request.header.x</Ref>
        <Value>fallback</Value>
    </AssignVariable>
    <IgnoreUnresolvedVariables>true</IgnoreUnresolvedVariables>
    <AssignTo createNew="false" transport="http" type="request">request</AssignTo>
</AssignMessage>
```

| Element | What it does |
|---|---|
| `<Add>` | Adds headers/query/form params **only if not already present** — non-destructive. |
| `<Copy source>` | Copies parts from another message (`request`/`response`/named) into the target. `<Payload>true</Payload>` copies the body. |
| `<Remove>` | Deletes the named headers/params/payload. Use to strip the `apikey` before it leaks to the backend. |
| `<Set>` | **Overwrites** headers/params/payload/verb/path/status. Listed children set exactly those parts. |
| `<AssignVariable>` | Creates/sets a flow variable: `<Name>` = variable, `<Ref>` = source variable, `<Value>` = literal/fallback if the ref is unresolved. |
| `<AssignTo>` | Which message this policy acts on. `createNew="true"` builds a brand-new message (used to assemble a Service Callout request). `type` = `request`/`response`. |

### 6.2 Extract Variables

Pulls values **out** of a message into flow variables for later use.

```xml
<ExtractVariables name="Extract-Variables-1">
    <Source clearPayload="false">request</Source>
    <VariablePrefix>extracted</VariablePrefix>
    <IgnoreUnresolvedVariables>true</IgnoreUnresolvedVariables>
    <URIPath>
        <Pattern ignoreCase="true">/accounts/{id}</Pattern>
    </URIPath>
    <QueryParam name="code">
        <Pattern ignoreCase="true">DBN{dbncode}</Pattern>
    </QueryParam>
    <Header name="Authorization">
        <Pattern>Bearer {token}</Pattern>
    </Header>
    <JSONPayload>
        <Variable name="customerName" type="string">
            <JSONPath>$.customer.name</JSONPath>
        </Variable>
    </JSONPayload>
    <XMLPayload>
        <Namespaces>
            <Namespace prefix="ns">http://example.com</Namespace>
        </Namespaces>
        <Variable name="orderId" type="string">
            <XPath>/ns:order/ns:id</XPath>
        </Variable>
    </XMLPayload>
</ExtractVariables>
```

| Element | What it does |
|---|---|
| `<Source clearPayload>` | Message to read; `clearPayload="true"` frees the body after extraction (memory). |
| `<VariablePrefix>` | Namespace for the variables created, e.g. `extracted.id`. Prevents collisions. |
| `<URIPath>/<QueryParam>/<Header>/<FormParam>` | The `<Pattern>` uses `{curly}` placeholders — each placeholder becomes a variable holding the matched substring. |
| `<JSONPayload><Variable>` | Each `<JSONPath>` `<Expression>` result is stored in the named `<Variable>` of the given `type`. |
| `<XMLPayload>` | Same via `<XPath>`; declare `<Namespaces>` first or your XPath won't match namespaced XML. |

### 6.3 JSON to XML

Converts a JSON message body to XML.

```xml
<JSONToXML name="JSON-to-XML-1">
    <Source>request</Source>
    <OutputVariable>request</OutputVariable>
    <Options>
        <NullValue>NULL</NullValue>
        <ObjectRootElementName>Root</ObjectRootElementName>
        <ArrayRootElementName>Array</ArrayRootElementName>
        <ArrayItemElementName>Item</ArrayItemElementName>
        <AttributePrefix>@</AttributePrefix>
        <AttributeBlockName>#attrs</AttributeBlockName>
        <TextNodeName>#text</TextNodeName>
        <NamespaceSeparator>:</NamespaceSeparator>
        <InvalidCharsReplacement>_</InvalidCharsReplacement>
    </Options>
</JSONToXML>
```

| Element | What it does |
|---|---|
| `<Source>` / `<OutputVariable>` | Read-from and write-to messages (often both `request`/`response`). |
| `<NullValue>` | What a JSON `null` becomes in XML. |
| `<ObjectRootElementName>` | Name for the single wrapping root element XML requires (JSON can be rootless). |
| `<ArrayRootElementName>` / `<ArrayItemElementName>` | How JSON arrays (no XML equivalent) get wrapped and named per item. |
| `<AttributePrefix>` / `<AttributeBlockName>` | Convention for turning JSON keys into XML attributes vs elements. |
| `<TextNodeName>` | Key whose value becomes element text content. |
| `<InvalidCharsReplacement>` | JSON keys can contain chars illegal in XML element names; this is the substitute. |

### 6.4 XML to JSON

The reverse, with type-recognition options.

```xml
<XMLToJSON name="XML-to-JSON-1">
    <Source>response</Source>
    <OutputVariable>response</OutputVariable>
    <Options>
        <RecognizeNumber>true</RecognizeNumber>
        <RecognizeBoolean>true</RecognizeBoolean>
        <RecognizeNull>true</RecognizeNull>
        <NullValue>NULL</NullValue>
        <AttributePrefix>@</AttributePrefix>
        <TextNodeName>#text</TextNodeName>
        <StripLevels>0</StripLevels>
        <TreatAsArray>
            <Path unwrap="true">orders/order</Path>
        </TreatAsArray>
    </Options>
</XMLToJSON>
```

| Element | What it does |
|---|---|
| `<RecognizeNumber>` / `<RecognizeBoolean>` / `<RecognizeNull>` | Emit JSON numbers/booleans/null instead of strings. Off → everything is a quoted string. |
| `<AttributePrefix>` / `<TextNodeName>` | How XML attributes and text nodes are named in the JSON. |
| `<StripLevels>` | Drop N levels of wrapper elements from the top of the result. |
| `<TreatAsArray><Path>` | **The important one.** A single XML element produces a JSON object; a repeated one produces an array — so output shape changes with the data. Force a path to *always* be an array for a stable schema. `unwrap="true"` removes the wrapper element. |

### 6.5 XSL Transform

Applies an XSLT stylesheet to an XML message — heavy structural transformation.

```xml
<XSL name="XSL-Transform-1">
    <Source>request</Source>
    <ResourceURL>xsl://transform.xsl</ResourceURL>
    <Parameters ignoreUnresolvedVariables="false">
        <Parameter name="cutoff" ref="request.header.cutoff"/>
        <Parameter name="mode" value="full"/>
    </Parameters>
    <OutputVariable>request.content</OutputVariable>
</XSL>
```

| Element | What it does |
|---|---|
| `<Source>` | XML message to transform. |
| `<ResourceURL>` | The stylesheet, stored as a proxy resource (`xsl://name.xsl`). |
| `<Parameters>` | XSLT `<xsl:param>` values, from literals (`value`) or flow variables (`ref`). |
| `<OutputVariable>` | Where the transformed result lands. |

### 6.6 Message Validation (SOAP)

Validates a message against a WSDL/XSD/schema — well-formedness and schema conformance.

```xml
<MessageValidation name="SOAP-Message-Validation-1">
    <Source>request</Source>
    <ResourceURL>wsdl://service.wsdl</ResourceURL>
    <SOAPMessage version="1.1"/>
    <Element namespace="http://example.com/ns">OrderRequest</Element>
</MessageValidation>
```

| Element | What it does |
|---|---|
| `<Source>` | Message to validate. |
| `<ResourceURL>` | The `wsdl://` or `xsd://` resource to validate against. Omit to only check XML well-formedness. |
| `<SOAPMessage version>` | Expected SOAP version (`1.1`/`1.2`). |
| `<Element namespace>` | The expected root element + namespace of the SOAP body. |

### 6.7 Key Value Map Operations

Read/write a **Key Value Map (KVM)** — persistent config/secrets store. Put rarely, Get often.

```xml
<KeyValueMapOperations name="KVM-Operations-1" mapIdentifier="backend-config">
    <Scope>environment</Scope>
    <ExpiryTimeInSecs>300</ExpiryTimeInSecs>
    <InitialEntries>
        <Entry>
            <Key><Parameter>baseUrl</Parameter></Key>
            <Value>https://backend.example.com</Value>
        </Entry>
    </InitialEntries>
    <Get assignTo="target.url" index="1">
        <Key><Parameter>baseUrl</Parameter></Key>
    </Get>
    <Put override="true">
        <Key><Parameter ref="request.queryparam.k"/></Key>
        <Value ref="request.queryparam.v"/>
    </Put>
    <Delete>
        <Key><Parameter>baseUrl</Parameter></Key>
    </Delete>
</KeyValueMapOperations>
```

| Element / attribute | What it does |
|---|---|
| `mapIdentifier` | Name of the KVM to operate on. |
| `<Scope>` | Visibility/isolation: `organization` / `environment` / `apiproxy` / `policy`. Wider scope = shared across more proxies. |
| `<ExpiryTimeInSecs>` | Cache TTL for read values (KVM reads are cached for performance). |
| `<InitialEntries>` | Seed values written at deploy time. |
| `<Get assignTo index>` | Read a key's value into a variable. `index` selects which value for multi-value keys. |
| `<Put override>` | Write a key/value; `override="true"` replaces an existing key. |
| `<Delete>` | Remove a key. |

> KVMs can be marked **encrypted** for secrets — use that for credentials rather than hardcoding in policies.

### 6.8 Raise Fault

Deliberately stops the flow and returns a **custom error** to the caller.

```xml
<RaiseFault name="Raise-Fault-1">
    <FaultResponse>
        <Set>
            <Headers>
                <Header name="Content-Type">application/json</Header>
            </Headers>
            <Payload contentType="application/json">{"error":"invalid request"}</Payload>
            <StatusCode>400</StatusCode>
            <ReasonPhrase>Bad Request</ReasonPhrase>
        </Set>
    </FaultResponse>
    <IgnoreUnresolvedVariables>true</IgnoreUnresolvedVariables>
</RaiseFault>
```

| Element | What it does |
|---|---|
| `<FaultResponse><Set>` | Builds the error response — headers, payload, `StatusCode`, `ReasonPhrase`. Same `<Set>` grammar as Assign Message. |
| `<IgnoreUnresolvedVariables>` | As elsewhere — controls behaviour if the payload references a missing variable. |

> Attach it conditionally (e.g. on a `<Step>` with a `<Condition>`) to short-circuit on validation failures, or inside a `<FaultRule>` to standardise backend errors.

---

## 7. Extension policies

Reach outside the proxy, run code, log, and collect custom analytics.

### 7.1 Service Callout

Calls **another service mid-flow** (e.g. enrich the request with a lookup) without changing the main backend target.

```xml
<ServiceCallout name="Service-Callout-1">
    <Request clearPayload="true" variable="lookupRequest">
        <IgnoreUnresolvedVariables>false</IgnoreUnresolvedVariables>
        <Set>
            <Verb>GET</Verb>
            <Path>/customers/{id}</Path>
        </Set>
    </Request>
    <Response>lookupResponse</Response>
    <Timeout>30000</Timeout>
    <HTTPTargetConnection>
        <URL>https://lookup.example.com</URL>
    </HTTPTargetConnection>
</ServiceCallout>
```

| Element | What it does |
|---|---|
| `<Request variable>` | Name of the request message to build/send; `<Set>` populates it. `clearPayload` frees it after. |
| `<Response>` | Variable to store the callout's response in — you parse it later with Extract Variables. |
| `<Timeout>` | Max wait in ms; protects the main flow from a slow dependency. |
| `<HTTPTargetConnection><URL>` | The external endpoint. Use `<LocalTargetConnection>` instead to call another proxy in the same tenant without leaving the gateway. |

### 7.2 Access Entity

Loads a **management entity** (developer, app, API product, company) into a variable for inspection within the flow.

```xml
<AccessEntity name="Access-Entity-1">
    <EntityType value="developer"/>
    <EntityIdentifier ref="developer.id" type="developerId"/>
    <SecondaryIdentifier ref="apiproduct.name" type="productName"/>
</AccessEntity>
```

| Element | What it does |
|---|---|
| `<EntityType value>` | Kind of entity to fetch: `developer` / `app` / `apiproduct` / `company`. |
| `<EntityIdentifier ref type>` | How to look it up — value source plus the identifier type. |
| `<SecondaryIdentifier>` | A second key when one isn't enough to resolve the entity. |

> Output is XML; pair with Extract Variables to read fields (e.g. a custom attribute on the app to drive routing).

### 7.3 Statistics Collector

Captures custom values into **analytics** for later reporting.

```xml
<StatisticsCollector name="Statistics-Collector-1">
    <Statistics>
        <Statistic name="region" ref="request.header.region" type="STRING">unknown</Statistic>
        <Statistic name="orderValue" ref="extracted.value" type="INTEGER">0</Statistic>
    </Statistics>
</StatisticsCollector>
```

| Element / attribute | What it does |
|---|---|
| `<Statistic name>` | The dimension/metric name as it appears in analytics. |
| `ref` | Flow variable supplying the value. |
| `type` | Data type — `STRING`/`INTEGER`/`FLOAT`/etc. Drives how analytics aggregates it. |
| (element text) | Default value when the `ref` is unresolved. |

### 7.4 JavaScript

Runs custom JavaScript for logic the declarative policies can't express.

```xml
<Javascript name="JavaScript-1" timeLimit="200">
    <ResourceURL>jsc://transform.js</ResourceURL>
    <IncludeURL>jsc://lib.js</IncludeURL>
    <Properties>
        <Property name="threshold">100</Property>
    </Properties>
</Javascript>
```

| Element / attribute | What it does |
|---|---|
| `timeLimit` | Max execution ms before the policy is killed — guards against runaway scripts. |
| `<ResourceURL>` | The JS file (proxy resource, `jsc://`). |
| `<IncludeURL>` | Library files loaded *before* the main script (can repeat). |
| `<Properties>` | Config passed to the script, readable via `properties.threshold`. |

> The script accesses the message via the `context` object (`context.getVariable(...)`, `context.setVariable(...)`). Keep it lightweight — it runs on every call.

### 7.5 Python Script

Same idea, Python (Jython).

```xml
<Script name="Python-Script-1" timeLimit="200">
    <ResourceURL>py://script.py</ResourceURL>
    <IncludeURL>py://helpers.py</IncludeURL>
</Script>
```

| Element | What it does |
|---|---|
| `<ResourceURL>` | Main Python file (`py://`). |
| `<IncludeURL>` | Dependency files loaded first. |

### 7.6 Java Callout

Runs compiled Java from a JAR — for heavy or library-dependent logic.

```xml
<JavaCallout name="Java-Callout-1">
    <ClassName>com.example.MyExecution</ClassName>
    <ResourceURL>java://custom-policy.jar</ResourceURL>
    <Properties>
        <Property name="config">value</Property>
    </Properties>
</JavaCallout>
```

| Element | What it does |
|---|---|
| `<ClassName>` | Fully-qualified class implementing the callout interface. |
| `<ResourceURL>` | The JAR resource (`java://`). |
| `<Properties>` | Config passed to the Java code. |

### 7.7 Flow Callout

Invokes a **shared flow** — a reusable bundle of policies (auth, logging, CORS) maintained once and called from many proxies.

```xml
<FlowCallout name="Flow-Callout-1">
    <SharedFlowBundle>common-security</SharedFlowBundle>
    <Parameters>
        <Parameter name="env">prod</Parameter>
    </Parameters>
</FlowCallout>
```

| Element | What it does |
|---|---|
| `<SharedFlowBundle>` | Name of the shared flow to run. This is your DRY mechanism — centralise cross-cutting concerns. |
| `<Parameters>` | Values handed to the shared flow. |

### 7.8 Message Logging

Sends log messages to **syslog** (or a local file in some deployments) — typically in PostFlow / fault rules so it runs after the response is ready.

```xml
<MessageLogging name="Message-Logging-1">
    <Syslog>
        <Message>{system.timestamp} {apiproxy.name} status={response.status.code}</Message>
        <Host>logs.example.com</Host>
        <Port>514</Port>
        <Protocol>TCP</Protocol>
        <FormatMessage>true</FormatMessage>
    </Syslog>
    <logLevel>INFO</logLevel>
</MessageLogging>
```

| Element | What it does |
|---|---|
| `<Syslog><Message>` | The log line; use `{variables}` to inject runtime data. |
| `<Host>` / `<Port>` / `<Protocol>` | Syslog destination and transport (`TCP`/`UDP`; add `<SSLInfo>` for TLS). |
| `<FormatMessage>` | `true` → prepend standard syslog formatting. |
| `<logLevel>` | Severity tag (`INFO`/`WARN`/`ERROR`/etc.). |

> Attach this with `continueOnError="true"` — you almost never want a logging hiccup to fail a customer's API call.

---

## 8. Practical gotchas

A few that bite in real proxies, beyond the per-policy notes:

- **It's Apigee Edge under the hood.** When this doc is thin, the Apigee Edge policy reference is the authoritative deep-dive — element names match. But don't assume *every* Apigee policy is exposed in your SAP tenant; the catalog is curated and version-dependent. Check the editor.
- **Order is everything.** Request-flow sequence should be: protect (Spike Arrest) → authenticate (Verify API Key / OAuth) → authorize/quota → threat-protect → transform. Putting Quota before auth means you're counting unauthenticated noise against the limit.
- **`continueOnError` is a deliberate choice, not a default to ignore.** `false` everywhere makes one flaky policy fail the whole call; `true` everywhere hides real failures. Set it per policy with intent — `true` for logging/analytics, `false` for security and validation.
- **Threat protection runs before parsing, by design.** Attach JSON/XML Threat Protection *ahead* of any policy that reads the body, or the bomb detonates first.
- **Response Cache and KVM are two-touch / cached-read.** Response Cache needs a lookup step and a populate step. KVM reads are cached for `ExpiryTimeInSecs` — a freshly `Put` value won't be visible until the cache expires unless scoped/configured for it.
- **Variables only exist after the policy that creates them.** `apikey`, `developer.*`, `app.*` are empty until Verify API Key / OAuth has run earlier in the same flow. Referencing them before that yields nulls.
- **Faults jump out of the normal flow.** A policy failure (with `continueOnError="false"`) skips straight to FaultRules — any cleanup/logging you wanted must live in a fault rule too, not only in PostFlow.

---

*Reference compiled for SAP API Management (Integration Suite). Policy availability and schema details are release-dependent — confirm against the policy editor's generated XML and current SAP Help before relying on any element in production.*
