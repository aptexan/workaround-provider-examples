# Implementing and testing a C++ provider

This guide accompanies the [wire protocol description](startup-handshake.md).
It describes an implementation plan and manual interoperability checks, not a
tested C++ library or a complete resource implementation.

## Protobuf inputs

Generate the provider service from a chosen OpenTofu protocol definition. For
major version 6 the protobuf package must remain `tfplugin6`, and for major
version 5 it must remain `tfplugin5`. Renaming the package or service changes
the gRPC method paths and prevents interoperability.

Use one minor-version definition per major version. Do not compile several
`tfplugin6.*.proto` files into the same program: they define overlapping symbols
in the same package. Select a definition from a release tag for your supported
OpenTofu release; a file on `main` may describe unreleased additions.

Also generate the supporting services from `go-plugin`'s
`grpc_controller.proto`, `grpc_stdio.proto`, and `grpc_broker.proto`. Preserve
their `plugin` package and their field numbers. The `go_package` option does not
require your implementation to be written in Go. Use the standard gRPC health
service definition if implementing health checks. Links to all these inputs
are in the [source map](sources.md#supporting-protobuf-definitions).

The Go framework does not introduce another startup wire protocol:
`providerserver.Serve` selects `tf5server.Serve` or `tf6server.Serve`, which
configures `go-plugin.Serve`. Its default is major 6; specifying 5 selects major
5. A single call to this framework entry point does not automatically serve
both majors. See [SDK layering](sources.md#go-sdk-layering).

## Implementation sequence

For an initial implementation, supporting one provider major version keeps the
dispatcher simple. The following is pseudocode; the exact TLS and server APIs
depend on your C++ libraries.

```text
main:
    if an explicitly selected debugger mode is active:
        start the separate reattach flow described below
        return when that server stops

    check TF_PLUGIN_MAGIC_COOKIE against the documented constant
    parse PLUGIN_PROTOCOL_VERSIONS
    select a mutually supported major version

    if PLUGIN_CLIENT_CERT is nonempty:
        parse the supplied PEM client certificate
        generate a private key and self-signed localhost server certificate
        configure TLS to require and verify the supplied client's certificate
        handshake_certificate = unpadded_standard_base64(server_certificate_DER)
    else:
        configure plaintext HTTP/2 gRPC
        handshake_certificate = ""

    register the provider service for the selected major
    register controller, broker, stdio, and health services
    bind a local Unix socket or loopback TCP listener
    start the server and check that binding succeeded

    write "1|major|network|address|grpc|handshake_certificate\n" to stdout
    flush stdout

    wait for shutdown while servicing concurrent RPCs
    stop server, cancel streams, remove any Unix socket, and exit
```

The final write in this pseudocode interpolates the named values; it is not a
literal handshake string. With C++ iostreams, explicitly flush with
`std::flush` or a successful equivalent operation. Check write failures.

Keep these implementation details visible during review:

- The incoming certificate is PEM with real newline characters. Do not treat it
  as a filename or as text containing literal `\n` escape sequences.
- The outgoing certificate is DER encoded with standard base64 and stripped of
  padding, not PEM encoded again. Preserve the standard `+` and `/` alphabet.
- Verify the peer certificate against Core's supplied certificate, and require
  a peer certificate. Trusting the system CA bundle alone is insufficient.
- Put `localhost` in the server certificate's DNS SAN, even for a Unix socket.
- Advertise the endpoint actually bound, without a URI scheme. Preserve the
  empty sixth field for plaintext by retaining the final `|`.
- Size your worker/executor arrangement for long-lived broker and stdio streams
  alongside provider calls. An idle stream must not monopolize the only worker
  available to handle schema or shutdown requests.
- Keep shutdown bounded. Cancel idle streams before waiting for all RPCs to
  finish, and avoid calling a blocking shutdown operation in a way that waits
  for its own handler to return.
- Choose explicit gRPC message size limits suitable for schemas and state. The
  inspected `terraform-plugin-go` servers use 256 MiB send and receive limits;
  this is an SDK setting, not a field negotiated in the handshake.

Schema encoding, `DynamicValue`, unknown/null values, diagnostics, and resource
lifecycle semantics still need a separate provider protocol implementation.
Getting a TLS channel open does not establish correctness of those features.

## Exercise the ordinary Core-managed startup

Use this route to test the real cookie, version offer, stdout line, automatic
TLS, and process cleanup. A reattach-only test bypasses those mechanisms.

Create a scratch configuration directory and a separate directory containing
your executable. In this example the provider type is `cppdemo`, so the binary
is named `terraform-provider-cppdemo` (`.exe` on Windows). On Unix it must have
execute permission.

Create `main.tf` in the scratch directory:

```hcl
terraform {
  required_providers {
    cppdemo = {
      source = "registry.opentofu.org/example/cppdemo"
    }
  }
}

provider "cppdemo" {}
```

Create `dev.tfrc` in that directory, replacing `/absolute/path/to/provider-bin`
with the directory containing your binary:

```hcl
provider_installation {
  dev_overrides {
    "registry.opentofu.org/example/cppdemo" = "/absolute/path/to/provider-bin"
  }
  direct {}
}
```

Run from the scratch directory in a POSIX shell:

```sh
env -u TF_DISABLE_PLUGIN_TLS -u TF_REATTACH_PROVIDERS -u PLUGIN_CLIENT_CERT \
  TF_CLI_CONFIG_FILE="$PWD/dev.tfrc" \
  TF_LOG=TRACE \
  tofu providers schema -json >schema.json 2>startup.log
```

The unset variables prevent inherited debugging settings from masking the
normal path. Core generates a fresh `PLUGIN_CLIENT_CERT` for the launched
child. This command is a local schema query, not an apply.

For this minimal development-override check, run the command directly rather
than using `tofu init` to install the example provider. Development overrides
replace runtime provider selection; `init` can still try to find a published
package for the provider address. An example address need not exist in a
registry for the local override to work.

Check that:

1. Core discovers and starts your binary, then logs the selected major version.
2. TLS and supporting streams connect without unexpected errors.
3. `schema.json` contains the schema under your provider's full source address.
4. Core's command terminates, the provider process exits, and any Unix socket is
   removed. Look for `plugin failed to exit gracefully` in the log.

Use a valid `GetProviderSchema` response for this check. If transport succeeds
but that RPC is unimplemented or returns an invalid schema, the command will
still fail; that is a separate stage of progress. Schema output alone also does
not prove orderly shutdown, so examine process exit and the log.

After normal TLS works, test the explicit plaintext mode separately:

```sh
env -u TF_REATTACH_PROVIDERS -u PLUGIN_CLIENT_CERT \
  TF_DISABLE_PLUGIN_TLS=1 \
  TF_CLI_CONFIG_FILE="$PWD/dev.tfrc" \
  TF_LOG=TRACE \
  tofu providers schema -json >schema-plaintext.json 2>startup-plaintext.log
```

The provider should now select plaintext and emit an empty sixth field. Do not
use success in this mode as evidence that the normal TLS exchange works.

Sources: [Core development override behavior and CLI documentation](sources.md#opentofu-core).

## Attaching a debugger

`TF_REATTACH_PROVIDERS` allows Core to use a provider you started yourself. It
does not ask Core to spawn a process or consume the stdout handshake. Use an
explicit C++ debug mode that starts a local gRPC server without requiring the
magic cookie, and reports its endpoint and PID to you.

For example, with a plaintext server already running on `127.0.0.1:12345` as PID
1234, run:

```sh
export TF_REATTACH_PROVIDERS='{"registry.opentofu.org/example/cppdemo":{"Protocol":"grpc","ProtocolVersion":6,"Pid":1234,"Test":true,"Addr":{"Network":"tcp","String":"127.0.0.1:12345"}}}'
tofu providers schema -json
```

Replace the PID, port, source address, and major version with the actual values.
Run this in a configuration requiring that provider. For a Unix socket use
`"Network":"unix"` and put the socket path in `String`.

The JSON fields have these meanings:

| Field | Meaning |
| --- | --- |
| Outer object key | Full provider source address; it must match the configuration. |
| `Protocol` | Transport selector, `grpc`. |
| `ProtocolVersion` | Provider major version, 5 or 6, not the handshake's core version 1. |
| `Pid` | PID of the process already running. |
| `Test` | Set to `true` for this externally owned debug process so Core's cleanup does not kill it. |
| `Addr.Network` / `Addr.String` | Same network and address representations used in the ordinary handshake. |

Core's reattach factory does not enable automatic TLS, and this JSON format has
no certificate exchange fields. The standard debug setup therefore serves
plaintext locally. It does not print the machine-readable stdout handshake;
debug instructions on stdout are acceptable because Core is not reading that
stream in this mode. Do not inherit a stale `PLUGIN_CLIENT_CERT` into the debug
process and accidentally select TLS.

Set `ProtocolVersion` explicitly. Core has a legacy compatibility fallback
treating version 0/omitted as major 5; it is not version negotiation. Set
`Test:true` explicitly too, matching the Go SDK's debug configuration. Stop the
debug process yourself when finished and unset `TF_REATTACH_PROVIDERS` before
testing normal startup again.

Source: [reattach parser and unmanaged provider factory](sources.md#opentofu-core),
[SDK debug serving](sources.md#go-sdk-layering),
[go-plugin reattach](sources.md#go-plugin-v180).

## Troubleshooting by stage

Exact errors vary by gRPC library and OpenTofu version. Use the failure stage
to choose what to inspect.

| Symptom | Inspect |
| --- | --- |
| Core cannot start the executable | Filename, platform/architecture, execute permission, dynamic library availability. The handshake has not begun. |
| Missing/incorrect cookie diagnostic | Whether this is a Core-managed launch or an explicitly selected debug launch. |
| `Unrecognized remote plugin message` | First stdout line, including banners from dependencies, blank lines, and stdout logging. |
| Timeout waiting for plugin to start | Newline and flush, listener initialization, blocking provider initialization before the handshake. |
| Incompatible core API version | First field must be `1`, not `5` or `6`. |
| Incompatible provider API version | Second field must be an offered integer major; do not send `6.10`. |
| Unknown address type or unsupported plugin protocol | Network must be `unix`/`tcp`; transport must be `grpc`; address must not include a URI scheme. |
| Error parsing server certificate | DER versus PEM, standard versus URL-safe base64, padding, or an accidentally appended seventh field. |
| TLS handshake/verification failure | SAN `localhost`, matching advertised/presented certificate, validity, client certificate trust, mandatory peer verification, HTTP/2 negotiation. |
| Connection refused or unavailable | Listener address, successful bind, server accept loop, and premature process exit. |
| Unknown/unimplemented provider RPC | Correct protobuf package, service registration, major version, and actual method implementation. |
| Stdio unavailable or broker stream closes | Supporting services missing or closing prematurely; distinguish these from failure of the provider service itself. |
| Schema succeeds but process is force-killed | Controller shutdown, cancellation of idle streams, blocked shutdown handlers, and main thread exit. |
| Reattach works but ordinary startup fails | Reattach bypassed cookie checks, stdout framing, version selection, and automatic TLS. Test the managed path directly. |

For an interoperability test suite, cover both advertised version orders
(`5,6` and `6,5`), no common version, missing/wrong cookie, TLS and plaintext,
idle-stream shutdown, and the endpoint types you intend to support. Include
negative TLS checks: an unrelated or absent client certificate must be rejected.
These are suggested checks for the C++ implementation; they were not executed
as part of writing these documents.
