# Provider startup and handshake protocol

OpenTofu runs a provider as a local child process. It passes startup parameters
through the environment, reads one handshake line from the child's standard
output, and connects to the advertised socket using gRPC. Provider RPC requests
are sent over that connection, not over stdin or stdout.

This description follows the [source revisions](sources.md) inspected for these
notes. “Core” means the OpenTofu process, and “provider” means the external
executable. The ordinary managed launch described below differs from
[debugger attachment](cpp-provider.md#attaching-a-debugger).

## Sequence

```mermaid
sequenceDiagram
    participant Core as OpenTofu Core
    participant Provider as Provider process
    Core->>Provider: Execute binary; supply environment
    Note over Provider: Check cookie and select provider major version
    Note over Provider: Prepare listener, TLS credentials, and gRPC services
    Provider-->>Core: One flushed stdout handshake line
    Core->>Provider: Connect to advertised socket; perform mutual TLS
    Core->>Provider: Open broker and stdio streams
    Core->>Provider: Provider RPCs, such as GetProviderSchema
    Note over Core,Provider: Provider configuration and resource operations follow
    Core->>Provider: GRPCController.Shutdown
    Note over Provider: Stop server, release resources, exit process
```

The broker and stdio streams can overlap provider RPCs. This diagram is not a
requirement that every RPC occur serially. A health check is also available, but
is not a mandatory step in Core's ordinary provider factory.

## 1. Executable launch

Core selects a provider executable from its installed package or development
override directory. Its filename starts with `terraform-provider-<type>`, with
an optional suffix starting with `_` or `.`. For example,
`terraform-provider-example`, `terraform-provider-example_v1.0.0`, and
`terraform-provider-example.exe` are recognized. The historical `terraform-`
prefix remains in use by OpenTofu.

Core executes the selected file directly, with no extra command-line arguments
for the handshake. It inherits the host environment and adds the variables
below. A provider must not require a `serve` subcommand for this launch mode.
Provider configuration is supplied later through RPCs; it is not a startup
argument or a document on stdin.

The child inherits Core's stdin. There is no handshake request to read there.
Core captures stdout to read the handshake and captures stderr for logging.
Keep startup diagnostics on stderr, including diagnostics produced by libraries
and static initialization.

Sources: [Core launch and discovery](sources.md#opentofu-core),
[go-plugin client](sources.md#go-plugin-v180).

## 2. Environment supplied by Core

| Variable | Meaning |
| --- | --- |
| `TF_PLUGIN_MAGIC_COOKIE` | Fixed marker indicating an expected plugin launch; exact value below. |
| `PLUGIN_PROTOCOL_VERSIONS` | Comma-separated decimal provider protocol major versions Core accepts. The inspected Core advertises 5 and 6; order is unspecified. |
| `PLUGIN_MIN_PORT` | Lower bound, inclusive, for a TCP listener's port selection. The inspected client's default is `10000`. |
| `PLUGIN_MAX_PORT` | Upper bound, inclusive, for TCP port selection. The inspected client's default is `25000`. |
| `PLUGIN_CLIENT_CERT` | Core's public X.509 certificate in PEM format, including actual newlines, when automatic mutual TLS is enabled. No private key is included. |

The cookie value is exactly:

```text
d602bf8f470bc67ca7faa0386276bbdd4330efaf76d1a219cb4d6991ca9872b2
```

The reference server checks that the cookie matches before starting normal
service. A missing or incorrect cookie causes a diagnostic on stderr and exit
status 1, without a handshake. An explicitly implemented debug mode can bypass
this check, as described in the C++ guide.

The cookie is public and constant. It prevents accidental direct execution; it
is not a credential and does not authenticate the parent process.

Other `go-plugin` environment variables are relevant only to particular
configurations:

| Variable | Reference behavior |
| --- | --- |
| `PLUGIN_UNIX_SOCKET_DIR` | Directory for creating Unix sockets; otherwise the Go runtime's temporary directory is used. |
| `PLUGIN_UNIX_SOCKET_GROUP` | Optional group to own the Unix socket, with group write access. |
| `PLUGIN_MULTIPLEX_GRPC` | Requests the optional broker multiplexing extension. Core's ordinary provider factory does not enable this extension. |

`TF_DISABLE_PLUGIN_TLS` is read by **Core**, not used as the provider's TLS
negotiation signal. Any nonempty value disables Core's automatic mutual TLS,
including the string `0`. With a clean environment this results in no
`PLUGIN_CLIENT_CERT`. A provider should choose its transport credentials from
the presence of `PLUGIN_CLIENT_CERT`, rather than independently interpreting
`TF_DISABLE_PLUGIN_TLS`.

Sources: [handshake constants and Core TLS switch](sources.md#opentofu-core),
[client, server, and environment constants](sources.md#go-plugin-v180).

## 3. Select the provider protocol major version

Parse `PLUGIN_PROTOCOL_VERSIONS` as a set of integers. Select the greatest
version that both Core and the provider implement. A provider implementing only
major version 6 selects 6 when the list contains 6. A provider implementing both
5 and 6 selects 6 when Core offers both, regardless of list order.

The reference server sorts both sets in descending order. If there is no common
version, or if the environment variable is absent, it advertises its lowest
implemented version as a fallback. Core then rejects a version it does not
support. Invalid integer entries are diagnosed and skipped by the reference
server. For a C++ implementation, failing early with a clear stderr diagnostic
on no overlap is also a useful failure policy, but produces a different Core
error from the reference fallback. Never advertise a version you do not serve.

There are three separate version concepts:

| Concept | Example | Where it appears |
| --- | --- | --- |
| `go-plugin` core protocol version | `1` | First handshake field. This is unrelated to the OpenTofu product version. |
| Provider protocol major version | `5` or `6` | Environment offer, second handshake field, and protobuf package `tfplugin5` or `tfplugin6`. |
| Provider protocol minor version | The `10` in `6.10` | Published `.proto` version and documentation; not a handshake field. |

The handshake's second field is `6`, never `6.10`. Additive minor changes are
handled by protobuf compatibility and capability/feature detection in the
provider protocol.

Core's `Handshake.ProtocolVersion` constant is still 4 for historical reasons.
Its ordinary provider client uses `VersionedPlugins` for 5 and 6 instead. The
constant is not an instruction to advertise 4, and does not add 4 to the
ordinary provider client's offered versions.

Sources: [Core plugin map](sources.md#opentofu-core),
[`protocolVersion` and `checkProtoVersion`](sources.md#go-plugin-v180).

## 4. Create a local listener

The handshake supports two network labels:

| Network | Address field | Reference server default |
| --- | --- | --- |
| `unix` | Filesystem socket path, such as `/tmp/plugin12345` | Non-Windows systems. |
| `tcp` | Host and port, such as `127.0.0.1:10000` | Windows. |

The address is not a URI: do not put `unix://`, `http://`, or `https://` in the
handshake field. A C++ gRPC library may require a URI internally when binding;
that does not change the advertised field format.

Core accepts both labels without imposing the reference server's OS selection.
For TCP, bind to the loopback interface and select an available port in the
advertised inclusive range. The reference TCP server tries ports in ascending
order. For Unix sockets, use a unique filesystem path, restrict access, and
remove the socket when the server terminates. TCP port variables do not apply
to Unix sockets.

Prepare the listener and register services before advertising the address.
The Go server binds before printing and starts accepting immediately after
printing; a C++ server can begin accepting before printing. Avoid advertising
an endpoint whose initialization might still fail.

Source: [server listeners and client address parsing](sources.md#go-plugin-v180).

## 5. Automatic mutual TLS

Ordinary Core-managed providers use automatic mutual TLS by default, including
when the transport is a Unix socket. TLS is layered over the advertised socket,
then gRPC runs over HTTP/2 on that TLS connection.

When `PLUGIN_CLIENT_CERT` is nonempty:

1. Parse its PEM certificate and use it as the trust anchor for verifying the
   client's certificate. Core retains its private key.
2. Generate a provider private key and a self-signed X.509 certificate for this
   process. Retain the private key in the provider.
3. Configure the gRPC server to present that certificate and require and verify
   client certificates against the supplied client certificate. Merely
   requesting a certificate, or accepting any certificate, is insufficient.
4. Encode the provider certificate for handshake field 6 as described below.
5. Accept TLS with a minimum version of TLS 1.2 and HTTP/2 support. Core verifies
   the provider certificate using the certificate from the handshake as its
   trust anchor and the DNS name `localhost` as its expected server name.

For compatibility with the reference certificate generator, use this profile:

| Property | Reference certificate |
| --- | --- |
| Signature/key | ECDSA with a P-521 key, self-signed. |
| Subject alternative name | DNS name `localhost`. This is needed even when listening on `127.0.0.1` or a Unix socket. |
| Extended key usages | Client authentication and server authentication. |
| Key usages | Digital signature, key encipherment, key agreement, and certificate signing. |
| Basic constraints | Valid, with CA flag set. |
| Validity | Starts 30 seconds before generation and ends 262980 hours after generation. |
| Serial | Random integer below 2^128. |

This describes the reference profile, not a requirement to use its organization
name or exact validity period. The wire protocol does not require the exact
key algorithm; any alternative still needs to work with Core's TLS verifier
and the selected TLS stack. Matching the reference profile minimizes variables
during initial interoperability testing.

Certificate encodings are deliberately asymmetric:

| Direction | Encoding |
| --- | --- |
| Core → provider, `PLUGIN_CLIENT_CERT` | PEM text, with header, footer, and newline characters. |
| Provider → Core, handshake field 6 | Standard base64 of the leaf certificate's DER bytes, **without `=` padding**. |

The handshake value is neither PEM, base64 of PEM, nor base64url. It contains
one certificate, not the private key or a concatenated chain. Its decoded DER
must be the certificate actually presented by the server. Core's decoder uses
Go's `base64.RawStdEncoding`.

When `PLUGIN_CLIENT_CERT` is absent or empty, the reference provider serves
plaintext gRPC and leaves field 6 empty. This is useful for an explicitly
selected local debug mode or for Core's TLS-disable mode. A provider that always
serves plaintext will not interoperate with Core's default launch configuration.
The normal exchange bootstraps trust through the inherited environment and
captured stdout; it is not public-CA authentication or a remote plugin discovery
mechanism.

Sources: [Core TLS setting](sources.md#opentofu-core),
[client certificate loading, server TLS setup, and certificate generation](sources.md#go-plugin-v180).

## 6. Write the stdout handshake

Emit this as the first line on the original process stdout, terminated by LF,
and explicitly flush the stream:

```text
1|<provider-major>|<network>|<address>|grpc|<server-certificate>\n
```

Here `\n` denotes one newline byte, not the two literal characters backslash and
`n`. There are six fields separated by five literal `|` characters:

| Field | Content |
| --- | --- |
| 1 | Decimal `go-plugin` core protocol version: `1`. |
| 2 | Selected provider protocol major: `5` or `6`. |
| 3 | `unix` or `tcp`. |
| 4 | Bound socket path or host:port. |
| 5 | Literal `grpc`. Core's provider client does not allow net/rpc. |
| 6 | Unpadded standard base64 DER server certificate for automatic TLS, or empty for plaintext. |

Example plaintext handshake, with an actual newline after the final `|`:

```text
1|6|tcp|127.0.0.1:10000|grpc|
```

Structural TLS example; the angle-bracketed text must be replaced by a real
encoded certificate:

```text
1|6|unix|/tmp/plugin12345|grpc|<unpadded-base64-DER>
```

Do not print a banner, blank line, JSON object, or log entry first. Core parses
the first stdout line; it does not search subsequent lines for a valid
handshake. There is no escaping mechanism for `|` or embedded newlines in
fields. Use an address that does not contain these delimiters.

The inspected client defaults to a one-minute wait for the startup line. It
reads using Go's default `bufio.Scanner` token limit (approximately 64 KiB), so
keep the line comfortably below that size. Ordinary leaf certificates fit.
Writing a newline without flushing a C++ buffered stream can look like a hung
provider.

After parsing the line, Core checks the core version, provider major, network,
transport, and certificate. It then connects. There is no acknowledgement line
on stdin and no second text handshake. Core drains subsequent original stdout
lines but does not treat them as application responses.

### Optional seventh field

`go-plugin` v1.8.0 conditionally appends a seventh boolean field advertising
support for gRPC broker multiplexing when `PLUGIN_MULTIPLEX_GRPC` is nonempty.
If the client actually requests multiplexing, it requires that field to be
`true` and expects the multiplexing protocol to be implemented.

Core's ordinary provider factory does not request this feature. A C++ provider
targeting that factory can emit the six-field form. Do not append `|true` unless
the broker multiplexing protocol is implemented; older clients can also
misinterpret an unconditionally appended seventh field as certificate data.
Broker multiplexing is separate from ordinary HTTP/2 multiplexing of gRPC calls.

Source: [server handshake formatter and client parser](sources.md#go-plugin-v180).

## 7. Services on the gRPC connection

Register the selected provider service, `tfplugin5.Provider` or
`tfplugin6.Provider`, using the corresponding protobuf specification.
For example, the protocol 6 schema method has the gRPC path
`/tfplugin6.Provider/GetProviderSchema`.

Core calls `Dispense("provider")` internally, but in the gRPC implementation
this only constructs a local client wrapper. There is **no remote Dispense RPC**
to implement and no service named simply `provider` to register.

The Go server also registers the following supporting services on the same
endpoint with the same TLS credentials:

| RPC/service | Behavior and relevance |
| --- | --- |
| `plugin.GRPCBroker/StartStream` | Bidirectional stream of `plugin.ConnInfo` for negotiating auxiliary connections. Core opens it when constructing the client. The ordinary provider adapters do not use the broker to create the provider connection. |
| `plugin.GRPCStdio/StreamStdio` | Takes `google.protobuf.Empty`; returns a stream of `plugin.StdioData`. Carries forwarded stdout/stderr, with channel 1 for stdout and 2 for stderr, and a bytes payload. Core opens it when constructing the client. |
| `plugin.GRPCController/Shutdown` | Takes and returns `plugin.Empty`. Requests server shutdown during managed cleanup. |
| `grpc.health.v1.Health/Check` | For service name `plugin`, the reference server reports `SERVING`. Used by `go-plugin`'s `Ping` operation. |
| gRPC reflection | Registered by the reference server; useful for inspection but not used by Core's ordinary provider factory. |

RPC paths have a leading `/`, for example
`/plugin.GRPCController/Shutdown`. The supporting protobuf files are listed in
the [source map](sources.md#supporting-protobuf-definitions).

For a C++ provider, implement the controller and make the supporting streams
behave predictably. An idle stdio stream can remain open without sending
messages; an idle broker stream does not need to invent connection messages.
Handle cancellation and shutdown so these long-lived streams do not prevent
process exit. If you later use auxiliary connections, implement the broker's
connection exchange rather than treating it as an idle stream.

These services are not all hard prerequisites for the first provider RPC.
The inspected client tolerates an unimplemented stdio service, starts the
broker stream asynchronously without surfacing its returned error from client
construction, and only health-checks when `Ping` is invoked. This tolerance is
not equivalent to full `go-plugin` compatibility: omitting shutdown support,
for example, leaves Core relying on forced process termination.

The original stderr pipe remains usable for diagnostics. The Go server
additionally redirects its Go-level stdout/stderr handles into the stdio
service after printing the handshake. A C++ server can keep diagnostic logging
on original stderr without duplicating Go's internal pipe machinery. Never
treat ordinary stdout as a provider response transport.

Sources: [provider adapters](sources.md#opentofu-core),
[gRPC client, server, broker, and stdio implementations](sources.md#go-plugin-v180).

## 8. Provider operation and process shutdown

A successful transport handshake does not configure the provider. Core can
request schema before supplying provider configuration, and may start separate
processes for different phases of a command. Do not assume startup means an
apply is about to happen, or require configured credentials merely to expose
the schema.

There are two distinct stop mechanisms:

- The provider RPC is `tfplugin5.Provider/Stop` for major 5 or
  `tfplugin6.Provider/StopProvider` for major 6. It asks provider operations to
  stop and returns a provider-level response.
- `plugin.GRPCController/Shutdown` ends the plugin server itself during process
  cleanup. Stop accepting work, cancel/close long-lived streams, release the
  listener, and arrange for the executable to exit. Implement it even if the
  provider's own stop method already exists.

Core's managed cleanup closes the broker, attempts the controller shutdown
call, and closes the gRPC connection. The inspected client allows up to two
seconds for process exit after a successful client close before force-killing
the child. A shutdown implementation must not wait indefinitely for the broker
or stdio streams to finish naturally. In C++, schedule shutdown outside a
handler if the server's blocking shutdown API would otherwise wait on that
same handler.

The reference managed server ignores `SIGINT`; Core coordinates interruption
through its provider lifecycle. Debug-mode servers have a different owner and
can use interrupt signals to exit. Do not use stdin EOF as the managed shutdown
protocol.

Sources: [Core provider Stop and Close methods](sources.md#opentofu-core),
[client Kill, gRPC Close, controller, and server signal handling](sources.md#go-plugin-v180).
