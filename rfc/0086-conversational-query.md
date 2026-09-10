# Meta

* RFC Name: Conversational Query
* RFC ID: 86
* Start Date: 2026-08-31
* Owner: Jared Casey
* Current Status: DRAFT
* Current Editor: Anirudh Lakhotia
* Revision: 3
* Supporting Material: [sdk-design](https://github.com/couchbaselabs/sdk-design/tree/main/server-aligned/totoro/conversational-query)

# Summary

Conversational Query allows applications to submit natural-language prompts to the Query service and
receive generated SQL++ statements and Query results. This RFC defines one-shot queries, multi-turn
chats, model credentials, Knowledge management, and the SDK behavior required to continue or persist
a chat across application requests.

# Motivation

Applications may need to continue a conversation across requests handled by different processes. A
live chat resides on one Query node, so the SDK must retain enough client state to address it from a
new SDK instance.

# General Design

Conversational Query uses the Query service to generate SQL++ from a natural-language prompt and,
when requested, execute the generated statement under the caller's Couchbase identity. Model access
uses separate model credentials.

A live chat is held in memory by one Query node. The SDK retains its logical owner for routing and can serialize
the required client state in a `ChatHandle`. `chat(handle)` reconstructs a `Chat` from a previously
serialized handle and performs no Query request. Persisting a chat requires PAUSE; RESUME restores a
paused chat to live server state.

Chat discovery, model-provider discovery, typed EXPLAIN/ADVISE actions, a public chat-id accessor,
single-entry Knowledge lookup, and changes to the inactivity timeout after BEGIN are outside this
revision.

## Public API

```
Keyspaces = list<string>  // SQL++ keyspace paths

Cluster.conversationalQuery([credential ModelCredential], [model ModelOptions])
    -> ConversationalQuery

ConversationalQuery
    query(prompt string,
          keyspaces Keyspaces,
          [options QueryOptions],
          [credential ModelCredential])
        -> ConversationalQueryResult

    beginChat(keyspaces Keyspaces,
              [options BeginChatOptions],
              [credential ModelCredential])
        -> Chat

    chat(handle ChatHandle,
         [credential ModelCredential])
        -> Chat                       // local operation; performs no server I/O

    knowledge() -> KnowledgeManager

KnowledgeManager
    upsert(name string, keyspace string, value string)   // CREATE OR REPLACE
    create(name string, keyspace string, value string)   // CREATE; fails if it exists
    drop(name string, keyspace string)
    getAll([keyspace string]) -> KnowledgeEntry[]

KnowledgeEntry {
    name string
    keyspace string
    value string
}

Chat
    ask(prompt string, [options AskOptions])   -> ConversationalQueryResult
    pause([options PauseChatOptions])          -> ConversationalQueryMetaData
    resume()                                   -> ConversationalQueryMetaData
    end()                                      -> ConversationalQueryMetaData

    handle() -> ChatHandle
    state()  -> ChatState

ModelCredential
    capella(email string, password string, organizationId string)
    stored(credentialName string)
    apiKey(key string)

ModelOptions {
    provider string
    name string
    endpoint string
    region string
    outputTokenLimit int
    moderation optional<bool>
    raw map<string, JsonValue>
}

QueryOptions {
    model ModelOptions
    hint string
    execute bool
    output ConversationalQueryOutput
    knowledge bool
}

BeginChatOptions {
    inactivityTimeout Duration
    knowledge bool
}

AskOptions {
    hint string
    execute bool
    output ConversationalQueryOutput
}

PauseChatOptions {
    summarize optional<bool>
}

ConversationalQueryOutput {
    Sqlpp
    FtsSqlpp
    JsUdf
}

ChatState {
    Active
    Paused
    Indeterminate
}
```

Common Query request controls and asynchronous forms follow existing SDK conventions.

## Model Credentials

`ModelCredential` authenticates Query's calls to the model provider. The SDK must not use it for
Couchbase authentication or accept it in `ClusterOptions`.

| Credential | Request parameters |
| --- | --- |
| `capella(email, password, organizationId)` | `natural_cred` = `email:password`; `natural_orgid` = organization id |
| `stored(credentialName)` | `natural_config.cred_id` |
| `apiKey(key)` | `natural_config.api_key` |

A credential supplied to `conversationalQuery` is the default. An explicit credential on `query`,
`beginChat`, or `chat(handle)` replaces that default for the call or returned `Chat`.

Query does not store model credentials with a live or paused chat. The SDK therefore sends the
effective credential on each request that may invoke the model. BEGIN, RESUME and END do not invoke
the model and do not carry model credentials.

The three credential forms are mutually exclusive. The SDK must not:

* combine Capella credentials with `stored` or `apiKey`;
* send both `cred_id` and `api_key`; or
* fall back to another credential form after a failure.

These rules apply to the assembled request because Query silently applies credential precedence;
Capella credentials cause it to ignore `natural_config`.

If no model credential remains after applying defaults and overrides, Query uses the direct-provider
path and validates whether the resulting configuration can authenticate. The SDK must not reproduce
provider-specific authentication rules or introduce additional credential kinds.

For `capella`, Query performs the Capella authentication and completion calls. The SDK does not call
Capella directly.

SDKs must not inspect or change cluster configuration to pre-validate model-credential prerequisites.
Those prerequisites are documented under [Documentation](#documentation); failures use normal Query
error handling.

Secrets must be sent as request parameters, never embedded in statement text. The SDK must redact
`natural_cred`, `natural_config.api_key`, and Capella passwords from errors, error contexts,
diagnostics, logs, and tracing.

`stored` sends only the name of an existing server credential as `cred_id`, not its secret. The SDK
does not create or update stored credentials.

## Model Options

All `ModelOptions` fields are optional.

`provider` and `name` are open string values. SDKs must not reject them based on a hard-coded list of supported providers or models.

* `provider`: provider identifier. If omitted, the direct-provider path may use a server default.
  Query rejects unsupported provider names.
* `name`: model identifier. Query passes it verbatim to the selected provider without validating
  that the model exists.
* `endpoint`: complete completions URL for the direct-provider path. Query appends no suffix. An
  explicit endpoint currently permits configurations without a model credential.
* `region`: provider region; currently used by the Bedrock path.
* `outputTokenLimit`: output-token limit for the direct-provider path.
* `moderation`: optional boolean. Unset omits the field and uses Query's default moderation behavior.
  Explicit `true` or `false` preserves the caller's choice.
* `raw`: additional top-level `natural_config` entries; values may be any JSON value.

A `ModelOptions` supplied to `conversationalQuery` is the default model configuration. A `Chat`
created by `beginChat` uses those defaults. A `Chat` reconstructed with `chat(handle)` uses the model
defaults of the attaching `ConversationalQuery`; the handle contains no model configuration.

For a one-shot query, `QueryOptions.model` replaces that default as a whole, including `raw`;
fields are not merged. An explicitly empty replacement inherits no model fields or raw entries.

Credential selection is independent of model replacement.

### Raw model options

`raw` exists only as a forward-compatibility mechanism for future Query fields. Current Query
versions silently discard unrecognized `natural_config` keys; `raw` is not provider passthrough.

The SDK must reject these keys in `raw` because they already have typed representations:

`provider`, `model`, `endpoint`, `region`, `output_token_limit`, `moderation`, `cred_id`, `api_key`.

This applies even when the corresponding typed option or credential is unset. Allowing the same
setting through two paths would make precedence SDK-specific.

Other `raw` entries are added unchanged to `natural_config`. They cannot override Query-controlled
settings such as temperature or seed.

The SDK must reject options with no wire representation for the selected credential path; see
[Wire Mapping](#wire-mapping).

## One-shot Query

`query` submits a single prompt with at least one caller-supplied keyspace that Query may use. It
returns a `ConversationalQueryResult` containing the generated SQL++ statement, when supplied, and
any execution results. Each call is independent, uses no chat state, creates no `ChatHandle`, and
requires no Query-node affinity.

Execution is enabled by default. `execute(false)` requests SQL++ generation without execution.

## Chat Lifecycle

A live chat is held in memory by one Query node. It can disappear if that node restarts, is failed over or rebalanced out, or if the chat is evicted or expires from inactivity. Unless the chat has been paused first, that state is not persisted elsewhere.

A paused chat is stored persistently by Query and has its own server-controlled expiry. SDKs must not try to predict when live or paused chats expire or reject requests based on hard-coded server limits.

Applications must not perform multiple operations on the same chat concurrently. SDKs are not required to coordinate concurrent use of separate `Chat` instances or copies of the same `ChatHandle`.

### State

`Chat.state()` reports the state the SDK last observed for the chat. It does not query the server or guarantee that the chat is still in that state.

| State | Meaning |
| --- | --- |
| `Active` | The SDK last saw the chat as live and knows which Query node owns it. |
| `Paused` | PAUSE succeeded, so the SDK no longer treats any Query node as the live owner. |
| `Indeterminate` | The SDK sent PAUSE but lost the response, so it does not know whether the chat is still live or was successfully paused. |

`Indeterminate` prevents another `ask` because PAUSE may already have removed the live chat from its owner.

| Operation | Permitted local state | Effect of a successful response |
| --- | --- | --- |
| `ask` | Active | Stay Active. |
| `pause` | Active | Become Paused. |
| `resume` | Paused or Indeterminate | Become Active; the Query node that serves RESUME becomes the new owner. |
| `end` | Active or Indeterminate | End the chat. |

Operations outside their permitted local states fail with the SDK's normal invalid-operation error.

### Begin

`beginChat` creates a new live chat. The SDK sends BEGIN to any Query node with the requested keyspaces and optional inactivity timeout. The node that serves the request becomes the chat's owner.

Unlike one-shot `query`, the SDK does not require the keyspace list to be nonempty. Query validates the supplied keyspaces and inactivity timeout.

BEGIN does not call the model. Model options, model credentials, and the Knowledge choice are therefore not sent with the BEGIN request.

The SDK keeps the chat's Knowledge choice for later `ask` calls. A credential passed to `beginChat` similarly becomes the credential used by the returned `Chat` when a later operation needs the model.

### Ask

`ask` sends a prompt to an Active chat. The SDK must send it to the Query node that owns the chat.

The chat's keyspaces were set by BEGIN and cannot be changed for an individual turn. The SDK sends the chat's Knowledge choice with every `ask`. `execute` and `output` behave the same way as for a one-shot query.

A hint applies only to the current turn and is not retained as chat configuration. Its text remains
in conversation history until that history is summarized or rebuilt.

A failed `ask` leaves the local state Active, but may already have added the prompt to the
conversation. The [non-replay rule](#retries) applies.

### Pause

`pause` saves the conversation so it can later be resumed and releases the chat's live slot on its Query node.

After a successful PAUSE, the SDK marks the chat Paused and no longer keeps a live owner for it.

`summarize` controls whether Query may summarize the conversation before saving it:

- unset: let Query decide;
- `true`: request summarization;
- `false`: do not summarize.

If summarization is possible, PAUSE may invoke the model, so the SDK sends the chat's model configuration and available model credential. When `summarize` is `false`, it sends neither. Because PAUSE may include a model call, its default request timeout must not be shorter than the SDK's model-turn timeout.

If Query returns a PAUSE error, the SDK keeps the chat Active and retains its owner. This does not guarantee that the conversation history is unchanged or that no persisted copy exists: summarization may already have replaced the live history before persistence failed.

If a PAUSE request may have reached Query before a transport failure, the outcome is handled under [Ambiguous Outcomes](#ambiguous-outcomes).

### Resume

`resume` turns a Paused chat back into a live chat.

For a Paused chat, the SDK may send RESUME to any Query node. If it succeeds, that node becomes the new owner and the SDK marks the chat Active.

If Query returns an error, the SDK keeps the chat Paused. Neither error 19243 nor metadata error 4500 proves that the persisted chat no longer exists.

For an Indeterminate chat, RESUME instead targets the Query node retained from the ambiguous PAUSE. The additional rules, including the 19255 case, are described under [Ambiguous Outcomes](#ambiguous-outcomes).

The SDK must not use RESUME as a way to check whether a chat is paused. A successful RESUME changes server state by consuming the persisted chat and creating a new live chat.

### End

`end` permanently ends a live chat and releases its live slot. The SDK sends END to the Query node that owns the chat.

END does not delete a paused chat. The SDK must reject `end()` in Paused state and must not
implicitly resume it. The caller may explicitly resume and end the chat, or let it expire.

After END succeeds, the `Chat` can no longer be used. The SDK must reject further server operations, and `handle()` and `state()` are no longer valid for that object.

Handles that were serialized earlier cannot be revoked. Applications should therefore discard saved handles after successfully ending a chat.

### Ambiguous Outcomes

If the SDK sends PAUSE but loses the response after the request may have reached Query, it cannot know whether the chat is still live or was successfully paused. The SDK therefore marks the chat `Indeterminate`.

The handle keeps the chat id, Knowledge choice, and the Query node that owned the chat before PAUSE. That node becomes the target for resolving the ambiguous PAUSE. If the SDK knows the request failed before it could reach Query, the chat remains `Active` instead.

From `Indeterminate`, the caller may explicitly call `resume()` or `end()`. Both operations must target the Query node retained from the failed PAUSE. The SDK must not search other Query nodes, because another node cannot establish what happened to the PAUSE request on the original owner.

If that Query node can no longer be resolved from the current topology, the SDK returns a routing error and keeps the chat `Indeterminate`.

A successful RESUME makes the chat `Active`; a successful END ends it. There is one special case: if RESUME returns 19255 from the retained Query node, that response proves the chat is already live there. The SDK therefore marks the chat `Active` with that node as its owner, while still returning the 19255 error to the caller.

Any other server error from RESUME or END leaves the chat `Indeterminate`. In particular, 19236 from END does not prove that PAUSE succeeded; it only says that the retained Query node does not currently have a live chat for that id.

RESUME and END are explicit recovery actions, not status checks. The SDK must not automatically probe other nodes, retry the operation, or try to reconcile the server state.

If the response to RESUME or END is itself lost, the SDK keeps the previous `ChatState` and returns the normal transport error. The [non-replay rule](#retries) still applies.

If the response to BEGIN is lost and the SDK never receives the chat id, it cannot return a usable `Chat`.

## Chat Handle and Re-entrancy

`ChatHandle` contains the SDK state needed to continue a chat later. It is opaque to the application, contains no credentials, and can be serialized so the chat can be continued from another process or SDK instance.

`chat(handle)` restores a `Chat` from that saved state. It is a local operation: it does not contact Query and must not issue RESUME.

The handle must retain:

* chat identity;
* the Couchbase cluster's `clusterUUID`;
* the Query node needed to route a live chat;
* the chat's Knowledge choice;
* the last-known `ChatState`;
* for an `Indeterminate` chat, the retained PAUSE target and ambiguity marker.

A handle can only be used with the cluster that created it. If the attaching `Cluster` already knows its `clusterUUID`, the SDK must reject a handle from a different cluster immediately. If the UUID is not yet known, the SDK may defer the check until the first operation that contacts the server. It must validate the UUID before sending that operation, but `chat(handle)` itself must not perform server I/O just to obtain it.

Serialized handles are snapshots; applications should save the latest handle after lifecycle changes.
The SDK must not repair stale handles by probing Query or interpreting 19236 as a state transition.

Each SDK must support JSON serialization and reconstruction of the complete, opaque `ChatHandle`
without requiring application access to its internal fields.

`ChatHandle` must be portable across processes and SDK instances. Its serialized format is SDK-specific and may evolve between versions, so implementations should be able to recognize unsupported handle formats.

A handle must not contain Couchbase credentials, model credentials, model options, a resolved Query endpoint, or turn-specific hints. Model configuration and credentials come from the `ConversationalQuery` used when the handle is attached. Keyspaces and inactivity settings remain part of the server-side chat state.

## Knowledge

Knowledge entries are notes associated with a Couchbase keyspace. They belong to the target bucket, so deleting that bucket also deletes its Knowledge entries.

`getAll` returns the full text of every entry the caller can read, optionally filtered by keyspace.
It is not paginated in this revision.

Creating, replacing, or deleting Knowledge requires the admin role. Listing is limited by the caller's access to the corresponding bucket. Knowledge should not be used to store secrets.

`QueryOptions.knowledge` controls whether Knowledge may be used for a one-shot query. `BeginChatOptions.knowledge` controls it for a chat. Both default to `false`.

For a chat, the SDK stores this choice and sends it with every `ask`. Query consults Knowledge only
when creating or rebuilding the model prompt; BEGIN does not create one. Updating Knowledge therefore
does not affect a chat using an existing prompt. A non-summarizing PAUSE preserves that prompt;
after a summarizing PAUSE and RESUME, Query may rebuild it using current Knowledge entries.

If Query fails to collect Knowledge while building a prompt, the request may still succeed without a Knowledge-specific warning.

# Implementation Details

## Node Affinity

A live chat is held in memory by one Query node. The node that serves BEGIN or RESUME becomes the chat's owner. Query does not route requests to the owner by chat id, so the SDK must send ASK, PAUSE, and END back to that node.

The SDK stores the owner's logical node identity, not the Query endpoint used for the original request. The logical identity is Query's canonical node name: the canonical hostname plus the default-network management port.

Before each affine operation, resolve the retained owner using normal SDK topology handling.

If the owner is absent or has no usable Query endpoint, return a routing error. The SDK must not
search or fail over to another Query node: the live chat exists only in memory on its owner.

The SDK must identify the node that served a successful BEGIN or RESUME before returning an Active
`Chat`; Query does not include the owner in its response.

Chat operations are unsupported through a proxy that prevents the SDK from identifying the serving Query node. Alternate addresses are supported as long as serving-node attribution is preserved.

## Wire Mapping

The statements below encode options as a JSON object after `WITH`. For `USING AI`, the
natural-language prompt follows that object as plain text.

For example:

```text
USING AI WITH {"keyspaces":["travel-sample.inventory.hotel"],"knowledge":true,"execute":false,"output":"SQL"} Find hotels in Paris
```

### Operations

| Operation   | Statement                          | `WITH` fields                                                  | Model configuration and credential                            |
| ----------- | ---------------------------------- | -------------------------------------------------------------- | ------------------------------------------------------------- |
| `query`     | `USING AI WITH <options> <prompt>` | `keyspaces`, `knowledge`, optional `hint`, `execute`, `output` | Send the effective model configuration and credential.        |
| `beginChat` | `BEGIN CHAT WITH <options>`        | `keyspaces`, optional `timeout`                                | Omit.                                                         |
| `ask`       | `USING AI WITH <options> <prompt>` | Stored `knowledge`, optional `hint`, `execute`, `output`       | Send the Chat's effective model configuration and credential. |
| `pause`     | `PAUSE CHAT WITH <options>`        | Optional `summarize`                                           | Send when `summarize` is unset or `true`; omit when `false`.  |
| `resume`    | `RESUME CHAT`                      | None                                                           | Omit.                                                         |
| `end`       | `END CHAT`                         | None                                                           | Omit.                                                         |

For every chat operation except BEGIN, the SDK sends the stored chat id as the string request parameter `natural_chatid`. The chat id must not also appear in the statement, where it would take precedence over `natural_chatid`.

An unset `summarize` may be encoded as:

```text
PAUSE CHAT WITH {}
```

SDKs must use the `WITH` fields defined here for `hint`, `execute`, and `output`. They must not also send the corresponding `natural_*` request parameters.

Knowledge has no separate request-parameter form.

### `WITH` Fields

| Field       | Encoding                                                                                                                                           |
| ----------- | -------------------------------------------------------------------------------------------------------------------------------------------------- |
| `keyspaces` | Array of keyspace-path strings, for example `["travel-sample.inventory.hotel"]`. Sent only by `query` and BEGIN.                                   |
| `knowledge` | JSON boolean. Defaults to `false`. Send the one-shot value on `query` and the stored chat value on every `ask`.                                    |
| `hint`      | JSON string when supplied; otherwise omit.                                                                                                         |
| `execute`   | JSON boolean. Defaults to `true`; omitting it uses the server default.                                                                             |
| `output`    | `Sqlpp` maps to `"SQL"`, `FtsSqlpp` to `"FTSSQL"`, and `JsUdf` to `"JSUDF"`. Omit when unset; the default is SQL output.                                             |
| `timeout`   | Inactivity timeout in whole seconds, separate from the Query request timeout. Omit when unset. |
| `summarize` | JSON boolean when explicitly `true` or `false`; omit when unset.                                                                                   |

Keyspaces use SQL++ path notation. SDK documentation should recommend fully qualified `bucket.scope.collection` paths. Other keyspace validation is left to Query.

### Model Options

Credential parameters are defined under [Model Credentials](#model-credentials).

| `ModelOptions`     | Capella path     | Direct-provider path                          |
| ------------------ | ---------------- | --------------------------------------------- |
| `provider`         | `natural_vendor` | `natural_config.provider`                     |
| `name`             | `natural_model`  | `natural_config.model`                        |
| `endpoint`         | Unsupported      | `natural_config.endpoint`                     |
| `region`           | Unsupported      | `natural_config.region`                       |
| `outputTokenLimit` | Unsupported      | `natural_config.output_token_limit`           |
| `moderation`       | Unsupported      | `natural_config.moderation`                   |
| `raw`              | Unsupported      | Additional top-level `natural_config` entries |

### Knowledge Statements

Knowledge operations use ordinary Query requests, without model parameters or chat affinity.

```text
create: CREATE KNOWLEDGE <name> FOR <keyspace> AS $value
upsert: CREATE OR REPLACE KNOWLEDGE <name> FOR <keyspace> AS $value
drop:   DROP KNOWLEDGE <name> FOR <keyspace>

getAll:
SELECT k.name, k.`bucket`, k.`scope`, k.`collection`, k.`value`
FROM system:natural_knowledge AS k
```

#### Name and keyspace encoding

`name` is supplied by the application as a plain string. The SDK encodes the entire value as one SQL++ identifier and doubles any embedded backticks. Applications must not pre-quote it.

`keyspace` is either:

* a bucket name, such as `travel-sample`; or
* a three-part `bucket.scope.collection` path, such as `travel-sample.inventory.hotel`.

The SDK must reject empty components, any other number of components, namespace prefixes, and pre-quoted identifiers.

For DDL, escape each keyspace component separately, for example:

```text
`travel-sample`.`inventory`.`hotel`
```

A bucket-only keyspace refers to that bucket's `_default._default` collection, but the DDL contains only the escaped bucket name.

Dots are used as component separators, so this API cannot address a bucket, scope, or collection whose name itself contains a dot.

Other keyspace validity checks are left to Query.

#### Knowledge text

The SDK must bind Knowledge text as the named parameter `$value`.

#### Listing entries

`getAll()` uses the projection above, escaping the reserved catalog field names. Construct
`KnowledgeEntry.keyspace` by joining `bucket`, `scope`, and `collection` with dots. A bucket-only
input such as `travel-sample` is therefore returned as `travel-sample._default._default`; both
forms identify the same entry.

Any `KnowledgeEntry.keyspace` returned by `getAll()` must be accepted unchanged by other `KnowledgeManager` operations, except when one of its component names contains a dot, which this string form cannot represent unambiguously.

When `getAll(keyspace)` includes a keyspace filter, add:

```text
WHERE k.`bucket` = $bucket
  AND k.`scope` = $scope
  AND k.`collection` = $collection
```

Bind the individual component values as Query parameters. For a bucket-only input, use `_default` for both scope and collection. The SDK must compare these components directly rather than constructing a namespace-qualified keyspace.

#### Create and drop behavior

The SDK must use the single-entry statement forms shown above.

`create` uses `CREATE KNOWLEDGE` without `IF NOT EXISTS`. If the entry already exists, Query returns error 20105.

`drop` uses `DROP KNOWLEDGE` without `IF EXISTS`. If the entry does not exist, Query returns error 20103.

Other server-supported Knowledge grammar is outside this SDK API.

## Results

`ConversationalQueryResult` extends normal Query result handling with the metadata below.

### Generated statement

When Query generates SQL++, the response includes a top-level `generated_statement` field. The SDK exposes it as:

```text
generatedStatement() -> optional<string>
```

The generated statement must be available without iterating over result rows.

In responses that contain rows, `generated_statement` appears before `results`. Other metadata fields may appear before it, so the parser must not depend on a fixed overall field order.

The parser must also accept valid responses that contain no `results` or `signature`, such as generation without execution and chat lifecycle responses.

### Chat metadata

BEGIN returns a top-level string `chatId`. The SDK stores it together with the Query node that served BEGIN before returning the `Chat`.

### Token usage

Conversational Query can return:

```text
requestTokens() -> optional<TokenUsage>
chatTokens()    -> optional<TokenUsage>

TokenUsage {
    promptTokens int
    completionTokens int
    totalTokens int
}
```

`requestTokens` and `chatTokens` are optional top-level response fields using the case-sensitive
names shown above.

`requestTokens` reports token usage for the current request.

`chatTokens` reports cumulative token usage for the chat and, when present, must also be exposed in metadata returned from PAUSE, RESUME, and END.

Both fields are optional. Query may omit an all-zero token object, and the Capella path does not report token usage. An absent value therefore means only that token usage was not reported; the SDK must not replace it with an all-zero `TokenUsage`.

The SDK must use the `totalTokens` value returned by Query. It must not calculate it from `promptTokens` and `completionTokens` or reject the response if those values do not add up.

`naturalLanguageProcessingTime` remains part of the normal Query metrics object and may be exposed through the SDK's existing Query metrics type.

### Execution

`execute(true)` requests execution subject to Query's restrictions; a successful response does not
guarantee that the generated SQL++ was executed.

`JsUdf` generates a JavaScript UDF as a `CREATE FUNCTION` statement. Query does not execute the
generated statement; the application must execute it separately to create the function.

This revision exposes no explicit execution-status field. The SDK must therefore not infer whether execution occurred from the presence of a generated statement, result rows, metrics, or a `signature`.

## Errors

Except for the rules below, normal Query error handling (RFC 0058) applies.

Preserve the server code, message, and nested `reason` in the Query error context, subject to
credential redaction. Some Conversational Query errors carry useful detail only in `reason`;
SDKs must not rely on `msg` alone or rewrite server error text.

| Code  | SDK handling                                                                                                                                     |
| ----- | ------------------------------------------------------------------------------------------------------------------------------------------------ |
| 19220 | `RateLimited` (RFC 0058, #21)                                                                                                                    |
| 19221 | `UnambiguousTimeout` (RFC 0058, #14)                                              |
| 19222 | `FeatureNotAvailable` (RFC 0058, #15)                                                                                                            |
| 19236 | No-live-chat error. Preserve the code and message; do not infer a particular `ChatState`.                                                        |
| 19240 | `QuotaLimited` (RFC 0058, #22), including when 19240 appears inside the `reason` of a 19243 RESUME error.                                        |
| 19241 | `AuthenticationFailure` (RFC 0058, #6)                                                                                                           |
| 19243 | Preserve as a Query error with its server context, except for the nested-19240 case above. It does not prove that the persisted chat is missing. |
| 19248 | Preserve as a Query error with its server context. Credential-resolution details may appear only in `reason`.                                    |
| 19249 | Moderation-rejection error.                                                                                                                      |
| 19255 | Preserve as a Query error. It changes local chat state only in the RESUME case described under [Ambiguous Outcomes](#ambiguous-outcomes).        |

When 19240 is nested inside a 19243 error, map the error to `QuotaLimited` but retain both the outer 19243 code and the nested `reason` in the error context.

Error 19236 means only that the Query node receiving the request does not currently have a live chat with that id. The chat may be on a different node, paused, ended, expired, or unknown. The SDK must therefore not translate 19236 into "invalid chat id" or use it to change `ChatState`.

HTTP status alone does not establish `ChatState`.

## Retries

The SDK must not automatically retry a Conversational Query request once it may have reached Query.

A request may already have called the model, changed chat history, or executed generated SQL++ before an error is returned. Replaying it could therefore repeat work or change state twice. This applies even when `execute(false)` is used or the generated Query is marked `readonly`.

The SDK may use normal retry handling (RFC 0049) only when the request is known not to have reached Query.
For a chat operation, any such retry must target the same Query node or retained PAUSE target.

Query performs its own retries for model and generation work. SDKs must not add another generic retry loop for model-provider failures.

Conversational Query-specific 192xx errors omit Query's `retry` flag; ordinary Query errors from
executing generated SQL++ may include it. The SDK should preserve and handle the flag under normal
Query error rules, but it must never override the Conversational Query non-replay rule.

## Feature Detection

Chat support is advertised by `conversationalQuery` in `clusterCapabilities.n1ql`.

If `conversationalQuery` is absent, `beginChat` and any `Chat` operation that would contact Query must fail locally with `FeatureNotAvailableException`.

The SDK capability check does not apply to local operations (`chat(handle)`, handle parsing or serialization),
one-shot `query`, or `KnowledgeManager` operations. Unsupported Knowledge operations use normal
Query error handling, including 19222 when the natural-language feature is disabled.

SDKs must not add a separate version check for the direct-provider path.

The capability is cluster-wide; natural-language feature control is per Query node. A node may
therefore return 19222 even when chat support is advertised. For affine operations, return the
error without rerouting: another enabled node cannot serve the live chat.

Because the feature-control check happens before chat lookup, a 19222 response also does not tell the SDK whether the chat exists.

# Documentation

SDK documentation should cover one-shot queries, chat lifecycle and persistence, `ChatHandle` usage, Knowledge, and generation without execution. It should also describe the user-visible consequences of Indeterminate PAUSE outcomes, stale handles, chat expiry, Knowledge prompt reuse, and serialized access to a chat.

Model-credential documentation must describe the prerequisites for each credential form. In particular:

* `capella(email, password, organizationId)` uses a Capella console login rather than an organization API key, requires accepted iQ terms, and is not available to accounts that can authenticate only through SSO or social login.
* `stored(credentialName)` requires node-to-node encryption on all nodes or `n2nEncryptionOverride=true` in `/settings/credentialStore`.

Operator documentation should reference server configuration for the credential store, chat persistence, model connectivity, and resource limits. It should also note that summarization can add model latency and token cost, and that chat operations require the SDK to identify the serving Query node.

Data-handling documentation must describe the Couchbase authorization boundary and the information that may be sent to model providers. Chat context can include prompts, generated statements and Query correction rounds, but not result rows. The `slm` provider can additionally include representative sample values. Knowledge belongs to its target bucket and is not currently covered by backup and restore ([MB-73627](https://jira.issues.couchbase.com/browse/MB-73627)).

# Changelog

* Revision 3 - 2026-09-10 (by Anirudh Lakhotia)

  * Reworked the draft to follow the SDK RFC structure and consolidated the proposed public API.
  * Added Knowledge management, feature detection, and the `Indeterminate` PAUSE state.
  * Consolidated model credential and option handling.
  * Removed superseded background, requirements, questions, deployment prerequisites, and wire-example sections.

* Revision 2 - 2026-09-02 (by Jared Casey)

  * Expanded the original draft with chat lifecycle, credential types, and result handling.

* Revision 1 - 2026-08-31 (by Jared Casey)

  * Original draft.

# Signoff

Before signoff, each SDK team must confirm that its Query dispatch path can identify the specific node
that served a successful BEGIN or RESUME. If it cannot, the team must identify the prerequisite core
change before implementation; the public chat contract depends on recording that node as the live
owner.

| Language | Team Member | Signoff Date | Revision |
|---|---|---|---|
| .NET | | | |
| C/C++ | | | |
| Go | | | |
| Java | | | |
| Kotlin | | | |
| Node.js | | | |
| PHP | | | |
| Python | | | |
| Ruby | | | |
| Scala | | | |
