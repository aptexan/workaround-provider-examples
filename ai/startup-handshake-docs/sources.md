# Source map and scope of verification

These notes were derived from the three local checkouts and the locally cached
`go-plugin` dependency. The source links below pin revisions so the description
can be checked without depending on a moving `main` branch.

| Repository | Inspected revision |
| --- | --- |
| `opentofu/opentofu` | `105b18bb846f80760ee983c8e49044b2bd30f0a4` |
| `hashicorp/terraform-plugin-go` | `94d759f3663ab3a72961cbfd339654aa76dc24a5` |
| `hashicorp/terraform-plugin-framework` | `00badb1b0584b22f3799c474b9f1cea4b49e5163` |
| `hashicorp/go-plugin` | `v1.8.0`, the dependency declared by both the inspected OpenTofu and terraform-plugin-go checkouts. |

The framework checkout declares `terraform-plugin-go v0.31.0`; the separate
plugin-go checkout is not assumed to be that exact dependency snapshot. The
framework is used here to establish its serving entry points and major-version
selection, while the plugin-go checkout establishes its own server setup.

Verification consisted of tracing code and protobuf definitions, comparing
the major 5 and 6 serving paths, and checking the documentation's internal
links and constants. No C++ provider was built or executed as part of this
documentation task. Implementation pseudocode and manual test commands are
guidance for subsequent integration testing, not reported test results.

## OpenTofu Core

| Source | Relevant behavior |
| --- | --- |
| [Plugin protocol README](https://github.com/opentofu/opentofu/blob/105b18bb846f80760ee983c8e49044b2bd30f0a4/docs/plugin-protocol/README.md) | Original documentation gap; protocol major/minor versioning and release status of definitions. |
| [internal/plugin/plugin.go](https://github.com/opentofu/opentofu/blob/105b18bb846f80760ee983c8e49044b2bd30f0a4/internal/plugin/plugin.go) | Cookie, historical default version 4, and ordinary client's versioned plugin sets for 5 and 6. |
| [internal/command/meta_providers.go](https://github.com/opentofu/opentofu/blob/105b18bb846f80760ee983c8e49044b2bd30f0a4/internal/command/meta_providers.go) | `enableProviderAutoMTLS`, `providerFactory`, `devOverrideProviderFactory`, and `unmanagedProviderFactory`: launch arguments, gRPC selection, TLS, output forwarding, and reattach behavior. |
| [internal/providercache/cached_provider.go](https://github.com/opentofu/opentofu/blob/105b18bb846f80760ee983c8e49044b2bd30f0a4/internal/providercache/cached_provider.go) | `ExecutableFile`: executable naming and package directory selection. |
| [cmd/tofu/main.go](https://github.com/opentofu/opentofu/blob/105b18bb846f80760ee983c8e49044b2bd30f0a4/cmd/tofu/main.go) | Reading `TF_REATTACH_PROVIDERS` and `parseReattachProviders`: JSON fields and network parsing. |
| [internal/plugin/grpc_provider.go](https://github.com/opentofu/opentofu/blob/105b18bb846f80760ee983c8e49044b2bd30f0a4/internal/plugin/grpc_provider.go) | Major 5 adapter, local `GRPCClient` creation, provider `Stop`, and process `Close`. |
| [internal/plugin6/grpc_provider.go](https://github.com/opentofu/opentofu/blob/105b18bb846f80760ee983c8e49044b2bd30f0a4/internal/plugin6/grpc_provider.go) | Major 6 adapter, `GetProviderSchema`, `StopProvider`, and process `Close`. |
| [CLI configuration documentation](https://github.com/opentofu/opentofu/blob/105b18bb846f80760ee983c8e49044b2bd30f0a4/website/docs/cli/config/config-file.mdx) | `TF_CLI_CONFIG_FILE`, development overrides, and their interaction with `init`. |
| [Provider schema command documentation](https://github.com/opentofu/opentofu/blob/105b18bb846f80760ee983c8e49044b2bd30f0a4/website/docs/cli/commands/providers/schema.mdx) | `tofu providers schema -json` as a schema query. |
| [go.mod](https://github.com/opentofu/opentofu/blob/105b18bb846f80760ee983c8e49044b2bd30f0a4/go.mod) | Pin to `go-plugin v1.8.0`. |

## Go SDK layering

| Source | Relevant behavior |
| --- | --- |
| [Framework providerserver.go](https://github.com/hashicorp/terraform-plugin-framework/blob/00badb1b0584b22f3799c474b9f1cea4b49e5163/providerserver/providerserver.go) | `Serve` delegates to tf5server or tf6server; debug option delegates to managed debug serving. |
| [Framework serve_opts.go](https://github.com/hashicorp/terraform-plugin-framework/blob/00badb1b0584b22f3799c474b9f1cea4b49e5163/providerserver/serve_opts.go) | Protocol 6 default and validation of 5/6 selections. |
| [tf5server/server.go](https://github.com/hashicorp/terraform-plugin-go/blob/94d759f3663ab3a72961cbfd339654aa76dc24a5/tfprotov5/tf5server/server.go) | Major 5 `Serve`, cookie, gRPC configuration, message size limits, and debug reattach output. |
| [tf6server/server.go](https://github.com/hashicorp/terraform-plugin-go/blob/94d759f3663ab3a72961cbfd339654aa76dc24a5/tfprotov6/tf6server/server.go) | Corresponding major 6 serving configuration. |
| [tf6server/plugin.go](https://github.com/hashicorp/terraform-plugin-go/blob/94d759f3663ab3a72961cbfd339654aa76dc24a5/tfprotov6/tf6server/plugin.go) | Registers `tfplugin6.Provider` on the gRPC server supplied by go-plugin. |
| [Plugin-go go.mod](https://github.com/hashicorp/terraform-plugin-go/blob/94d759f3663ab3a72961cbfd339654aa76dc24a5/go.mod) | Pin to `go-plugin v1.8.0`. |

## go-plugin v1.8.0

This dependency contains the actual text handshake and TLS implementation;
those details are not defined by the provider `.proto` files. Its source was
available in the local Go module cache.

| Source | Relevant behavior |
| --- | --- |
| [client.go](https://github.com/hashicorp/go-plugin/blob/v1.8.0/client.go) | `NewClient`: defaults; `Start`: environment and stdout parser; `loadServerCert`: raw base64 DER decoding; `checkProtoVersion`: compatibility; `reattach`: test ownership; `Kill`: shutdown and forced exit. |
| [server.go](https://github.com/hashicorp/go-plugin/blob/v1.8.0/server.go) | `protocolVersion`: highest common version/fallback; `Serve`: cookie, TLS, handshake formatting, stdio redirection, signals; `serverListener*`: socket selection. |
| [mtls.go](https://github.com/hashicorp/go-plugin/blob/v1.8.0/mtls.go) | `generateCert`: reference key type and X.509 profile. |
| [constants.go](https://github.com/hashicorp/go-plugin/blob/v1.8.0/constants.go) | Unix socket and broker multiplexing environment names. |
| [grpc_client.go](https://github.com/hashicorp/go-plugin/blob/v1.8.0/grpc_client.go) | TLS/plaintext dialing, stream setup, local `Dispense`, health `Ping`, and controller shutdown on `Close`. |
| [grpc_server.go](https://github.com/hashicorp/go-plugin/blob/v1.8.0/grpc_server.go) | Service registration, health name `plugin`, reflection, and server stop. |
| [grpc_controller.go](https://github.com/hashicorp/go-plugin/blob/v1.8.0/grpc_controller.go) | `Shutdown` stops the gRPC server. |
| [grpc_stdio.go](https://github.com/hashicorp/go-plugin/blob/v1.8.0/grpc_stdio.go) | Stdio streaming, channel dispatch, and unavailable/unimplemented-service tolerance. |
| [grpc_broker.go](https://github.com/hashicorp/go-plugin/blob/v1.8.0/grpc_broker.go) | Bidirectional broker stream and auxiliary connection mechanism. |

Some comments describe older behavior. In particular, comments concerning a
base64 JSON server configuration in `grpc_server.go` do not define the current
sixth handshake field. The active `server.go` formatter and `client.go` parser
use that field for the raw-base64 DER certificate. Similarly, Core's historical
default protocol constant does not describe the effective offered majors in
its ordinary provider client. These notes follow the executed code paths.

## Supporting protobuf definitions

- [go-plugin grpc_controller.proto](https://github.com/hashicorp/go-plugin/blob/v1.8.0/internal/plugin/grpc_controller.proto)
- [go-plugin grpc_stdio.proto](https://github.com/hashicorp/go-plugin/blob/v1.8.0/internal/plugin/grpc_stdio.proto)
- [go-plugin grpc_broker.proto](https://github.com/hashicorp/go-plugin/blob/v1.8.0/internal/plugin/grpc_broker.proto)
- [OpenTofu provider protocol definitions at the inspected revision](https://github.com/opentofu/opentofu/tree/105b18bb846f80760ee983c8e49044b2bd30f0a4/docs/plugin-protocol)

The health service is the standard `grpc.health.v1.Health` service from gRPC,
registered by the reference server above. The provider definitions and the
three supporting go-plugin definitions have different packages and roles;
generating the former does not generate the latter.
