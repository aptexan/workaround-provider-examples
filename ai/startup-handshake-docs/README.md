# OpenTofu provider startup and handshake

These documents describe how an external provider executable becomes a gRPC
server that OpenTofu can use. They are intended for SDK authors and authors of
providers in languages such as C++, where the Go SDK does not supply the process
and transport machinery.

- [Startup and handshake protocol](startup-handshake.md): process launch,
  environment, version selection, stdout framing, mutual TLS, gRPC services,
  and shutdown.
- [Implementing and testing a C++ provider](cpp-provider.md): implementation
  sequence, protobuf inputs, development overrides, debugger attachment, and
  troubleshooting.
- [Source map](sources.md): the exact revisions and implementation locations
  used to establish the behavior described here.

The scope is external **provider** plugins using provider protocol major 5 or 6.
Provider installation, provisioners, historical net/rpc protocols, and the
semantics of resource planning and state encoding are outside this document.

The startup protocol is supplied by `github.com/hashicorp/go-plugin`. The
provider RPC protocol is supplied by OpenTofu's `tfplugin5` or `tfplugin6`
protobuf definitions. Implementing only the provider service does not reproduce
the complete startup and lifecycle behavior of the Go SDK.

These are implementation-derived notes for the revisions in the source map,
not a newly declared protocol version. In particular, numeric defaults and
failure handling can change with the version of `go-plugin` used by OpenTofu.
Protocol definitions on development branches can also contain unreleased
features; choose a released definition for a released compatibility target.
