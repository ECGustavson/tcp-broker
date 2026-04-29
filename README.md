# Scale Broker

TCP bridge between Rice Lake 1280 indicators and Ignition SCADA via OPC UA. The 1280 connects outbound as a TCP client; the broker listens, parses the CSV record, writes to OPC UA tags, and waits for Ignition to send an ACK back through a writable tag.

Supports two 1280 user programs — finish scale (manual print key) and rail scale (automatic peak detection). Designed for dual-instance redundancy where both brokers run simultaneously and each 1280 connects to both.

---

## Requirements

- Python 3.12+
- `asyncua`
- `cryptography` (only needed if using OPC UA security)

```bash
pip install asyncua cryptography
```

---

## Setup

Edit `scales.json` with your scale names, IPs, ports, and program types, then run:

```bash
python broker.py
```

Run inside `tmux` if you want the console to survive SSH disconnects:

```bash
tmux new -s broker
python broker.py
# Ctrl-B D to detach
# tmux attach -t broker to come back
```

---

## scales.json

```json
{
  "scales": [
    {
      "name": "FinishScale_01",
      "program": "finish",
      "port": 9001,
      "ip": "192.168.1.101",
      "enabled": true,
      "description": "Optional label",
      "ack_timeout_seconds": 5
    }
  ],
  "opc": {
    "endpoint": "opc.tcp://0.0.0.0:4840/broker",
    "namespace": "http://floweigh/scalebroker",
    "pki_dir": "pki",
    "allow_no_security": true
  },
  "logging": {
    "log_dir": "logs",
    "max_bytes": 5242880,
    "backup_count": 10
  }
}
```

**`program`** — `"finish"` or `"rail"`, case-insensitive.

**`ip`** — set to `null` or omit to accept connections from any IP on that port. Useful if multiple 1280s share one port.

**`ack_timeout_seconds`** — how long the broker holds the connection open waiting for Ignition to write the ACK. Default 5.

**`allow_no_security`** — set `true` to run without OPC UA certificates. The recommended approach for most deployments is to rely on VLANs and port lockdown at the network level rather than OPC UA cert-based security. If you do want certs, run `gen_cert.py` first and set this to `false`.

---

## OPC UA tags

Each scale gets its own folder named from the `name` field in config. Tags within each folder:

| Tag | Type | Notes |
|---|---|---|
| `GrossWeight` | String | |
| `TareWeight` | String | |
| `NetWeight` | String | |
| `H1Gross` | String | Rail only — first half peak gross |
| `H1Net` | String | Rail only — first half peak net |
| `H2Gross` | String | Rail only — second half. Empty if skip side pressed. |
| `H2Net` | String | Rail only — second half. Empty if skip side pressed. |
| `Serial` | String | |
| `ScaleID` | String | Rail: 4=dynamic, 9=static |
| `ScaleName` | String | Rail: always RAIL. Finish: operator-configured. |
| `KillID` | String | Rail: sequential head count, resets midnight. Finish: always 0. |
| `LotCode` | String | |
| `WeightUnits` | String | e.g. LB, KG |
| `Temperature` | String | Finish only |
| `TempUnits` | String | Finish only |
| `PrinterNumber` | String | Finish only |
| `OrderNumber` | String | Finish only |
| `Timestamp` | String | YYYYMMDDHHMMSS |
| `TransactionID` | String | 18-char deduplication key |
| `Message` | String | Full raw CSV — Ignition watches this tag for new records |
| `Writable` | String | Ignition writes the ACK string here |
| `HandshakeAgain` | Boolean | Write True to resend the last ACK without re-inserting to DB |
| `RawRecord` | String | Same as Message — kept for diagnostics |
| `Connected` | Boolean | |
| `RecordCount` | Int32 | Resets on broker restart |
| `SecondsSinceLastRecord` | Int32 | -1 until first record received |
| `LastError` | String | Last parse failure. Clears on next good record. |

---

## ACK handshake

After writing the OPC tags, the broker holds the TCP connection open and polls `Writable` every 100ms for up to `ack_timeout_seconds`. Ignition's tag change script fires on `Message`, inserts to the database, then writes the ACK string to `Writable`.

ACK format: `F#1=OK{TransactionID}{HHmmDDMMyy}` + CRLF

The `F#1=` prefix triggers the 1280's `Cmd1Handler`. The remaining 30 characters (`OK` + 18-char TransactionID + 10-char timestamp) are what the 1280 validates. If the length check passes, the 1280 also syncs its RTC to the returned timestamp.

**HandshakeAgain** — if the 1280 didn't receive the ACK and reconnects to retry, write `True` to `HandshakeAgain`. The broker will resend the cached ACK from the previous transaction without Ignition needing to re-run the DB insert.

---

## Console

The `>` prompt appears on startup.

| Command | Description |
|---|---|
| `status` | All scales — connected, record count, last record age |
| `status <name>` | Detail including last TransactionID and last ACK sent |
| `list` | All scales from config |
| `enable <name>` | Enable in config |
| `disable <name>` | Disable in config |
| `rename <old> <new>` | Rename in config |
| `reload` | Reload config + restart TCP listeners |
| `loglevel <name> <LEVEL>` | DEBUG / INFO / WARNING / ERROR |
| `tail <name> [n]` | Last n log lines (default 20) |
| `ack <name>` | Manually ACK last record on a scale — for testing without Ignition |
| `ack last` | Same, but targets whichever scale most recently received a record |
| `certstatus` | Cert expiry, SANs, trusted store contents |
| `quit` | Shut down |

`enable`, `disable`, `rename` write to `scales.json` immediately but require `reload` to take effect. Renaming a scale requires a full broker restart to update the OPC UA folder name.

---

## Message formats

Both programs use the same 20-field comma-delimited layout, CRLF terminated.

**Finish** (fields 8–11 always empty, fields 13–16 populated):
```
Serial,,ScaleName,0,Lot,Gross,Tare,Net,,,,,Units,Temp,TempUnits,Printer,Order,YYYYMMDD,HHMMSS,TransactionID
```

**Rail** (fields 8–11 populated, fields 13–16 always empty — note four consecutive commas):
```
Serial,ScaleID,RAIL,KillID,Lot,Gross,Tare,Net,H1Gross,H1Net,H2Gross,H2Net,Units,0.0,,,,YYYYMMDD,HHMMSS,TransactionID
```

---

## Ignition connection

`Config → OPC UA → Connections → Add Connection`

Point it at `opc.tcp://<broker-ip>:<port>/broker`. For dual-broker redundancy, add a second connection as the failover endpoint — Ignition will switch automatically if the primary goes unreachable.

The Ignition UDT watches `Message` for new records and writes ACKs to `Writable`. Finish scales use a tag called `Writable` on the Ignition side; rail scales use `Handshake` — both map to the broker's `Writable` tag.

---

## Troubleshooting

**Scale not connecting** — check the broker log for a "Rejected connection" line. The source IP doesn't match the `ip` field. Either update the config or set `ip` to `null` to accept any source.

**Parse errors in LastError** — check `RawRecord` for the raw string. Field count must be exactly 20. Rail scale messages have four consecutive commas between Units and Date (`LB,0.0,,,,20260421`).

**ACK timeout warnings** — Ignition didn't write to `Writable` within the timeout window. Check the Ignition tag change script is enabled and the DB connection is healthy. Use `ack <name>` in the console to test the TCP path independently.

**OPC tags show Bad quality** — OPC UA connection is down. If running with `allow_no_security: false`, check `certstatus` and verify the cert exchange with Ignition is complete.

**1280 keeps retrying** — the rail scale retransmits every ~60 seconds until ACKed. Check `status <name>` for the last ACK sent. If needed, use `HandshakeAgain` or `ack <name>` to manually push an ACK.