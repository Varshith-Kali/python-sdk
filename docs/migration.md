# Migration Guide: v1 to v2

This guide covers the breaking changes in v2 of the MCP Python SDK and how to update code written against v1.x.

## Find your changes

Every section heading below names the API it affects, so searching this page for the symbol your code uses is the fastest route to the change that broke it.

### Changes almost every project hits

| Change | First symptom | Section |
|---|---|---|
| `FastMCP` renamed to `MCPServer` | `ModuleNotFoundError: No module named 'mcp.server.fastmcp'` | [`FastMCP` renamed](#fastmcp-renamed-to-mcpserver) |
| Fields renamed from camelCase to snake_case | `AttributeError: 'Tool' object has no attribute 'inputSchema'` | [snake_case fields](#field-names-changed-from-camelcase-to-snake_case) |
| `mcp.types` moved to the `mcp-types` package | `ModuleNotFoundError: No module named 'mcp.types'` | [`mcp.types` moved](#mcptypes-moved-to-the-mcp-types-package) |
| `McpError` renamed to `MCPError` | `ImportError: cannot import name 'McpError' from 'mcp'` | [`McpError` renamed](#mcperror-renamed-to-mcperror) |
| Resource URIs are `str`, not `AnyUrl` | `AttributeError: 'str' object has no attribute 'host'` | [URI type](#resource-uri-type-changed-from-anyurl-to-str) |
| `streamablehttp_client` removed | `ImportError: cannot import name 'streamablehttp_client'` | [`streamablehttp_client`](#streamablehttp_client-removed) |
| Transport parameters moved off the `MCPServer` constructor | `TypeError: MCPServer.__init__() got an unexpected keyword argument 'port'` | [transport parameters](#transport-parameters-moved-from-the-mcpserver-constructor-to-run) |
| Sync handlers run on a worker thread | `asyncio.get_running_loop()` in a `def` handler raises `RuntimeError` | [worker threads](#sync-handler-functions-now-run-on-a-worker-thread) |
| Lowlevel decorators replaced with `on_*` constructor params | `AttributeError: 'Server' object has no attribute 'list_tools'` | [`on_*` handlers](#lowlevel-server-decorator-based-handlers-replaced-with-constructor-on_-params) |
| Lowlevel return value wrapping removed | bare list or dict returns fail result validation instead of being wrapped | [wrapping removed](#lowlevel-server-automatic-return-value-wrapping-removed) |
| Lowlevel tool exceptions no longer become `isError: true` results | clients raise a JSON-RPC error instead of seeing the error text | [tool exceptions](#lowlevel-server-tool-handler-exceptions-no-longer-become-error-results) |

### Find your area

| If you... | Read |
|---|---|
| pin dependencies or use the `mcp` CLI | [Packaging, dependencies, and CLI](#packaging-dependencies-and-cli) |
| import `mcp.types` or touch protocol types | [Types and wire format](#types-and-wire-format) |
| run `FastMCP`/`MCPServer` servers | [MCPServer (formerly FastMCP)](#mcpserver-formerly-fastmcp) |
| use the lowlevel `Server` | [Lowlevel Server](#lowlevel-server), plus [Timeouts take `float` seconds](#timeouts-take-float-seconds-instead-of-timedelta) and [Experimental Tasks support removed](#experimental-tasks-support-removed) under Clients |
| write client code with `Client` or `ClientSession` | [Clients](#clients), plus [`streamablehttp_client` removed](#streamablehttp_client-removed) under Transports |
| use stdio or streamable HTTP directly, or maintain a custom transport | [Transports](#transports) |
| maintain OAuth client auth or a protected server | [OAuth and server auth](#oauth-and-server-auth) |
| relied on lenient handling of off-schema traffic, or assert on exact wire bytes | [Stricter protocol validation and wire behavior](#stricter-protocol-validation-and-wire-behavior) |
| test against in-memory server/client pairs | [Testing utilities](#testing-utilities) |

## Suggested migration order

1. Update your dependency pins: [Packaging, dependencies, and CLI](#packaging-dependencies-and-cli).
2. Apply the mechanical renames and import moves: [Types and wire format](#types-and-wire-format).
3. Port your server surface: [MCPServer (formerly FastMCP)](#mcpserver-formerly-fastmcp) or [Lowlevel Server](#lowlevel-server).
4. Port your client code: [Clients](#clients).
5. Update transport setup and auth: [Transports](#transports) and [OAuth and server auth](#oauth-and-server-auth).
6. Run your tests and check anything that now errors against [Stricter protocol validation and wire behavior](#stricter-protocol-validation-and-wire-behavior) and [Testing utilities](#testing-utilities).

## Packaging, dependencies, and CLI

### Dependency requirements changed

v2 raises several dependency floors and drops or adds packages: a pin or cap in your project that conflicts with a new floor makes dependency resolution fail, and code that imported a dropped package without declaring it must now depend on it directly.

| Dependency | v1.x | v2 | Change |
|---|---|---|---|
| anyio | `>=4.5` | `>=4.9` (`>=4.10` on Python 3.14+) | floor raised |
| pydantic | `>=2.11,<3` (`>=2.12,<3` on Python 3.14+) | `>=2.12` | floor raised on Python <3.14; `<3` cap dropped |
| sse-starlette | `>=1.6.1` | `>=3.0.0` | floor raised to 3.x |
| typing-extensions | `>=4.9.0` | `>=4.13.0` | floor raised |
| pywin32 (Windows) | `>=310` (`>=311` on Python 3.14+) | `>=311` | floor raised on Python <3.14 |
| httpx | `>=0.27.1,<1.0.0` | removed | see [`httpx` and `httpx-sse` replaced by `httpx2`](#httpx-and-httpx-sse-replaced-by-httpx2) |
| httpx-sse | `>=0.4` | removed | see above |
| pydantic-settings | `>=2.5.2` | removed | no longer installed with `mcp`; declare it in your own project if you import `pydantic_settings` |
| websockets (`ws` extra) | `>=15.0.1` | removed | see [WebSocket transport removed](#websocket-transport-removed) |
| httpx2 | not a dependency | `>=2.5.0` | new required dependency |
| mcp-types | not a dependency | pinned to the exact `mcp` version | new required dependency; do not pin it separately from `mcp` |
| opentelemetry-api | not a dependency | `>=1.28.0` | new required dependency; tracing is a no-op until you install an OpenTelemetry SDK ([OpenTelemetry](run/opentelemetry.md)) |

Relax any conflicting pin. sse-starlette moves to 3.x, so code that imports `sse_starlette` directly must also port to its 3.x API.

### `httpx` and `httpx-sse` replaced by `httpx2`

The SDK depends on [`httpx2`](https://pypi.org/project/httpx2/) (a fork of `httpx` with server-sent events built in) instead of `httpx` and `httpx-sse`, and no longer installs either. Every `httpx` type in the client API is now the `httpx2` equivalent: `http_client=` on `streamable_http_client` (and anything your `httpx_client_factory` returns) is an `httpx2.AsyncClient`, `auth=` on `sse_client` is an `httpx2.Auth`, and `OAuthClientProvider` is an `httpx2.Auth`, so it goes on `httpx2.AsyncClient(auth=...)`. If you built the client only for default behavior, `Client("http://.../mcp")` needs no HTTP client at all; see [Client transports](client/transports.md).

**Before (v1):**

```python
import httpx

http_client = httpx.AsyncClient(headers={"Authorization": "Bearer <token>"})
try:
    ...
except httpx.ConnectError:
    ...
```

**After (v2):**

```python
import httpx2

http_client = httpx2.AsyncClient(headers={"Authorization": "Bearer <token>"})
try:
    ...
except httpx2.ConnectError:
    ...
```

`httpx2` keeps the `httpx` API, so usually only the import name changes; direct `httpx-sse` usage becomes `httpx2.EventSource` or `AsyncClient.sse()`. The transports now raise `httpx2` exceptions (`httpx2.ConnectError`, `httpx2.HTTPStatusError`, and so on), so an old `except httpx.ConnectError:` clause either fails at import time or, if `httpx` is still installed for another reason, silently never matches — audit `except` clauses and `isinstance` checks along with the imports. Log filters and instrumentation keyed to the `httpx`/`httpcore` names (a `logging.getLogger("httpx")` suppression, OpenTelemetry's httpx instrumentation) no longer see the SDK's traffic: the loggers are `httpx2`/`httpcore2.*` and the default User-Agent is `python-httpx2/<version>`.

TLS verification changes too: `httpx` validated against the bundled `certifi` CA list, while `httpx2` validates against the operating-system trust store via [`truststore`](https://pypi.org/project/truststore/). With no usable system CA store (some minimal containers), point `SSL_CERT_FILE` or `SSL_CERT_DIR` at a CA bundle (`httpx2` checks them before the system store) or pass `verify=ssl_context` to your `httpx2.AsyncClient`.

## Types and wire format

### `mcp.types` moved to the `mcp-types` package

The protocol wire types now live in a separate distribution, `mcp-types`, imported as
`mcp_types`; `mcp` depends on it, so it installs automatically. The `mcp.types` module is
gone: `import mcp.types as types` and `from mcp import types` become `import mcp_types as types`,
and `from mcp.types import X` becomes `from mcp_types import X`. Type names re-exported at the
top level of `mcp` are unchanged. The generated type reference is at
[`mcp_types`](api/mcp_types/index.md).

**Before (v1):**

```python
import mcp.types as types
from mcp.types import CallToolResult, TextContent
from mcp import Resource, Tool
```

**After (v2):**

```python
import mcp_types as types
from mcp_types import CallToolResult, TextContent
from mcp import Resource, Tool  # unchanged: still re-exported by `mcp`
```

`mcp.shared.version` is gone as well; its version constants live in `mcp_types.version` now,
with changed values and meaning: see
[`LATEST_PROTOCOL_VERSION` and `SUPPORTED_PROTOCOL_VERSIONS` changed](#latest_protocol_version-and-supported_protocol_versions-changed).

### Removed type aliases and classes

These `mcp.types` names have no `mcp_types` equivalent under the same name:

| Removed | Replacement |
|---------|-------------|
| `Content` | `ContentBlock` |
| `ResourceReference` | `ResourceTemplateReference` |
| `Cursor` | `str` |
| `AnyFunction` | `Callable[..., Any]` |
| `MethodT`, `RequestParamsT`, `NotificationParamsT` | internal `TypeVar`s; no longer exported |
| `ClientRequestType`, `ClientNotificationType`, `ClientResultType`, `ServerRequestType`, `ServerNotificationType`, `ServerResultType` | the union is now the bare name (`ClientRequest`, `ClientNotification`, `ClientResult`, `ServerRequest`, `ServerNotification`, `ServerResult`); see [`RootModel` message unions](#rootmodel-message-unions-replaced-by-plain-unions-and-typeadapters) |
| `TaskExecutionMode`, `TASK_FORBIDDEN`, `TASK_OPTIONAL`, `TASK_REQUIRED`, `TASK_STATUS_*` | string literals; `TaskStatus` remains as the literal-union type (see [Experimental Tasks support removed](#experimental-tasks-support-removed)) |

**Before (v1):**

```python
from mcp.types import TASK_REQUIRED, ToolExecution

execution = ToolExecution(taskSupport=TASK_REQUIRED)
```

**After (v2):**

```python
from mcp_types import ToolExecution

execution = ToolExecution(task_support="required")
```

### Field names changed from camelCase to snake_case

Every field on the `mcp_types` models is now snake_case in Python. The JSON wire format is unchanged: the SDK still sends camelCase through Pydantic aliases.

**Before (v1):**

```python
result = await session.call_tool("my_tool", {"x": 1})
if result.isError:
    ...

tools = await session.list_tools()
cursor = tools.nextCursor
schema = tools.tools[0].inputSchema
```

**After (v2):**

```python
result = await session.call_tool("my_tool", {"x": 1})
if result.is_error:
    ...

tools = await session.list_tools()
cursor = tools.next_cursor
schema = tools.tools[0].input_schema
```

Common renames:

| v1 (camelCase) | v2 (snake_case) |
|-----------------|-----------------|
| `inputSchema` | `input_schema` |
| `outputSchema` | `output_schema` |
| `isError` | `is_error` |
| `nextCursor` | `next_cursor` |
| `mimeType` | `mime_type` |
| `structuredContent` | `structured_content` |
| `serverInfo` | `server_info` |
| `protocolVersion` | `protocol_version` |
| `uriTemplate` | `uri_template` |
| `listChanged` | `list_changed` |
| `progressToken` | `progress_token` |

Rename attribute access and constructor kwargs alike: `tool.input_schema`, `Tool(input_schema={...})`. A leftover camelCase kwarg such as `Tool(inputSchema={...})` still runs but fails type checking. Parsing is unaffected: `model_validate()` accepts camelCase wire JSON and snake_case dumps alike.

**If you serialize models yourself, pass `by_alias=True`.** `model_dump()` and `model_dump_json()` now emit snake_case keys, silently the wrong shape for any MCP peer:

```python
tool.model_dump()                                # {"name": ..., "input_schema": ...}
tool.model_dump(by_alias=True, mode="json")      # {"name": ..., "inputSchema": ...}  (wire format)
```

### Extra fields on MCP types are no longer preserved

v1 protocol models set `extra="allow"`, so unknown fields passed to a constructor or received from a peer were stored and re-serialized. v2 models silently drop unknown fields during validation: the data no longer round-trips, and reading it as an attribute (`msg.customField`) raises `AttributeError`. Carry custom data in `_meta` instead.

**Before (v1):**

```python
from mcp.types import CallToolResult

result = CallToolResult.model_validate({"content": [], "customField": "x"})
result.model_dump()["customField"]  # 'x'
```

**After (v2):**

```python
from mcp_types import CallToolResult, TextContent

result = CallToolResult.model_validate({"content": [], "customField": "x"})
"customField" in result.model_dump()  # False

result = CallToolResult(
    content=[TextContent(type="text", text="ok")],
    _meta={"myapp/custom": "x"},
)
result.meta  # {'myapp/custom': 'x'}
```

### Resource URI type changed from `AnyUrl` to `str`

The `uri` field on resource types and the `uri` parameter of the client's `read_resource()`, `subscribe_resource()`, and `unsubscribe_resource()` are now plain `str`. Passing an `AnyUrl` object raises `ValidationError` ("Input should be a valid string"); wrap it in `str(...)`. Relative URIs such as `users/me`, which v1 rejected, are now accepted.

**Before (v1):**

```python
from pydantic import AnyUrl
from mcp.types import Resource

resource = Resource(name="test", uri=AnyUrl("resource://items/1"))
scheme = resource.uri.scheme
result = await session.read_resource(AnyUrl("resource://items/1"))
```

**After (v2):**

```python
from urllib.parse import urlparse

from mcp_types import Resource

resource = Resource(name="test", uri="resource://items/1")
scheme = urlparse(resource.uri).scheme
result = await session.read_resource("resource://items/1")
```

Affected `uri` fields: `Resource` (and `ResourceLink`), `ResourceContents` (`TextResourceContents`, `BlobResourceContents`), `ReadResourceRequestParams`, `SubscribeRequestParams`, `UnsubscribeRequestParams`, `ResourceUpdatedNotificationParams`, and the `MCPServer` resource classes (`TextResource`, `FileResource`, ...).

v2 also stops normalizing URIs during validation: v1 sent `https://example.com` as `https://example.com/`, v2 sends the string exactly as given.

### `RootModel` message unions replaced by plain unions and `TypeAdapter`s

`ClientRequest`, `ServerRequest`, `ClientNotification`, `ServerNotification`, `ClientResult`, `ServerResult`, and `JSONRPCMessage` are now plain `X | Y | ...` unions in `mcp_types`, not `RootModel` subclasses. `.root`, `model_validate()`, and the wrapper call are gone: validate with the matching `TypeAdapter`, and narrow with `isinstance` against the concrete member type instead of matching on `.root`.

**Before (v1):**

```python
from mcp.types import ClientRequest, ServerNotification

request = ClientRequest.model_validate(data)
actual_request = request.root

notification = ServerNotification.model_validate(data)
actual_notification = notification.root
```

**After (v2):**

```python
from mcp_types import client_request_adapter, server_notification_adapter

request = client_request_adapter.validate_python(data)  # already the concrete type
notification = server_notification_adapter.validate_python(data)
```

Constructing a value no longer takes the wrapper call:

**Before (v1):**

```python
from mcp.types import ClientNotification, ClientRequest, InitializedNotification, ListToolsRequest, ListToolsResult

await session.send_notification(ClientNotification(InitializedNotification()))
result = await session.send_request(ClientRequest(ListToolsRequest()), ListToolsResult)
```

**After (v2):**

```python
from mcp_types import InitializedNotification, ListToolsRequest, ListToolsResult

await session.send_notification(InitializedNotification())
result = await session.send_request(ListToolsRequest(), ListToolsResult)
```

| Union | Adapter (exported from `mcp_types`) |
|-------|--------------------------------------|
| `ClientRequest` | `client_request_adapter` |
| `ServerRequest` | `server_request_adapter` |
| `ClientNotification` | `client_notification_adapter` |
| `ServerNotification` | `server_notification_adapter` |
| `ClientResult` | `client_result_adapter` |
| `ServerResult` | `server_result_adapter` |
| `JSONRPCMessage` | `jsonrpc_message_adapter` |

### `RequestParams.Meta` replaced with the `RequestParamsMeta` TypedDict

The nested `RequestParams.Meta` Pydantic model is now the `RequestParamsMeta` TypedDict (`from mcp_types import RequestParamsMeta`), and the nested `NotificationParams.Meta` class is gone: notification `_meta` is a plain `dict[str, Any]`. `ctx.meta` (lowlevel handlers) and `ctx.request_context.meta` (`MCPServer` tools) are dicts, so attribute access becomes key access, `progressToken` becomes the optional `progress_token` key, and `_meta` you built with `types.RequestParams.Meta(...)` or `types.NotificationParams.Meta(...)` is passed as a plain dict. The wire format is unchanged.

If you only read `progressToken` to send progress, call `report_progress()` instead: it reports to the caller and is a no-op when the caller didn't ask for progress (see [Progress](handlers/progress.md)).

**Before (v1):**

```python
ctx = server.request_context
if ctx.meta and ctx.meta.progressToken:
    await ctx.session.send_progress_notification(ctx.meta.progressToken, 0.5, 100)
```

**After (v2):**

```python
await ctx.session.report_progress(0.5, 100)  # lowlevel handler, ctx: ServerRequestContext
# in an MCPServer tool with ctx: Context:
await ctx.report_progress(0.5, 100)
```

Other `_meta` keys become dictionary lookups (`meta` is still `None` when the request carries no `_meta`): `ctx.meta.traceparent` becomes `(ctx.meta or {}).get("traceparent")`.

### `LATEST_PROTOCOL_VERSION` and `SUPPORTED_PROTOCOL_VERSIONS` changed

Both constants moved from `mcp.shared.version` to `mcp_types.version` (see [`mcp.types` moved to the `mcp-types` package](#mcptypes-moved-to-the-mcp-types-package)), and both changed under the same name:

- `LATEST_PROTOCOL_VERSION` was `"2025-11-25"`, the version the v1 client offered in `initialize`. In v2 it is the newest revision the SDK speaks, `"2026-07-28"`, which the `initialize` handshake never negotiates. Where your code used it as the handshake version (a hand-built `initialize`, or a comparison against the negotiated version), use `LATEST_HANDSHAKE_VERSION` (`"2025-11-25"`).
- `SUPPORTED_PROTOCOL_VERSIONS` is now a `tuple` (was a `list`) with `"2026-07-28"` appended: membership tests still pass, but `[-1]` is now `"2026-07-28"` and list operations such as `+ [...]` raise `TypeError`. The handshake-only set is `HANDSHAKE_PROTOCOL_VERSIONS`.

**Before (v1):**

```python
from mcp.shared.version import LATEST_PROTOCOL_VERSION, SUPPORTED_PROTOCOL_VERSIONS
```

**After (v2):**

```python
from mcp_types.version import HANDSHAKE_PROTOCOL_VERSIONS, LATEST_HANDSHAKE_VERSION

# LATEST_HANDSHAKE_VERSION == "2025-11-25", the value v1 called LATEST_PROTOCOL_VERSION
```

### `McpError` renamed to `MCPError`

Hard rename, no compatibility alias; import from `mcp` or `mcp.shared.exceptions` as before. `e.error` still holds the `ErrorData`, so `e.error.code`/`e.error.message` keep working; `e.code`, `e.message`, and `e.data` are new shortcuts.

**Before (v1):**

```python
from mcp.shared.exceptions import McpError

try:
    result = await session.call_tool("my_tool")
except McpError as e:
    print(f"Error: {e.error.message}")
```

**After (v2):**

```python
from mcp import MCPError

try:
    result = await session.call_tool("my_tool")
except MCPError as e:
    print(f"Error: {e.message}")
```

The constructor now takes `code`, `message`, and optional `data` instead of an `ErrorData` (the v1 form raises `TypeError`). If you already hold an `ErrorData`, use `MCPError.from_error_data(error)`.

**Before (v1):**

```python
from mcp.shared.exceptions import McpError
from mcp.types import ErrorData, INVALID_REQUEST

raise McpError(ErrorData(code=INVALID_REQUEST, message="bad input"))
```

**After (v2):**

```python
from mcp import MCPError
from mcp_types import INVALID_REQUEST

raise MCPError(INVALID_REQUEST, "bad input")
```

## MCPServer (formerly FastMCP)

### `FastMCP` renamed to `MCPServer`

`FastMCP` is now `MCPServer`, and the `mcp.server.fastmcp` subpackage is now `mcp.server.mcpserver` (the old paths are removed, not deprecated).

**Before (v1):**

```python
from mcp.server.fastmcp import Context, FastMCP

mcp = FastMCP("Demo")
```

**After (v2):**

```python
from mcp.server import MCPServer
from mcp.server.mcpserver import Context

mcp = MCPServer("Demo")
```

The rest of the subpackage keeps its layout under `mcp.server.mcpserver.*`:

- `Image`, `Audio`, `Icon` — from `mcp.server.mcpserver`
- `UserMessage`, `AssistantMessage` — from `mcp.server.mcpserver.prompts.base`
- `ToolError`, `ResourceError`, `MCPServerError` (was `FastMCPError`) — from `mcp.server.mcpserver.exceptions`

The `ctx.fastmcp` property is now `ctx.mcp_server`.

### Default server identity changed

A nameless server now reports `serverInfo.name == "mcp-server"` (v1: `"FastMCP"`), and an unversioned server reports an empty `serverInfo.version` (v1: the installed `mcp` package version). Nothing raises, but tests asserting on the `initialize` result and clients that display `serverInfo` see the new values. Set both explicitly:

**Before (v1):**

```python
from mcp.server.fastmcp import FastMCP

mcp = FastMCP()  # serverInfo.name == "FastMCP", serverInfo.version == installed mcp version
```

**After (v2):**

```python
from mcp.server import MCPServer, Server

mcp = MCPServer("my-server", version="1.2.3")
server = Server("my-server", version="1.2.3")  # lowlevel Server takes the same keywords
```

### `MCPServer` constructor: positional parameter order changed

The positional order is now `name, title, description, instructions, website_url, icons, version` (v1: `name, instructions, website_url, icons`). A v1 call that passed `instructions` as the second positional argument still runs on v2, but the string lands in `title`: the server reports it as `serverInfo.title` and clients receive no `instructions`.

**Before (v1):**

```python
from mcp.server.fastmcp import FastMCP

mcp = FastMCP("Demo", "You answer questions about the weather.")
```

**After (v2):**

```python
from mcp.server import MCPServer

mcp = MCPServer("Demo", instructions="You answer questions about the weather.")
```

Keep `name` positional and pass everything else by keyword.

### `mount_path` parameter removed from `MCPServer`

`mount_path` is gone from the constructor, `Settings`, `run()`, `run_sse_async()`, and `sse_app()`: passing it raises `TypeError`, and `mcp.settings.mount_path = ...` raises `ValueError`. The SSE transport already prefixes its message endpoint with the ASGI `root_path` that Starlette's `Mount` sets, so delete `mount_path` and mount as before.

**Before (v1):**

```python
from starlette.applications import Starlette
from starlette.routing import Mount

from mcp.server.fastmcp import FastMCP

github_mcp = FastMCP("GitHub", mount_path="/github")
search_mcp = FastMCP("Search")
search_mcp.settings.mount_path = "/search"

app = Starlette(
    routes=[
        Mount("/github", app=github_mcp.sse_app()),
        Mount("/search", app=search_mcp.sse_app()),
    ]
)
```

**After (v2):**

```python
from starlette.applications import Starlette
from starlette.routing import Mount

from mcp.server import MCPServer

github_mcp = MCPServer("GitHub")
search_mcp = MCPServer("Search")

app = Starlette(
    routes=[
        Mount("/github", app=github_mcp.sse_app()),
        Mount("/search", app=search_mcp.sse_app()),
    ]
)
```

`mcp.run(transport="sse", mount_path="/x")` becomes `mcp.run(transport="sse")`. See [Add to an existing app](run/asgi.md) for mounting details.

### Transport parameters moved from the `MCPServer` constructor to `run()`

Transport options are no longer constructor arguments; `MCPServer(...)` raises `TypeError: MCPServer.__init__() got an unexpected keyword argument` for any of them. Pass them where the server starts instead: `run()` (per transport), or `streamable_http_app()` / `sse_app()` when you build the ASGI app yourself (same options minus `port`, which belongs to whatever serves the app).

Moved: `host`, `port`, `streamable_http_path`, `json_response`, `stateless_http`, `max_request_body_size` (default 4 MiB; oversized POSTs get HTTP 413), `event_store`, `retry_interval`, `sse_path`, `message_path`, `transport_security`. (`debug` and `log_level` stay on the constructor; `mount_path` is removed, not moved.)

**Before (v1):**

```python
from mcp.server.fastmcp import FastMCP

mcp = FastMCP("Demo", json_response=True, stateless_http=True)
mcp.run(transport="streamable-http")

# or, for SSE:
mcp = FastMCP("Server", host="0.0.0.0", port=9000, sse_path="/events")
mcp.run(transport="sse")
```

**After (v2):**

```python
from mcp.server import MCPServer

mcp = MCPServer("Demo")
mcp.run(transport="streamable-http", json_response=True, stateless_http=True)

# or, for SSE:
mcp = MCPServer("Server")
mcp.run(transport="sse", host="0.0.0.0", port=9000, sse_path="/events")
```

If you build the ASGI app yourself, the same options go on the app method:

```python
from mcp.server import MCPServer

mcp = MCPServer("App")
app = mcp.streamable_http_app(json_response=True)  # v1: FastMCP("App", json_response=True).streamable_http_app()
```

The moved fields are gone from `mcp.settings` too, so post-construction mutation (`mcp.settings.port = 9000`) must move to the same call sites. Localhost DNS-rebinding protection is now armed from the `host=` these methods receive (default `127.0.0.1`) rather than from the constructor: v1 code that set `host="0.0.0.0"` or `transport_security=` on `FastMCP(...)` to serve a real hostname must pass it at the new call site, or non-localhost requests are rejected with `421`. See [Running your server](run/index.md) and [Add to an existing app](run/asgi.md).

### Streamable HTTP: lifespan runs once, not per session or request

Under streamable HTTP, v1 entered the server's `lifespan` for every session (stateful) or every request (`stateless_http=True`) and exited it when that session or request ended. In v2 `StreamableHTTPSessionManager.run()` enters it once at startup, and the object it yields is the same `ctx.request_context.lifespan_context` for every session and request (see [Lifespan](handlers/lifespan.md)). SSE and stdio are unchanged.

A lifespan that builds server-wide state (connection pool, HTTP client, loaded model) needs no change. State you scoped to a session or request by yielding it from the lifespan (a per-client cache, a per-connection transaction) is now shared between all clients and only torn down at server shutdown. v2 exposes no per-session teardown hook to handlers: acquire and release such state per call inside the handler, and keep only server-wide resources in the lifespan.

### `MCPServer.get_context()` removed

`MCPServer.get_context()` is gone; there is no ambient request context to fetch. Declare a `ctx: Context` parameter on the handler and the SDK injects it. Injection only happens for the registered function, so helpers that called `get_context()` now take `ctx` as an argument. See [The Context](handlers/context.md).

**Before (v1):**

```python
@mcp.tool()
async def my_tool(x: int) -> str:
    ctx = mcp.get_context()
    await ctx.report_progress(1, 2)
    return f"{ctx.request_id}: {x}"
```

**After (v2):**

```python
from mcp.server.mcpserver import Context

@mcp.tool()
async def my_tool(x: int, ctx: Context) -> str:
    await ctx.report_progress(1, 2)
    return f"{ctx.request_id}: {x}"
```

### Sync handler functions now run on a worker thread

v1 called a synchronous (`def`) tool, resource, or prompt function directly on the event-loop thread. v2 runs it on a worker thread via `anyio.to_thread.run_sync()`; `async def` handlers are unchanged. Most servers just gain concurrency, but a synchronous handler that depended on the event-loop thread now behaves differently:

- `asyncio.get_running_loop()` in the body raises `RuntimeError: no running event loop`.
- Thread-affine state (a `sqlite3` connection created at import time, thread locals set at startup) is now used from another thread.
- Concurrent requests to synchronous handlers run in parallel, so unlocked shared state races.

Declare the handler `async def` to keep it on the event-loop thread.

**Before (v1):**

```python
import asyncio

@mcp.tool()
def now_ms() -> int:
    return int(asyncio.get_running_loop().time() * 1000)
```

**After (v2):**

```python
import asyncio

@mcp.tool()
async def now_ms() -> int:
    return int(asyncio.get_running_loop().time() * 1000)
```

### Calling `MCPServer.call_tool()`, `get_prompt()`, and `read_resource()` directly

Called on the server object (not through a client), these methods changed in two ways. `call_tool()` returns a `CallToolResult` instead of a bare content list or a `(content, structured_content)` tuple, and all three return types are widened with `InputRequiredResult` (produced only when a [multi-round-trip](handlers/multi-round-trip.md) handler asks the client for input), so narrow with `isinstance`. They also no longer inherit the caller's request context: calling one from inside a handler builds a request-less [`Context`](handlers/context.md), so the inner handler's `ctx.session` or `ctx.request_id` raises "Context is not available outside of a request" unless you pass `context=ctx`.

**Before (v1):**

```python
from mcp.server.fastmcp import Context, FastMCP

mcp = FastMCP("Demo")


@mcp.tool()
async def report(ctx: Context) -> str:
    return f"report for request {ctx.request_id}"


@mcp.tool()
async def digest(ctx: Context) -> str:
    content, structured = await mcp.call_tool("report", {})  # inner handler sees this request's ctx
    return f"digest: {content[0].text}"
```

**After (v2):**

```python
from mcp.server import MCPServer
from mcp.server.mcpserver import Context
from mcp_types import CallToolResult, TextContent

mcp = MCPServer("Demo")


@mcp.tool()
async def report(ctx: Context) -> str:
    return f"report for request {ctx.request_id}"


@mcp.tool()
async def digest(ctx: Context) -> str:
    result = await mcp.call_tool("report", {}, context=ctx)
    assert isinstance(result, CallToolResult)
    block = result.content[0]
    assert isinstance(block, TextContent)
    return f"digest: {block.text}"
```

In tests, prefer the in-memory [`Client`](get-started/testing.md), whose `call_tool()`, `get_prompt()`, and `read_resource()` return plain results and need no narrowing.

### `MCPError` raised from an `@mcp.tool()` handler now surfaces as a JSON-RPC error

Raising `MCPError` (or any subclass) inside an `@mcp.tool()` handler, or letting one propagate from a call the tool makes, now fails the `tools/call` request with a JSON-RPC error carrying the raised `code`, `message`, and `data`, so the client's `call_tool()` raises `MCPError` instead of returning a result. v1 caught it like any other exception and returned `CallToolResult(isError=True)` with text `Error executing tool <name>: <message>`, dropping the code and `data` (`UrlElicitationRequiredError` was already re-raised and is unchanged).

**Before (v1):**

```python
from mcp.server.fastmcp import FastMCP
from mcp.shared.exceptions import McpError
from mcp.types import INVALID_PARAMS, ErrorData

mcp = FastMCP("demo")


@mcp.tool()
def strict(x: int) -> int:
    if x < 0:
        raise McpError(ErrorData(code=INVALID_PARAMS, message="x must be non-negative"))
    return x

# session.call_tool("strict", {"x": -1}) returned CallToolResult(isError=True)
# with text "Error executing tool strict: x must be non-negative"
```

**After (v2):**

```python
from mcp import MCPError
from mcp.server import MCPServer
from mcp_types import INVALID_PARAMS

mcp = MCPServer("demo")


@mcp.tool()
def strict(x: int) -> int:
    if x < 0:
        raise MCPError(INVALID_PARAMS, "x must be non-negative", {"x": x})
    return x

# client.call_tool("strict", {"x": -1}) raises MCPError(code=-32602, data={"x": -1})
```

To keep the old model-visible `is_error=True` result, raise a plain exception (any non-`MCPError` still becomes one) or return `CallToolResult(is_error=True, ...)` from the tool. See [Handling errors](servers/handling-errors.md).

### Missing resource reads return `-32602` and raise `ResourceNotFoundError`

Reading a URI that matches no resource or template now fails with JSON-RPC code `-32602` (invalid params) and `error.data = {"uri": ...}`, per [SEP-2164](https://github.com/modelcontextprotocol/modelcontextprotocol/pull/2164); v1 returned code `0` with no `data`. Other resource read failures now return `-32603` with the same `data` and a generic message instead of the exception text.

Server-side, `mcp.read_resource()` and `ctx.read_resource()` raise `ResourceNotFoundError` for an unknown URI and its parent `ResourceError` for a failing template, where v1 raised `ValueError` for both.

**Before (v1):**

```python
try:
    contents = await ctx.read_resource("config://missing")
except ValueError:
    ...
```

**After (v2):**

```python
from mcp.server.mcpserver.exceptions import ResourceNotFoundError

try:
    contents = await ctx.read_resource("config://missing")
except ResourceNotFoundError:
    ...
```

A template handler that raised a plain exception for a missing instance should now raise `ResourceNotFoundError` so its message and the URI reach the client as `-32602`; see [Handling errors](servers/handling-errors.md).

### `Resource` classes reject unknown keyword arguments

The `Resource` base class now sets `extra="forbid"`, so `TextResource`, `BinaryResource`, `FunctionResource`, `FileResource`, `HttpResource`, `DirectoryResource`, and your own subclasses raise `pydantic.ValidationError` on an unrecognised keyword argument instead of silently dropping it. A misspelled or since-removed parameter (such as `FileResource(is_binary=...)`, below) now fails at construction. Remove the stray argument; to restore v1's silent-drop behavior on a subclass, set `model_config = ConfigDict(extra="ignore")`.

### `FileResource.is_binary` replaced by `encoding`

`FileResource` no longer takes `is_binary`; passing it raises `pydantic.ValidationError` at construction. Text versus blob is now chosen by `encoding: str | None` (`None` reads bytes and serves a base64 blob). When omitted, `encoding` is derived from `mime_type`: a declared `charset=` wins, otherwise `"utf-8-sig"` (UTF-8, BOM tolerated) for `text/*`, `application/json`, `application/xml` and any `+json`/`+xml` type, and `None` for everything else.

Two defaults change with no edit on your side: files whose mime type is textual but not `text/*` (`application/json`, `application/xml`, `image/svg+xml`, …), which v1 served as blobs, are now served as text; and text files are decoded as UTF-8 instead of with the platform locale encoding (v1 called `Path.read_text()` with no encoding). Where either default is wrong for a file, pass `encoding` explicitly.

**Before (v1):**

```python
from pathlib import Path

from mcp.server.fastmcp.resources import FileResource

FileResource(uri="file:///logo.png", path=Path("/srv/logo.png"), mime_type="image/png", is_binary=True)
FileResource(uri="file:///notes.txt", path=Path("/srv/notes.txt"))  # text, locale encoding
FileResource(uri="file:///data.json", path=Path("/srv/data.json"), mime_type="application/json")  # blob
```

**After (v2):**

```python
from pathlib import Path

from mcp.server.mcpserver.resources import FileResource

FileResource(uri="file:///logo.png", path=Path("/srv/logo.png"), mime_type="image/png")  # blob, from mime_type
FileResource(uri="file:///notes.txt", path=Path("/srv/notes.txt"))  # text, UTF-8
FileResource(uri="file:///data.json", path=Path("/srv/data.json"), mime_type="application/json")  # now text
FileResource(uri="file:///legacy.txt", path=Path("/srv/legacy.txt"), encoding="cp1252")  # non-UTF-8 text
# keep this JSON file as a blob
FileResource(uri="file:///raw.json", path=Path("/srv/raw.json"), mime_type="application/json", encoding=None)
```

### Resource templates: matching behavior changes

Template matching now follows [RFC 6570](https://datatracker.ietf.org/doc/html/rfc6570) instead of a regex built from the URI string. Existing `{var}` templates behave differently in these ways:

**Extracted values are percent-decoded.** `docs://{name}` read as `docs://hello%20world` calls the handler with `name = "hello world"`; v1 passed `"hello%20world"`. Because `%2F` decodes to `/`, a simple `{var}` value can now contain slashes (`docs://a%2Fb` gives `name = "a/b"`), which v1's `[^/]+` capture ruled out. Remove any manual `unquote()` in the handler.

**Path-safety checks reject unsafe values by default.** A decoded value that escapes upward through `..` segments, looks like an absolute path (`/etc/passwd`, `C:\Windows`, or any single-letter `x:` prefix), or contains a null byte fails the read with the same "Unknown resource" error as an unmatched URI, and the handler never runs. `..` only counts as a whole path segment, so `v1.0..v2.0` still passes. Exempt a parameter that legitimately carries such values, or set `resource_security=` on `MCPServer` to change the policy server-wide:

```python
from mcp.server.mcpserver import ResourceSecurity

@mcp.resource("inspect://file/{target}", security=ResourceSecurity(exempt_params={"target"}))
def inspect_file(target: str) -> str: ...
```

**Literals and delimiters match exactly.** `data://v1.0/{id}` no longer matches `data://v1X0/42` (a template `.` was regex any-character), and `{var}` no longer captures `?`, `#`, `&`, or `,`: `api://{id}` no longer matches `api://foo?x=1`. To keep accepting the query string, write `api://{id}{?x}` and give the `x` parameter a default.

**`{var}` matches an empty value.** `tickets://{ticket_id}` now matches `tickets://` with `ticket_id=""`; v1's `[^/]+` returned "Unknown resource" instead. Validate non-empty values in the handler if you relied on that.

**Templates v1 accepted can now fail at decoration time.** Duplicate variable names and similar syntax errors raise `InvalidUriTemplate` (a `ValueError`) when the decorator runs instead of `re.error` on the first read. Two adjacent variables (`x://{name}{path}`), which v1's regex accepted, are rejected — separate them with a literal (`x://{name}/{path}`). A static URI whose handler takes only a `Context` parameter, silently unreachable in v1, now raises `ValueError` at decoration time.

See [URI templates and path safety](servers/uri-templates.md) for the operators added in v2 and the security settings.

### `Context` logging: `message` keyword renamed to `data`

On the `MCPServer` `Context`, `log()`, `debug()`, `info()`, `warning()`, and `error()` now take `data: Any` (any JSON-serializable value) instead of `message: str`; the only other keyword option is `logger_name=`.

**Before (v1):**

```python
await ctx.log(level="info", message="hello")
```

**After (v2):**

```python
await ctx.log(level="info", data="hello")
```

Positional calls (`await ctx.info("hello")`) are unaffected. These methods are also deprecated in v2 and emit `MCPDeprecationWarning`; see [Deprecated features](deprecated.md).

### `Context.client_id` removed

`ctx.client_id` is removed; it only echoed a non-standard `client_id` key from the request's `_meta`. Read that key directly, or use the access token for the authenticated OAuth client (see [Authorization](run/authorization.md#the-callers-identity)).

**Before (v1):**

```python
client_id = ctx.client_id
```

**After (v2):**

```python
from mcp.server.auth.middleware.auth_context import get_access_token

# the raw _meta key, if you set it yourself
meta = ctx.request_context.meta
client_id = meta.get("client_id") if meta else None

# the authenticated OAuth client (usually what you want)
token = get_access_token()
client_id = token.client_id if token else None
```

### `mcp.shared.progress` module removed

`Progress`, `ProgressContext`, and the `progress()` context manager are removed. Use `Context.report_progress()`, which takes the absolute progress value (`ProgressContext.progress(amount)` added its argument to a running total) and no-ops instead of raising `ValueError` when the request carries no progress token.

**Before (v1):**

```python
from mcp.shared.progress import progress

with progress(request_context, total=100) as p:
    await p.progress(25)  # adds 25 to a running total
```

**After (v2):**

```python
from mcp.server import MCPServer
from mcp.server.mcpserver import Context

mcp = MCPServer("Progress")


@mcp.tool()
async def crunch(steps: int, ctx: Context) -> str:
    for done in range(1, steps + 1):
        await ctx.report_progress(done, total=steps)
    return "done"
```

In a lowlevel `Server` handler, call `await ctx.session.report_progress(...)` on the `ServerRequestContext` instead. See [Progress](handlers/progress.md).

### `Context.elicit()` rejects non-spec schemas; mismatched answers raise `ValueError`

`ctx.elicit()` (and `elicit_with_validation()`) now reject two field shapes v1 accepted, raising `TypeError` before anything is sent: a bare `list[str]` and a union of primitives such as `int | str`. Use `list[Literal[...]]` for a multi-select and a single primitive type per field.

**Before (v1)** (accepted, sent a non-spec schema):

```python
from pydantic import BaseModel


class Preferences(BaseModel):
    tags: list[str]
    level: int | str
```

**After (v2):**

```python
from typing import Literal

from pydantic import BaseModel


class Preferences(BaseModel):
    tags: list[Literal["news", "sports"]]
    level: str
```

An accepted answer whose content doesn't match the schema now raises `ValueError` (the pydantic `ValidationError` is its `__cause__`) instead of letting `ValidationError` propagate; change `except ValidationError` around `ctx.elicit()` to `except ValueError`.

See [Elicitation](handlers/elicitation.md).

### `isinstance()` against `ElicitationResult` raises `TypeError`

`ElicitationResult` is now a generic `TypeAliasType` rather than a plain union, and it is no longer a valid `isinstance()` target. Check the member classes instead, or branch on `result.action` (`"accept"` / `"decline"` / `"cancel"`) as before.

**Before (v1):**

```python
from mcp.server.elicitation import ElicitationResult

result = await ctx.elicit("Proceed?", Confirm)
if isinstance(result, ElicitationResult):
    ...
```

**After (v2):**

```python
from mcp.server.mcpserver import AcceptedElicitation, CancelledElicitation, DeclinedElicitation

result = await ctx.elicit("Proceed?", Confirm)
if isinstance(result, (AcceptedElicitation, DeclinedElicitation, CancelledElicitation)):
    ...
```

## Lowlevel Server

### Lowlevel `Server`: decorator-based handlers replaced with constructor `on_*` params

The lowlevel `Server` no longer has `@server.list_tools()`-style registration decorators. Pass each handler to the constructor as an `on_*` keyword argument; every handler has the signature `async (ctx: ServerRequestContext, params) -> Result`, taking the typed params model and returning the full result type.

**Before (v1):**

```python
from mcp.server.lowlevel.server import Server
import mcp.types as types

server = Server("my-server")

@server.list_tools()
async def handle_list_tools():
    return [types.Tool(name="my_tool", description="A tool", inputSchema={})]

@server.call_tool()
async def handle_call_tool(name: str, arguments: dict):
    return [types.TextContent(type="text", text=f"Called {name}")]
```

**After (v2):**

```python
from mcp.server import Server, ServerRequestContext
from mcp_types import (
    CallToolRequestParams,
    CallToolResult,
    ListToolsResult,
    PaginatedRequestParams,
    TextContent,
    Tool,
)


async def handle_list_tools(ctx: ServerRequestContext, params: PaginatedRequestParams | None) -> ListToolsResult:
    return ListToolsResult(tools=[Tool(name="my_tool", description="A tool", input_schema={"type": "object"})])


async def handle_call_tool(ctx: ServerRequestContext, params: CallToolRequestParams) -> CallToolResult:
    return CallToolResult(content=[TextContent(type="text", text=f"Called {params.name}")])


server = Server("my-server", on_list_tools=handle_list_tools, on_call_tool=handle_call_tool)
```

`ctx.session` is the `ServerSession`; `ctx.lifespan_context`, `ctx.request_id`, and `ctx.meta` are on the context too. The `jsonschema` input/output validation that `@server.call_tool()` performed is gone: `input_schema` is advertised, never applied, so validate `params.arguments` yourself.

| v1 decorator | v2 constructor kwarg | `params` type | return type |
|---|---|---|---|
| `@server.list_tools()` | `on_list_tools` | `PaginatedRequestParams \| None` | `ListToolsResult` |
| `@server.call_tool()` | `on_call_tool` | `CallToolRequestParams` | `CallToolResult` |
| `@server.list_resources()` | `on_list_resources` | `PaginatedRequestParams \| None` | `ListResourcesResult` |
| `@server.list_resource_templates()` | `on_list_resource_templates` | `PaginatedRequestParams \| None` | `ListResourceTemplatesResult` |
| `@server.read_resource()` | `on_read_resource` | `ReadResourceRequestParams` | `ReadResourceResult` |
| `@server.subscribe_resource()` | `on_subscribe_resource` | `SubscribeRequestParams` | `EmptyResult` |
| `@server.unsubscribe_resource()` | `on_unsubscribe_resource` | `UnsubscribeRequestParams` | `EmptyResult` |
| `@server.list_prompts()` | `on_list_prompts` | `PaginatedRequestParams \| None` | `ListPromptsResult` |
| `@server.get_prompt()` | `on_get_prompt` | `GetPromptRequestParams` | `GetPromptResult` |
| `@server.completion()` | `on_completion` | `CompleteRequestParams` | `CompleteResult` |
| `@server.set_logging_level()` | `on_set_logging_level` | `SetLevelRequestParams` | `EmptyResult` |
| `@server.progress_notification()` | `on_progress` | `ProgressNotificationParams` | `None` |
| (none) | `on_ping` | `RequestParams \| None` | `EmptyResult` |
| (none) | `on_roots_list_changed` | `NotificationParams \| None` | `None` |

All `params` and result types come from `mcp_types`. `on_set_logging_level`, `on_roots_list_changed`, and `on_progress` are deprecated and emit `MCPDeprecationWarning` when passed; see [Deprecated features](deprecated.md). The full handler model is in [The low-level Server](advanced/low-level-server.md).

### Lowlevel `Server`: automatic return value wrapping removed

The v1 handler decorators converted several return shapes into the wire result type; v2 `on_*` handlers return the fully constructed result (see [The low-level Server](advanced/low-level-server.md)). If you want the conveniences back, use `MCPServer`, whose `@mcp.tool()` and `@mcp.resource()` still wrap return values.

`call_tool` wrapped `list[ContentBlock]`, `dict`, or `(list, dict)` into a `CallToolResult`; a `dict` became `structuredContent` plus a JSON `TextContent`. Build it yourself:

**Before (v1):**

```python
@server.call_tool()
async def handle(name: str, arguments: dict) -> dict:
    return {"temperature": 22.5, "city": "London"}
```

**After (v2):**

```python
import json

from mcp.server import ServerRequestContext
from mcp_types import CallToolRequestParams, CallToolResult, TextContent


async def handle(ctx: ServerRequestContext, params: CallToolRequestParams) -> CallToolResult:
    data = {"temperature": 22.5, "city": "London"}
    return CallToolResult(
        content=[TextContent(type="text", text=json.dumps(data, indent=2))],
        structured_content=data,
    )
```

`params.arguments` is `None` when the client sends no arguments (the v1 decorator passed `{}`); use `params.arguments or {}`.

`read_resource` converted `Iterable[ReadResourceContents]` (and the deprecated `str`/`bytes` shorthand) into `TextResourceContents`/`BlobResourceContents`, base64-encoding bytes and defaulting the mime type. Build the contents yourself:

**Before (v1):**

```python
from collections.abc import Iterable

from pydantic import AnyUrl

from mcp.server.lowlevel.helper_types import ReadResourceContents


@server.read_resource()
async def handle(uri: AnyUrl) -> Iterable[ReadResourceContents]:
    return [ReadResourceContents(content="file contents", mime_type="text/plain")]
```

**After (v2):**

```python
import base64

from mcp.server import ServerRequestContext
from mcp_types import (
    BlobResourceContents,
    ReadResourceRequestParams,
    ReadResourceResult,
    TextResourceContents,
)


async def handle_read(ctx: ServerRequestContext, params: ReadResourceRequestParams) -> ReadResourceResult:
    if params.uri.endswith(".png"):
        blob = base64.b64encode(b"\x89PNG...").decode()
        return ReadResourceResult(contents=[BlobResourceContents(uri=params.uri, blob=blob, mime_type="image/png")])
    return ReadResourceResult(
        contents=[TextResourceContents(uri=params.uri, text="file contents", mime_type="text/plain")]
    )
```

The `list_*` handlers likewise return `ListToolsResult(tools=[...])`, `ListResourcesResult(resources=[...])`, and `ListPromptsResult(prompts=[...])` instead of bare lists.

### Lowlevel `Server`: tool handler exceptions no longer become error results

The v1 `@server.call_tool()` decorator caught any exception the handler raised and returned it as `CallToolResult(isError=True)`, so the model saw the error text and could retry. The v2 `on_call_tool` handler has no such wrapper: the exception fails the whole `tools/call` request with a JSON-RPC error instead of a result, the client raises `MCPError`, and the model gets no tool result.

**Before (v1):**

```python
@server.call_tool()
async def call_tool(name: str, arguments: dict):
    raise ValueError("kaboom")
```

**After (v2):** catch the exception in the handler and build the error result yourself:

```python
from mcp.server import ServerRequestContext
from mcp_types import CallToolRequestParams, CallToolResult, TextContent


async def handle_call_tool(ctx: ServerRequestContext, params: CallToolRequestParams) -> CallToolResult:
    try:
        args = params.arguments or {}
        text = str(args["a"] + args["b"])  # tool logic that may raise
    except Exception as e:
        return CallToolResult(content=[TextContent(type="text", text=str(e))], is_error=True)
    return CallToolResult(content=[TextContent(type="text", text=text)])
```

Raise `MCPError` only when the request itself should fail as a protocol error. If you want the automatic conversion back, `MCPServer`'s `@mcp.tool()` still turns handler exceptions into `is_error=True` results; see [Handling errors](servers/handling-errors.md).

### Lowlevel `Server`: constructor parameters are now keyword-only

All parameters after `name` are now keyword-only, so passing `version` (or anything else) positionally raises `TypeError` at construction.

**Before (v1):**

```python
from mcp.server import Server

server = Server("my-server", "1.0")
```

**After (v2):**

```python
from mcp.server import Server

server = Server("my-server", version="1.0")
```

### Lowlevel `Server`: type parameter reduced from 2 to 1

`Server[LifespanResultT, RequestT]` is now `Server[LifespanResultT]`. Drop the second type argument; a two-argument subscript fails type checking and raises `TypeError` wherever the subscript is evaluated at runtime (a module-level annotation on Python 3.13 or earlier, a subclass base, `get_type_hints`).

**Before (v1):**

```python
from typing import Any

from mcp.server import Server

server: Server[dict[str, Any], Any] = Server("my-server")
```

**After (v2):**

```python
from typing import Any

from mcp.server import Server

server: Server[dict[str, Any]] = Server("my-server")
```

The transport request object that `RequestT` typed (`server.request_context.request` in v1) is now typed on the handler's `ctx: ServerRequestContext[LifespanContextT, RequestT]`; see [`RequestContext` type parameters simplified](#requestcontext-type-parameters-simplified).

### Lowlevel `Server`: `request_handlers` and `notification_handlers` attributes removed

The public `server.request_handlers` and `server.notification_handlers` dicts are gone. Register handlers after construction with `add_request_handler` / `add_notification_handler`, and look them up with `get_request_handler` / `get_notification_handler`, which return the registered entry (with `.handler` and `.params_type`) or `None`. Keys are method strings, not request types.

**Before (v1):**

```python
from mcp.types import ListToolsRequest

server.request_handlers[ListToolsRequest] = handle_list_tools
if ListToolsRequest in server.request_handlers:
    ...
```

**After (v2):**

```python
from mcp_types import PaginatedRequestParams

server.add_request_handler("tools/list", PaginatedRequestParams, handle_list_tools)
if server.get_request_handler("tools/list") is not None:
    ...
```

`handle_list_tools` uses the v2 `(ctx, params)` handler signature; see [The low-level Server](advanced/low-level-server.md) for `add_request_handler` and custom methods.

### Lowlevel `Server`: private `_handle_*` dispatch methods removed

`Server._handle_message`, `_handle_request`, and `_handle_notification` no longer exist, so a `Server` subclass that overrode them to intercept traffic still constructs but the override is never called. Append a middleware to `server.middleware` instead; it wraps every inbound request and notification as `(ctx, call_next)`. See [Middleware](advanced/middleware.md).

**After (v2):**

```python
from mcp.server import Server, ServerRequestContext
from mcp.server.context import CallNext, HandlerResult


async def logging_middleware(ctx: ServerRequestContext, call_next: CallNext) -> HandlerResult:
    print(f"handling {ctx.method}")
    result = await call_next(ctx)
    print(f"done {ctx.method}")
    return result


server = Server("my-server")
server.middleware.append(logging_middleware)
```

### Lowlevel `Server.run(raise_exceptions=True)`: transport errors no longer re-raised

`raise_exceptions=True` now only affects handler exceptions: an unexpected exception from a handler still propagates out of `run()`, but the JSON-RPC error response is written to the client first (v1 re-raised without responding).

Exceptions the transport places on the read stream (e.g. a stdio JSON parse error) are now debug-logged and dropped regardless of the flag: `run()` keeps serving instead of raising, and the client no longer receives the `Internal Server Error` `notifications/message` log entry v1 sent.

### Lowlevel `Server.run()` no longer takes a `stateless` flag

The `stateless` parameter is gone from `Server.run()`; passing it raises `TypeError`. Statelessness is now configured on the HTTP transport, not per connection.

**Before (v1):**

```python
await server.run(read_stream, write_stream, init_options, stateless=True)
```

**After (v2):**

```python
from mcp.server.streamable_http_manager import StreamableHTTPSessionManager

app = server.streamable_http_app(stateless_http=True)  # the lowlevel Server now builds the ASGI app
# or, driving the session manager yourself (unchanged from v1):
manager = StreamableHTTPSessionManager(app=server, stateless=True)
```

Server-initiated requests (`create_message`, `elicit`, `list_roots`) from a stateless-HTTP handler no longer stall waiting for a reply that can never arrive; they raise `NoBackChannelError` (an `MCPError` subclass in `mcp.shared.exceptions`). See [Serving legacy clients](run/legacy-clients.md).

### Lowlevel `Server`: `request_context` property removed

`server.request_context` and the module-level `request_ctx` contextvar are gone. The request context is now the handler's first argument, `ctx: ServerRequestContext`, with the same `session`, `lifespan_context`, `request_id`, `meta`, and `request` fields. There is no ambient accessor in v2, so pass `ctx` down to any helper that called `request_ctx.get()`.

**Before (v1):**

```python
import mcp.types as types
from mcp.server.lowlevel import Server
from mcp.server.lowlevel.server import request_ctx

server = Server("my-server")


@server.call_tool()
async def query_db(name: str, arguments: dict) -> list[types.TextContent]:
    ctx = server.request_context  # or, from any nested helper, request_ctx.get()
    db = ctx.lifespan_context["db"]
    return [types.TextContent(type="text", text=f"queried {db}")]
```

**After (v2):**

```python
from mcp.server import Server, ServerRequestContext
from mcp_types import CallToolRequestParams, CallToolResult, TextContent


async def query_db(ctx: ServerRequestContext, params: CallToolRequestParams) -> CallToolResult:
    db = ctx.lifespan_context["db"]
    return CallToolResult(content=[TextContent(type="text", text=f"queried {db}")])


server = Server("my-server", on_call_tool=query_db)
```

See [Low-level server](advanced/low-level-server.md) for the handler shape and `ctx` fields.

### `RequestContext` type parameters simplified

`mcp.shared.context.RequestContext[SessionT, LifespanContextT, RequestT]` is gone (`from mcp.shared.context import RequestContext` raises `ImportError`). It is split by side, and the leading session type parameter is dropped:

- Client callbacks (`sampling`, `elicitation`, `list_roots`) receive `ClientRequestContext` from `mcp.client`. It is not generic; its fields are `session`, `request_id`, `meta` (the always-`None` `lifespan_context` is gone).
- Server code uses `ServerRequestContext[LifespanContextT, RequestT]` from `mcp.server`.
- The injected high-level `Context[ServerSessionT, LifespanContextT, RequestT]` becomes `Context[LifespanContextT, RequestT]`: `Context[ServerSession, None]` becomes bare `Context`, and `Context[ServerSession, AppContext]` becomes `Context[AppContext]`.

A leftover two-argument `Context[ServerSession, AppContext]` still runs but types `lifespan_context` as `ServerSession`; three type arguments raise `TypeError`.

**Before (v1):**

```python
from dataclasses import dataclass
from typing import Any

from mcp import ClientSession, types
from mcp.server.fastmcp import Context
from mcp.server.session import ServerSession
from mcp.shared.context import RequestContext


@dataclass
class AppContext:
    db_url: str


async def handle_sampling(
    context: RequestContext[ClientSession, None], params: types.CreateMessageRequestParams
) -> types.CreateMessageResult: ...


def db_url_from(ctx: RequestContext[ServerSession, AppContext, Any]) -> str:
    return ctx.lifespan_context.db_url


async def typed_tool(ctx: Context[ServerSession, AppContext]) -> str:
    return ctx.request_context.lifespan_context.db_url
```

**After (v2):**

```python
from dataclasses import dataclass

import mcp_types as types

from mcp.client import ClientRequestContext
from mcp.server import ServerRequestContext
from mcp.server.mcpserver import Context


@dataclass
class AppContext:
    db_url: str


async def handle_sampling(
    context: ClientRequestContext, params: types.CreateMessageRequestParams
) -> types.CreateMessageResult: ...


def db_url_from(ctx: ServerRequestContext[AppContext]) -> str:
    return ctx.lifespan_context.db_url


async def typed_tool(ctx: Context[AppContext]) -> str:
    return ctx.request_context.lifespan_context.db_url
```

`isinstance(ctx, RequestContext)` checks must name the concrete class, and the `LifespanContextT`/`RequestT` TypeVars now live in `mcp.server.context`. `ServerRequestContext.request_id` is `RequestId | None` (`None` when the same context reaches a notification handler), so code passing `ctx.request_id` on as a definite `RequestId` needs a `None` check.

### `ServerSession` can no longer be constructed or driven directly

`ServerSession` is no longer a `BaseSession` subclass or an async context manager. The framework builds one for each inbound request (not one per connection, so don't rely on `ctx.session` object identity across requests) and hands it to handlers as `ctx.session`; its helpers (`send_*`, `create_message`, `elicit_*`, `client_params`, `check_client_capability`) are still there, but constructing or subclassing the class is not supported.

**Before (v1):**

```python
import anyio

from mcp.server.models import InitializationOptions
from mcp.server.session import ServerSession
from mcp.server.stdio import stdio_server
from mcp.types import ServerCapabilities

init_options = InitializationOptions(server_name="mcp", server_version="0.1.0", capabilities=ServerCapabilities())


async def main() -> None:
    async with stdio_server() as (read_stream, write_stream):
        async with ServerSession(read_stream, write_stream, init_options) as session:
            async for message in session.incoming_messages:
                ...


anyio.run(main)
```

**After (v2):**

```python
import anyio

from mcp.server import Server
from mcp.server.stdio import stdio_server

server = Server("mcp")


async def main() -> None:
    async with stdio_server() as (read_stream, write_stream):
        await server.run(read_stream, write_stream, server.create_initialization_options())


anyio.run(main)
```

Register handlers via the `Server` constructor's `on_*` arguments (or `add_request_handler`). To observe every inbound request and notification, which is what `incoming_messages` or an overridden `_receive_loop`/`_received_request` gave you, append a [middleware](advanced/middleware.md).

Also removed from `mcp.server.session`: `InitializationState`, `ServerRequestResponder`, and the `ServerSessionT` TypeVar (see [`RequestContext` type parameters simplified](#requestcontext-type-parameters-simplified)); the tasks-only `experimental`, `send_message()`, and `add_response_router()` members went with the [experimental tasks API](#experimental-tasks-support-removed).

### `ServerSession.elicit()` and `elicit_form()` take `requested_schema`, not `requestedSchema`

The schema parameter of `ServerSession.elicit()` and `ServerSession.elicit_form()` is renamed from `requestedSchema` to `requested_schema`; keyword callers get `TypeError: ServerSession.elicit_form() got an unexpected keyword argument 'requestedSchema'`.

**Before (v1):**

```python
result = await ctx.session.elicit_form(
    message="Your name?",
    requestedSchema={
        "type": "object",
        "properties": {"name": {"type": "string"}},
        "required": ["name"],
    },
)
```

**After (v2):**

```python
result = await ctx.session.elicit_form(
    message="Your name?",
    requested_schema={
        "type": "object",
        "properties": {"name": {"type": "string"}},
        "required": ["name"],
    },
)
```

Positional callers (`session.elicit_form(message, schema)`) are unaffected.

## Clients

### `ClientSession.get_server_capabilities()` replaced by properties

`get_server_capabilities()` is removed; read the `server_capabilities` property, which is likewise `None` until the session is initialized. `server_info`, `instructions`, and `protocol_version` are exposed the same way (`initialize()` still returns the `InitializeResult`).

**Before (v1):**

```python
init = await session.initialize()  # captured for serverInfo / instructions / protocolVersion
capabilities = session.get_server_capabilities()
```

**After (v2):**

```python
await session.initialize()
capabilities = session.server_capabilities
server_info = session.server_info
instructions = session.instructions
version = session.protocol_version
```

The high-level [`Client`](client/index.md) exposes the same four properties, populated on entering its `async with` block.

### `cursor` parameter removed from `ClientSession` list methods

`ClientSession.list_resources()`, `list_resource_templates()`, `list_prompts()`, and `list_tools()` no longer accept the deprecated `cursor` argument; passing it raises `TypeError`. Wrap the cursor in `PaginatedRequestParams`, or use the high-level `Client`, whose `list_*` methods take `cursor=` directly ([Pagination](advanced/pagination.md)).

**Before (v1):**

```python
result = await session.list_resources(cursor="next_page_token")
result = await session.list_tools(cursor="next_page_token")
```

**After (v2):**

```python
from mcp_types import PaginatedRequestParams

result = await session.list_resources(params=PaginatedRequestParams(cursor="next_page_token"))
result = await session.list_tools(params=PaginatedRequestParams(cursor="next_page_token"))

# or, on the high-level Client:
result = await client.list_resources(cursor="next_page_token")
result = await client.list_tools(cursor="next_page_token")
```

### `args` parameter removed from `ClientSessionGroup.call_tool()`

Calls using the old `args=` keyword now raise `TypeError`. Use `arguments=`, or pass the dict positionally (works on both v1 and v2).

**Before (v1):**

```python
result = await session_group.call_tool("my_tool", args={"key": "value"})
```

**After (v2):**

```python
result = await session_group.call_tool("my_tool", arguments={"key": "value"})
```

### Timeouts take `float` seconds instead of `timedelta`

Every timeout parameter that took a `datetime.timedelta` in v1 now takes plain seconds as a `float`:

| Surface | v1 type | v2 type |
|---|---|---|
| `ClientSession(read_timeout_seconds=...)` | `timedelta \| None` | `float \| None` |
| `ClientSession.call_tool(read_timeout_seconds=...)` | `timedelta \| None` | `float \| None` |
| `ClientSession.send_request(request_read_timeout_seconds=...)` | `timedelta \| None` | `float \| None` |
| `ClientSessionGroup.call_tool(read_timeout_seconds=...)` | `timedelta \| None` | `float \| None` |
| `ClientSessionParameters.read_timeout_seconds` | `timedelta \| None` | `float \| None` |
| `StreamableHttpParameters.timeout` / `.sse_read_timeout` | `timedelta` | `float` |
| `ServerSession.send_request(request_read_timeout_seconds=...)` | `timedelta \| None` | `float \| None` |

**Before (v1):**

```python
from datetime import timedelta

session = ClientSession(read_stream, write_stream, read_timeout_seconds=timedelta(seconds=30))
result = await session.call_tool("slow_tool", {}, read_timeout_seconds=timedelta(minutes=2))

params = StreamableHttpParameters(
    url="https://example.com/mcp",
    timeout=timedelta(seconds=30),
    sse_read_timeout=timedelta(seconds=300),
)
```

**After (v2):**

```python
session = ClientSession(read_stream, write_stream, read_timeout_seconds=30)
result = await session.call_tool("slow_tool", {}, read_timeout_seconds=120)

params = StreamableHttpParameters(
    url="https://example.com/mcp",
    timeout=30,
    sse_read_timeout=300,
)
```

Replace `timedelta(...)` with plain seconds, or append `.total_seconds()`. Only `StreamableHttpParameters` fails loudly on a leftover `timedelta` (pydantic `ValidationError: Input should be a valid number`); the session and per-call parameters accept it silently, and the first request that arms the timeout raises `TypeError: unsupported operand type(s) for +: 'float' and 'datetime.timedelta'` from anyio.

### Client request timeouts raise `-32001` (`REQUEST_TIMEOUT`) instead of `408`

A request that exceeds `read_timeout_seconds` still raises `MCPError` (v1's `McpError`), but its error code changed from the HTTP status `408` to the JSON-RPC code `-32001`, exported as `mcp_types.REQUEST_TIMEOUT`. The message wording changed as well and varies by transport, so match on the code, not the text. A leftover `e.error.code == 408` check still evaluates without error and silently never matches.

**Before (v1):**

```python
from mcp.shared.exceptions import McpError

try:
    result = await session.call_tool("slow_tool", {})
except McpError as e:
    if e.error.code == 408:
        ...  # retry / back off
    else:
        raise
```

**After (v2):**

```python
from mcp.shared.exceptions import MCPError
from mcp_types import REQUEST_TIMEOUT  # -32001

try:
    result = await client.call_tool("slow_tool", {})
except MCPError as e:
    if e.code == REQUEST_TIMEOUT:
        ...  # retry / back off
    else:
        raise
```

### `ClientSession` now runs on `JSONRPCDispatcher`; `BaseSession` removed

The `mcp.shared.session` module (`BaseSession`, `RequestResponder`) is gone with no shim, and `ClientSession` no longer subclasses `BaseSession` (its constructor, typed methods, `initialize()`, and async context-manager lifecycle are unchanged). The receive loop now lives in `JSONRPCDispatcher` (`mcp.shared.jsonrpc_dispatcher`); customization goes through the constructor callbacks or the new keyword-only `dispatcher=` argument. `ProgressFnT` moved to `mcp.shared.dispatcher`, `RequestId` to `mcp_types`.

`message_handler` no longer receives requests (the typed callbacks answer them), so its parameter is `IncomingMessage = ServerNotification | Exception`, exported from `mcp.client` (see [Callbacks](client/callbacks.md)). Notifications arrive as the concrete model with no `.root` to unwrap (see [`RootModel` message unions replaced](#rootmodel-message-unions-replaced-by-plain-unions-and-typeadapters)).

**Before (v1):**

```python
import mcp.types as types
from mcp.shared.session import RequestResponder


async def message_handler(
    message: RequestResponder[types.ServerRequest, types.ClientResult] | types.ServerNotification | Exception,
) -> None:
    if isinstance(message, Exception):
        raise message
```

**After (v2):**

```python
from mcp.client import IncomingMessage


async def message_handler(message: IncomingMessage) -> None:
    if isinstance(message, Exception):
        raise message
```

Behavior changes on unchanged session code:

- **Callbacks and notifications run concurrently.** v1 handled one inbound message at a time on the receive loop; v2 runs each server-initiated request callback and each notification callback in its own task. They can interleave, an in-flight callback is cancelled if the server sends `notifications/cancelled`, and a `progress_callback` may fire after its request has returned. Callbacks that need strict ordering must coordinate themselves.
- **A raising request callback** is answered with error `code=0` and the exception text; v1 sent `INVALID_PARAMS` (`"Invalid request parameters"`). Return `ErrorData` or raise `MCPError` to send a specific error (a pydantic `ValidationError` still maps to `INVALID_PARAMS`).
- **Sending outside the session's lifetime raises differently.** `send_request` before `__aenter__` raises `RuntimeError` (v1 wrote to the stream and waited out the timeout); after the session closes it raises `MCPError` (`CONNECTION_CLOSED`) instead of `anyio.ClosedResourceError`.
- **`send_notification` after close is dropped with a debug log** (`send_roots_list_changed` too); v1 raised `anyio.ClosedResourceError`/`BrokenResourceError`, so that exception no longer works as a disconnect signal.
- **`send_notification` no longer takes `related_request_id`.**
- **A timed-out or caller-cancelled request now sends `notifications/cancelled`**, so the server-side handler is cancelled instead of running to completion.
- **Request callbacks receive `mcp.client.ClientRequestContext`** instead of `RequestContext[ClientSession, Any]`; see [`RequestContext` type parameters simplified](#requestcontext-type-parameters-simplified).

### Experimental Tasks support removed

The deprecated experimental Tasks API is removed with no v2 replacement. The `mcp.client.experimental`, `mcp.server.experimental`, `mcp.shared.experimental`, and `mcp.server.lowlevel.experimental` modules are gone, along with the `experimental` accessors on `Server`, `ServerSession`, and `ClientSession` and the `ClientSession(experimental_task_handlers=...)` argument.

Delete code built on `server.experimental.enable_tasks()`, the `@server.experimental.get_task()` / `list_tasks()` / `cancel_task()` / `get_task_result()` decorators, `ctx.experimental.run_task()`, `ServerTaskContext` / `TaskStore`, and `session.experimental.call_tool_as_task()` / `poll_task()`. Run the tool's work inline in its handler; clients call the tool with `call_tool()` and await the result.

The `Task*` models (`Task`, `TaskMetadata`, `CreateTaskResult`, `GetTaskResult`, ...) remain in `mcp_types` as types-only definitions; the `TASK_*` constants and the `TaskExecutionMode` alias are gone (see [Removed type aliases and classes](#removed-type-aliases-and-classes)).

## Transports

### `streamablehttp_client` removed

The deprecated `streamablehttp_client` function is gone. Use `streamable_http_client`, which takes only `url`, `http_client`, and `terminate_on_close`: headers, timeouts, and auth move onto an `httpx2.AsyncClient` you pass as `http_client` (see [Client transports](client/transports.md)).

**Before (v1):**

```python
from mcp.client.streamable_http import streamablehttp_client

async with streamablehttp_client(
    url="http://localhost:8000/mcp",
    headers={"Authorization": "Bearer token"},
    timeout=30,
    sse_read_timeout=300,
    auth=my_auth,
) as (read_stream, write_stream, get_session_id):
    ...
```

**After (v2):**

```python
import httpx2
from mcp.client.streamable_http import streamable_http_client

async with httpx2.AsyncClient(
    headers={"Authorization": "Bearer token"},
    timeout=httpx2.Timeout(30, read=300),
    auth=my_auth,
    follow_redirects=True,
) as http_client:
    async with streamable_http_client(
        url="http://localhost:8000/mcp",
        http_client=http_client,
    ) as (read_stream, write_stream):
        ...
```

v1's internal client set `follow_redirects=True`; set it explicitly when supplying your own `httpx2.AsyncClient` to preserve that behavior.

### `get_session_id` callback removed from `streamable_http_client`

`streamable_http_client` now yields `(read_stream, write_stream)` instead of a 3-tuple: the `get_session_id` callback is gone, along with the `GetSessionIdCallback` alias (`from mcp.client.streamable_http import GetSessionIdCallback` raises `ImportError`; inline `Callable[[], str | None]` if you still need the type).

**Before (v1):**

```python
from mcp import ClientSession
from mcp.client.streamable_http import streamable_http_client

async with streamable_http_client(url) as (read_stream, write_stream, get_session_id):
    async with ClientSession(read_stream, write_stream) as session:
        await session.initialize()
        session_id = get_session_id()
```

**After (v2):** unpack two values. If you still need the session ID, read the `mcp-session-id` response header with an `httpx2` event hook:

```python
import httpx2

from mcp import ClientSession
from mcp.client.streamable_http import streamable_http_client

captured_session_ids: list[str] = []


async def capture_session_id(response: httpx2.Response) -> None:
    session_id = response.headers.get("mcp-session-id")
    if session_id:
        captured_session_ids.append(session_id)


http_client = httpx2.AsyncClient(event_hooks={"response": [capture_session_id]}, follow_redirects=True)

async with http_client:
    async with streamable_http_client(url, http_client=http_client) as (read_stream, write_stream):
        async with ClientSession(read_stream, write_stream) as session:
            await session.initialize()
            session_id = captured_session_ids[0] if captured_session_ids else None
```

### `StreamableHTTPTransport` parameters removed

`StreamableHTTPTransport` now takes only `url`; passing `headers`, `timeout`, `sse_read_timeout`, or `auth` raises `TypeError`. Configure them on the `httpx2.AsyncClient` you pass to `streamable_http_client(url, http_client=...)` (`headers=`, `auth=`, `timeout=httpx2.Timeout(timeout, read=sse_read_timeout)`), as shown in [`streamablehttp_client` removed](#streamablehttp_client-removed) and [Client transports](client/transports.md). `sse_client` keeps all four parameters; only the streamable HTTP transport changed.

### `StreamableHTTPTransport.protocol_version` and `MCP_PROTOCOL_VERSION` removed

`StreamableHTTPTransport` no longer exposes a `protocol_version` attribute; read the negotiated version from [`session.protocol_version` or `client.protocol_version`](#clientsessionget_server_capabilities-replaced-by-properties). The header-name constant moved from `mcp.client.streamable_http` to `mcp.shared.inbound` under a new name (same value, `"mcp-protocol-version"`):

**Before (v1):**

```python
from mcp.client.streamable_http import MCP_PROTOCOL_VERSION

headers = {MCP_PROTOCOL_VERSION: "2025-11-25"}
```

**After (v2):**

```python
from mcp.shared.inbound import MCP_PROTOCOL_VERSION_HEADER

headers = {MCP_PROTOCOL_VERSION_HEADER: "2025-11-25"}
```

### Streamable HTTP: non-2xx responses now surface as per-request JSON-RPC errors

A non-2xx response no longer tears down the transport: v2 resolves the failing request with a JSON-RPC error, raised as `MCPError` from that one call, and the connection stays usable.

| Server response | v1 | v2 |
| --- | --- | --- |
| 404, session established | `McpError` with positive code `32600` | `MCPError(-32600, 'Session terminated')` |
| 404, no session yet | `McpError` with positive code `32600` | `MCPError(-32601, 'Not Found')` |
| Any other 4xx/5xx | `httpx.HTTPStatusError` escapes the context as `ExceptionGroup` | `MCPError(-32603, 'Server returned an error response')` |
| Any of the above with a JSON-RPC error body | body ignored | body's error surfaced verbatim, e.g. `MCPError(-32602, 'Invalid params')` |

Two v1 patterns silently stop working: an `except* httpx.HTTPStatusError` around the transport context is dead code, and a session-expiry check on `error.code == 32600` never matches the now-negative `-32600`.

**Before (v1):**

```python
import httpx
from mcp import ClientSession
from mcp.client.streamable_http import streamable_http_client
from mcp.shared.exceptions import McpError

try:
    async with streamable_http_client(url) as (read, write, _):
        async with ClientSession(read, write) as session:
            await session.initialize()
            try:
                await session.list_tools()
            except McpError as exc:
                if exc.error.code == 32600:  # 404: "Session terminated"
                    ...  # rebuild the connection
                else:
                    raise
except* httpx.HTTPStatusError:
    ...  # any other 4xx/5xx: rebuild the connection
```

**After (v2):**

```python
from mcp import ClientSession, MCPError
from mcp.client.streamable_http import streamable_http_client
from mcp_types import INVALID_REQUEST  # -32600

async with streamable_http_client(url) as (read, write):
    async with ClientSession(read, write) as session:
        await session.initialize()
        try:
            await session.list_tools()
        except MCPError as exc:
            if exc.code == INVALID_REQUEST and exc.message == "Session terminated":
                ...  # 404 with an established session: rebuild the connection
            else:
                raise  # other 4xx/5xx: this call failed, the session is still usable
```

Catch `MCPError` per call ([`McpError` renamed to `MCPError`](#mcperror-renamed-to-mcperror)); connect-level failures such as `httpx2.ConnectError` still escape the transport context as an `ExceptionGroup`, so keep context-level handling for those only. The `-32603` fallback (a non-2xx body that is not JSON-RPC) is diagnosed in [Troubleshooting](troubleshooting.md#mcperror-server-returned-an-error-response).

### `mcp.os.win32.utilities` process helpers changed

Code that reused the SDK's Windows process helpers (e.g. a copy of the stdio
client's cleanup) needs two edits: the deprecated `terminate_windows_process` is
gone with no replacement (`stdio_client` handles termination itself), and
`terminate_windows_process_tree` no longer accepts `timeout_seconds` — the argument
was already ignored, so drop it. `terminate_posix_process_tree` keeps its signature.

**Before (v1):**

```python
from mcp.os.win32.utilities import terminate_windows_process_tree

await terminate_windows_process_tree(process, timeout_seconds)
```

**After (v2):**

```python
from mcp.os.win32.utilities import terminate_windows_process_tree

await terminate_windows_process_tree(process)
```

### `stdio_server` diverts fd 0 and fd 1 while serving

While `stdio_server()` (and so any stdio `MCPServer`) is serving, the protocol runs on private duplicates of the standard descriptors: fd 0 points at the null device and fd 1 at stderr, both restored on exit. Ordinary servers need no change; code that inspects fd 0 or fd 1 during a session now sees the diversions, not the client pipe. Explicit `stdin=`/`stdout=` streams passed to `stdio_server()` are used as given and not diverted.

The pattern that breaks is a POSIX watchdog thread polling fd 0 to detect a vanished client: the null device never reports `POLLHUP` or `POLLERR` and always polls readable, so a hangup wait blocks forever and an any-event wait fires at startup. Watch the parent process instead:

**Before (v1):**

```python
import os
import select
import threading


def watch_stdin() -> None:
    poller = select.poll()
    poller.register(0, select.POLLHUP | select.POLLERR)
    poller.poll()  # blocked until the client closes the pipe
    os._exit(0)


threading.Thread(target=watch_stdin, daemon=True).start()
```

**After (v2):**

```python
import os
import threading
import time


def watch_parent() -> None:
    parent = os.getppid()
    while os.getppid() == parent:  # reparented once the launching parent dies
        time.sleep(1)
    os._exit(0)


threading.Thread(target=watch_parent, daemon=True).start()
```

### WebSocket transport removed

`mcp.client.websocket.websocket_client`, `mcp.server.websocket.websocket_server`, and the `mcp[ws]` extra are gone (WebSocket was never part of the MCP spec). Use the streamable HTTP transport instead.

**Before (v1):**

```python
from mcp.client.websocket import websocket_client

async with websocket_client("ws://localhost:8000/ws") as (read, write):
    ...
```

**After (v2):**

```python
from mcp.client.streamable_http import streamable_http_client

async with streamable_http_client("http://localhost:8000/mcp") as (read, write):
    ...
```

Or connect with `Client("http://localhost:8000/mcp")` directly. On the server, drop the `websocket_server` ASGI route and mount `streamable_http_app()` (available on `MCPServer` and the lowlevel `Server`), or call `mcp.run(transport="streamable-http")`. See [Client transports](client/transports.md) and [Running your server](run/index.md).

## OAuth and server auth

### `RFC7523OAuthClientProvider` and `JWTParameters` removed

`RFC7523OAuthClientProvider` and its `JWTParameters` model are removed from `mcp.client.auth.extensions.client_credentials`; importing them raises `ImportError`. Use `ClientCredentialsOAuthProvider` for a client secret or `PrivateKeyJWTOAuthProvider` for a JWT.

**Before (v1):**

```python
from mcp.client.auth.extensions.client_credentials import JWTParameters, RFC7523OAuthClientProvider

provider = RFC7523OAuthClientProvider(
    server_url=server_url,
    client_metadata=client_metadata,
    storage=storage,
    jwt_parameters=JWTParameters(issuer="my-client-id", subject="my-client-id", jwt_signing_key=key_pem),
)
```

**After (v2):**

```python
from mcp.client.auth.extensions.client_credentials import PrivateKeyJWTOAuthProvider, SignedJWTParameters

provider = PrivateKeyJWTOAuthProvider(
    server_url=server_url,
    storage=storage,
    client_id="my-client-id",
    assertion_provider=SignedJWTParameters(
        issuer="my-client-id", subject="my-client-id", signing_key=key_pem
    ).create_assertion_provider(),
)
```

The provider sends a `client_credentials` grant with `private_key_jwt` client authentication rather than the `jwt-bearer` grant `RFC7523OAuthClientProvider` sent. `SignedJWTParameters` drops the `jwt_` prefix (`jwt_signing_key` -> `signing_key`, `jwt_signing_algorithm` -> `signing_algorithm`, `jwt_lifetime_seconds` -> `lifetime_seconds`), renames `claims` to `additional_claims`, and has no `audience` (the provider supplies the authorization server's issuer); a prebuilt `JWTParameters(assertion=...)` becomes `assertion_provider=static_assertion_provider(token)`. See [OAuth clients](client/oauth-clients.md#machine-to-machine). For an enterprise ID-JAG `jwt-bearer` grant, use `IdentityAssertionOAuthProvider` ([Identity assertion](client/identity-assertion.md)). The `authorization_code` flow with `private_key_jwt` token-endpoint authentication has no v2 equivalent.

### OAuth metadata URLs no longer gain a trailing slash

`OAuthMetadata`, `ProtectedResourceMetadata`, `OAuthClientMetadata`, and `AuthSettings` now set
`url_preserve_empty_path=True`: a path-less URL parsed from a string (constructor argument or
wire JSON) keeps its empty path instead of gaining a trailing slash.

```python
from mcp.shared.auth import OAuthClientMetadata

meta = OAuthClientMetadata.model_validate({"redirect_uris": ["http://localhost:8080"]})
print(meta.model_dump(mode="json")["redirect_uris"])
# v1: ['http://localhost:8080/']
# v2: ['http://localhost:8080']
```

The client sends `redirect_uris[0]` verbatim in the `/authorize` and token requests, and
authorization servers match redirect URIs by exact string comparison
([RFC 6749](https://datatracker.ietf.org/doc/html/rfc6749) §3.1.2.3). If a v1 run registered such
a URI via Dynamic Client Registration (as `http://localhost:8080/`) and your `TokenStorage` still
returns that client information, clear it so the client re-registers, or write the URI with an
explicit trailing slash. Redirect URIs with a path (`http://localhost:3000/callback`) and values
passed as `AnyUrl(...)`/`AnyHttpUrl(...)` objects are unchanged.

Likewise, a path-less string `issuer_url` or `resource_server_url` in `AuthSettings` is now
advertised as `https://as.example.com` rather than `https://as.example.com/`; update anything
that pinned the old string form.

### OAuth `callback_handler` returns `AuthorizationCodeResult`

The `callback_handler` passed to `OAuthClientProvider` must return an `AuthorizationCodeResult` instead of a `(code, state)` tuple; an unchanged tuple-returning handler crashes the authorization flow (`AttributeError`). Forward the redirect's `iss` parameter as well: the provider now validates it against the authorization server's issuer ([RFC 9207](https://datatracker.ietf.org/doc/html/rfc9207), [SEP-2468](https://github.com/modelcontextprotocol/modelcontextprotocol/pull/2468)) and raises `OAuthFlowError` on a mismatch, or on a missing `iss` when the server advertises `authorization_response_iss_parameter_supported`.

**Before (v1):**

```python
async def callback_handler() -> tuple[str, str | None]:
    params = parse_qs(urlparse(await wait_for_redirect()).query)
    return params["code"][0], params.get("state", [None])[0]
```

**After (v2):**

```python
from mcp.client.auth import AuthorizationCodeResult


async def callback_handler() -> AuthorizationCodeResult:
    params = parse_qs(urlparse(await wait_for_redirect()).query)
    return AuthorizationCodeResult(
        code=params["code"][0],
        state=params.get("state", [None])[0],
        iss=params.get("iss", [None])[0],
    )
```

See [OAuth clients](client/oauth-clients.md) for the full handler contract.

### `scopes=` renamed to `scope=` on the client-credentials providers

`ClientCredentialsOAuthProvider` and `PrivateKeyJWTOAuthProvider` renamed the `scopes` keyword to `scope`; the value is unchanged (a single space-separated string).

**Before (v1):**

```python
provider = ClientCredentialsOAuthProvider(server_url, storage, client_id, client_secret, scopes="read write")
```

**After (v2):**

```python
provider = ClientCredentialsOAuthProvider(server_url, storage, client_id, client_secret, scope="read write")
```

### `timeout` parameter removed from `OAuthClientProvider`

`OAuthClientProvider` no longer accepts a `timeout` argument, and `OAuthContext.timeout` is gone. The value was never read, so dropping it changes no behavior.

**Before (v1):**

```python
provider = OAuthClientProvider(server_url, client_metadata, storage, timeout=120.0)
```

**After (v2):**

```python
provider = OAuthClientProvider(server_url, client_metadata, storage)
```

If you passed `timeout` intending to bound the wait for the user to authorize, apply that bound where you actually wait: in your `redirect_handler`/`callback_handler`, e.g. `with anyio.fail_after(120): ...`.

### Client rejects authorization server metadata with a mismatched `issuer`

`OAuthClientProvider` now requires the authorization server metadata `issuer` to be
string-equal to the URL taken from the protected resource metadata's `authorization_servers`
list ([RFC 8414](https://datatracker.ietf.org/doc/html/rfc8414) section 3.3,
[SEP-2468](https://github.com/modelcontextprotocol/modelcontextprotocol/pull/2468)). v1 never
compared the two, so a pairing that disagrees (even by a trailing slash) authenticated under v1
and now fails discovery. For example, protected resource metadata advertising
`"authorization_servers": ["https://as.example.com"]` against authorization server metadata
with `"issuer": "https://as.example.com/"` raises:

```text
OAuthFlowError: Authorization server metadata issuer mismatch: https://as.example.com/ != https://as.example.com
```

There is no client-side override; make the two strings identical. If the MCP server also uses
this SDK, `AuthSettings(issuer_url=...)` is what it advertises in `authorization_servers`, so
set it to exactly the authorization server's `issuer`. See
[OAuth metadata URLs no longer gain a trailing slash](#oauth-metadata-urls-no-longer-gain-a-trailing-slash)
for how v2 preserves the exact string form of these URLs.

### OAuth client requests `offline_access` and sends `prompt=consent`

Per [SEP-2207](https://github.com/modelcontextprotocol/modelcontextprotocol/pull/2207), when the authorization server's metadata lists `offline_access` in `scopes_supported` and the client's `grant_types` include `refresh_token` (the default), the client appends `offline_access` to the requested scope; whenever `offline_access` is in the scope, the authorization request also carries `prompt=consent`. Unchanged v1 code sends a different authorization URL:

**Before (v1):**

```text
https://as.example.com/authorize?...&scope=read
```

**After (v2):**

```text
https://as.example.com/authorize?...&scope=read+offline_access&prompt=consent
```

End users see the provider's consent screen on every authorization instead of being re-authorized silently, the requested scope is broader by `offline_access` (an authorization server that ties refresh tokens to that scope now issues them), and an authorization server that does not allow `offline_access` for the client can fail the flow with `invalid_scope`.

To send the v1 authorization request again, remove the `refresh_token` grant:

```python
from pydantic import AnyUrl

from mcp.shared.auth import OAuthClientMetadata

client_metadata = OAuthClientMetadata(
    client_name="my-client",
    redirect_uris=[AnyUrl("http://localhost:3000/callback")],
    grant_types=["authorization_code"],
)
```

Unlike v1, this also drops `refresh_token` from the Dynamic Client Registration payload, so authorization servers that gate refresh tokens on the registered grants stop issuing them. If `offline_access` reaches the scope another way (a `WWW-Authenticate` challenge or the resource's `scopes_supported`), `prompt=consent` is still sent; there is no separate switch for it.

### Dynamic Client Registration now sends `application_type`

Per [SEP-837](https://github.com/modelcontextprotocol/modelcontextprotocol/pull/837), the registration body now always includes `application_type`, defaulting to `"native"` (loopback/`localhost` redirect URIs). A browser-based client served from a non-local host must set `"web"`, or an OIDC authorization server may reject its non-loopback redirect URIs:

```python
from pydantic import AnyUrl

from mcp.shared.auth import OAuthClientMetadata

client_metadata = OAuthClientMetadata(
    redirect_uris=[AnyUrl("https://app.example.com/callback")],
    application_type="web",
)
```

Non-OIDC authorization servers ignore the parameter.

### Stricter client authentication at `/token` and `/revoke`

Two behavior changes affect authorization servers embedded via `auth_server_provider=` / `create_auth_routes`.

**Failed client authentication at `/token` returns `invalid_client`.** v1 answered every client-authentication failure at `/token` (unknown `client_id`, missing, wrong, or expired secret) with 401 `unauthorized_client`; v2 sends 401 `invalid_client` ([RFC 6749](https://datatracker.ietf.org/doc/html/rfc6749) §5.2). Update tests, dashboards, or non-SDK clients that match on the error code.

**Before (v1):**

```text
401 {"error":"unauthorized_client","error_description":"Invalid client_id"}
```

**After (v2):**

```text
401 {"error":"invalid_client","error_description":"Invalid client_id"}
```

**Secret-method client records with no stored secret are rejected.** In v1, a client record whose `token_endpoint_auth_method` was `client_secret_post` or `client_secret_basic` but whose `client_secret` was `None` authenticated without a valid secret. v2 rejects it before grant processing: `/token` returns 401 `invalid_client` and `/revoke` returns 401 `unauthorized_client`, both with "Client is registered for secret-based authentication but has no stored secret". Clients registered through `/register` are unaffected (they always receive a secret); hand-provisioned records need a stored secret or a public auth method.

**Before (v1):**

```python
from mcp.shared.auth import OAuthClientInformationFull

LEGACY_CLIENT = OAuthClientInformationFull(
    client_id="legacy-client",
    client_secret=None,  # no secret stored
    token_endpoint_auth_method="client_secret_post",  # but a secret-based method
    redirect_uris=["http://localhost:1234/cb"],
)
```

**After (v2):**

```python
from mcp.shared.auth import OAuthClientInformationFull

LEGACY_CLIENT = OAuthClientInformationFull(
    client_id="legacy-client",
    token_endpoint_auth_method="none",  # public client, no secret expected
    redirect_uris=["http://localhost:1234/cb"],
)
```

## Stricter protocol validation and wire behavior

### Server handler results are validated against the protocol schema

Results returned from server handlers are now validated against the negotiated protocol version's schema before they are sent. On failure the server logs `handler for <method> returned an invalid result` with the pydantic error, and the client receives `-32603` (`INTERNAL_ERROR`, "Handler returned an invalid result"). The common case is a hand-built `Tool` whose input schema lacks the required `"type": "object"` (schemas generated by `@mcp.tool()` already conform):

**Before (v1):**

```python
from mcp.types import Tool

Tool(name="ping", inputSchema={})
```

**After (v2):**

```python
from mcp_types import Tool

Tool(name="ping", input_schema={"type": "object"})
```

### Client validates inbound traffic against the protocol schema

The client (`Client`, and `ClientSession` beneath it) validates server results against the negotiated protocol version's schema before parsing them. Spec-invalid output that v1's lenient parse accepted now raises `pydantic.ValidationError` from `list_tools()`, `call_tool()`, and the other request methods (same common cause as the previous section). Catch `pydantic.ValidationError` around calls to servers you don't control, or fix the server.

### Every outbound request now carries a `_meta` envelope; OpenTelemetry is on by default

Every request the SDK sends (client-to-server and server-to-client, at every negotiated protocol version) now includes `params._meta`, even where v1 sent no `params` at all. No setting restores the v1 wire shape, so update fixtures, mock servers, and snapshot tests that assert on raw request bytes.

**Before (v1)**, against a 2025-11-25 peer:

```text
{"method":"ping","jsonrpc":"2.0","id":1}
{"method":"tools/list","jsonrpc":"2.0","id":2}
```

**After (v2)**, same peer:

```text
{"jsonrpc":"2.0","id":2,"method":"ping","params":{"_meta":{}}}
{"jsonrpc":"2.0","id":3,"method":"tools/list","params":{"_meta":{}}}
```

The envelope is the carrier for OpenTelemetry trace propagation ([SEP-414](https://github.com/modelcontextprotocol/modelcontextprotocol/pull/414)), which v2 enables by default: every server wraps each inbound message in a span and the client opens a span per outbound request, both under the `mcp-python-sdk` tracer. With no OpenTelemetry SDK configured these are no-ops. If your application already installs a global tracer provider, upgrading starts recording MCP client and server spans and injects a W3C `traceparent` into every outbound `_meta`, propagating your trace ids to the servers you call. To opt out, filter the `mcp-python-sdk` instrumentation scope in your span pipeline, or remove the server middleware as shown in [OpenTelemetry](run/opentelemetry.md); the client-side span and `traceparent` injection have no switch.

## Testing utilities

### `create_connected_server_and_client_session` removed

`mcp.shared.memory.create_connected_server_and_client_session` is gone. `Client` takes the `MCPServer` or lowlevel `Server` instance directly and yields an already-connected client (see [Testing](get-started/testing.md)).

**Before (v1):**

```python
from mcp.shared.memory import create_connected_server_and_client_session

async with create_connected_server_and_client_session(server) as session:
    result = await session.call_tool("my_tool", {"x": 1})
```

**After (v2):**

```python
from mcp import Client

async with Client(server) as client:
    result = await client.call_tool("my_tool", {"x": 1})
```

`Client` takes the same callbacks (`sampling_callback`, `elicitation_callback`, `list_roots_callback`, `logging_callback`, `message_handler`, `client_info`), keeps `raise_exceptions`, and takes `read_timeout_seconds` as a `float` (see [Timeouts take `float` seconds instead of `timedelta`](#timeouts-take-float-seconds-instead-of-timedelta)). The request methods (`call_tool`, `list_tools`, `read_resource`, `get_prompt`, ...) live on `client`; code that drove the yielded `ClientSession` directly (`session.send_request(...)`, `session.send_notification(...)`) finds it at `client.session`.

`Client(server)` does not open the connection the v1 helper did: its default `mode="auto"` negotiates protocol `2026-07-28` over a direct in-process dispatcher (no JSON-RPC framing), so two things ported v1 tests commonly relied on are gone:

- Server-to-client requests: `ctx.elicit()`, `ctx.session.create_message()`, and `ctx.session.list_roots()` inside a handler raise `NoBackChannelError` (an `MCPError`) even with the matching client callback set, so the test's `call_tool` fails (see [Troubleshooting](troubleshooting.md)).
- Wire `_meta`: `ctx.request_context.meta` carries no `progress_token`, so handlers that gate progress on the token go silent; `ctx.report_progress()` still reaches the client's `progress_callback`.

Pass `Client(server, mode="legacy")` to reproduce v1's initialize handshake over the in-memory JSON-RPC transport, which restores both; [resolvers](handlers/dependencies.md) are the modern replacement for push-style elicitation and sampling. [Protocol versions](protocol-versions.md) covers `mode`.

For the raw stream pair and a manual `ClientSession` (lowlevel `Server` shown), `create_client_server_memory_streams` remains, but the streams it yields are `ContextReceiveStream`/`ContextSendStream` wrappers: `send`, `receive`, iteration, `close`/`aclose`, and `clone` work, while the anyio-only `send_nowait`, `receive_nowait`, and `statistics()` are gone:

```python
import anyio
from mcp.client.session import ClientSession
from mcp.shared.memory import create_client_server_memory_streams

async with create_client_server_memory_streams() as (client_streams, server_streams):
    async with anyio.create_task_group() as tg:
        tg.start_soon(lambda: server.run(*server_streams, server.create_initialization_options()))
        async with ClientSession(*client_streams) as session:
            await session.initialize()
            ...
        tg.cancel_scope.cancel()
```

## Need Help?

If you encounter issues during migration:

1. Check the [API Reference](api/mcp/index.md) for updated method signatures
2. Review the [examples](https://github.com/modelcontextprotocol/python-sdk/tree/main/examples) for updated usage patterns
3. Open an issue on [GitHub](https://github.com/modelcontextprotocol/python-sdk/issues) if you find a bug or need further assistance
