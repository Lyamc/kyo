# kyo

Small web controller for an Onkyo or Integra AV receiver. It speaks eISCP: one TCP session to port 60128, and a page on port 8095 for power, volume, input, listening mode, tuner, net playback, and zone 2.

The page is LAN-only. A browser on a public address gets 403.

## Discovery

kyo does not need a fixed receiver address. It finds the receiver the same way the Onkyo Controller app does: a UDP broadcast to `255.255.255.255:60128` carrying `!xECNQSTN` and `!pECNQSTN`.

The receiver answers that broadcast only when the query is sent **from UDP port 60128**. kyo binds that port for discovery. An ephemeral source port gets no answer.

Behavior:

- With no saved address, kyo broadcasts immediately and connects to the first receiver that answers.
- It keeps that TCP connection open and probes it with `PWRQSTN` every 30 seconds.
- After the connection fails three times, it broadcasts again.
- Further searches wait five minutes, unless you press **Refresh** on the page (or `POST /api/refresh`).
- The last address is written to `$STATE_DIRECTORY/receiver` when that variable is set (systemd `StateDirectory=kyo` does this).

`KYO_RECEIVER` is an optional hint. Leave it empty to discover.

## Run

```sh
cargo run --release
```

Then open `http://<this-host>:8095/`.

| Variable | Default | Meaning |
| --- | --- | --- |
| `KYO_LISTEN` | `0.0.0.0:8095` | Web UI address |
| `KYO_RECEIVER` | unset | Last-known host. Empty discovers |
| `KYO_RECEIVER_PORT` | `60128` | eISCP TCP port |
| `STATE_DIRECTORY` | unset | Directory for the saved address |

## HTTP

| Method | Path | |
| --- | --- | --- |
| GET | `/` | The controller page |
| GET | `/api/status` | Current status, JSON |
| GET | `/ws` | Same JSON pushed as it changes. Send `{"cmd":"PWR01"}` or `{"cmd":"refresh"}` |
| POST | `/api/cmd` | `{"cmd":"MVLUP"}`. 202 when queued |
| POST | `/api/refresh` | Drop the session and broadcast again |
| GET | `/healthz` | `ok` |

Commands are a short allow-list (`PWR`, `MVL`, `AMT`, `SLI`, `LMD`, `ZPW`, `ZVL`, `ZMT`, `SLZ`, `TUN`, `PRS`, `NTC`) plus any `XXXQSTN` status query. Anything else is rejected.

## NixOS

```nix
imports = [ /path/to/kyo/nix/module.nix ];

services.kyo = {
  enable = true;
  openFirewall = true;
  firewallInterface = "eth0"; # your LAN interface
};
```

`nix/package.nix` builds the binary with `buildRustPackage`. The module runs it as a dynamic user, remembers the receiver under `/var/lib/kyo`, and limits the process to private and link-local addresses plus the discovery broadcast.

## License

MIT. See [LICENSE](LICENSE).
