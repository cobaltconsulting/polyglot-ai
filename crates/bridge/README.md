# Polyglot Bridge

A WebSocket bridge for the [historical Expo mobile client](https://github.com/tugcantopaloglu/polyglot-ai-mobile). It forwards requests to a Polyglot-AI server over QUIC, or invokes `polyglot-local` on the bridge host.

**Status: part of the discontinued [Polyglot-AI project](../../README.md).** These instructions document retained source; they do not establish a currently supported deployment or tested mobile integration.

## Usage

```bash
cargo build --locked -p polyglot-bridge --release

# WS on 8787, QUIC to server on 4433
./target/release/polyglot-bridge --listen 0.0.0.0:8787 --server 127.0.0.1:4433
```

Run these commands from the repository root. On Windows, use `target\release\polyglot-bridge.exe`. The release workflow packages the local, client, and server binaries, but not the bridge.

Server mode requires a running `polyglot-server` and matching mTLS credentials. The default files are `./certs/client.crt`, `./certs/client.key`, and `./certs/ca.crt`. Use `--cert`, `--key`, and `--ca` for other locations. See the root README's [client-server setup](../../README.md#client-server-mode) and [certificate setup](../../README.md#certificate-setup).

## Local mode

```bash
./target/release/polyglot-bridge --listen 0.0.0.0:8787 --mode local --local-bin polyglot-local
```

Build `polyglot-local` with `cargo build --locked -p polyglot-local --release`, then place it on `PATH` or pass its full path to `--local-bin`. Its AI tools and provider credentials belong on the bridge host. Local mode does not require QUIC certificates or a separate Polyglot server. It rejects file synchronization and file retrieval messages, so the mobile Files screen requires server mode.

### Optional auth token

```bash
./target/release/polyglot-bridge --token '<shared-token>'
```

Mobile clients connect to `ws://<host>:8787/ws`. Enter the token in the connection profile's token field; the client adds the `?token=...` query. Use the host's address reachable from the phone, rather than the phone's own `localhost`.

### Transport and credentials

The listener uses plain `ws://` by default. A token restricts access but does not encrypt prompts or credentials. Use a trusted private network for plain WebSocket development, or enable `--tls` and configure a certificate trusted by the device for `wss://`. `--tls-cert` and `--tls-key` set the certificate paths. Auto-generated certificates are self-signed and are not automatically trusted by phones.

The mobile profile's certificate fingerprint is sent in the Polyglot `Auth` message; it is not WebSocket TLS certificate pinning. In server mode it must correspond to the server's configured user identity. The bridge itself holds the client certificate and private key for the QUIC hop.

## Notes

- The bridge forwards Polyglot `ClientMessage` and `ServerMessage` frames.
- JSON payloads are accepted as text frames, MessagePack as binary frames.
- The mobile app also sends JSON control messages for bridge status, QR payloads, and optional Drive sync. Drive sync requires an independently configured `rclone` remote on the bridge host; it is not configured by the mobile app.
- Use JSON for the documented mobile setup. Compatibility of the mobile MessagePack decoder with the bridge's Rust serialization has not been validated.
