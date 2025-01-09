# wg_watchdog

Bash script that monitors a Wireguard interface for staleness and restarts it
if it seems "frozen", i.e. the Wireguard endpoint can be reached via ping but
the latest successful Wireguard handshake is "old".

This can happen when some routers/firewalls improperly handle NAT rules and
routing tables when a gateway changes (e.g. Internet connection issues) and
start dropping Wireguard packets. This situation cannot resolve itself as Wireguard
will keep the bogus NAT rule or routing rule alive by sending handshake packets
every 5 seconds. The only solution is to stop the stream of packets and wait for the
rule to expire on the offending middleware (often after 30s or 60s),
or simply restart the interface so it picks a different source UDP port.

**NOTE**: This approach will not work if your Wireguard interface is configured
with a static source UDP port using the `ListenPort` field. You would have
to adapt this script to stop the interface, wait for e.g. 60s then start it.

The script detects this situation by checking if the latest handshake is "stale"
and attempting to ping the peer's IP address. If the ping is successful but
the Wireguard handshake is stale, the Wireguard packets are most likely dropped
on the way.

If `FORCE_RESTART` is set to 1, the script does not check the Wireguard interface for
staleness and simply restarts it if it has not been restarted in more than
`RESTART_INTERVAL_MINUTES` minutes. This is useful for cases where the Wireguard
endpoint cannot be pinged (e.g. due to firewall), to ensure a "frozen" interface
will eventually recover.

The Wireguard interface is restarted using systemd and `wg-quick`, i.e.
`systemd restart wg-quick@$WG_INTERFACE_NAME`.

## Installation

```bash
  $ sudo mv wg_watchdog /usr/local/sbin/
```

The script is designed to be invoked periodically, e.g. using cron or a similar
system.

## Usage

```bash
$ wg_watchdog <interface>
```

### Environment variables

- `HANDSHAKE_THRESHOLD_MINUTES` (default 15): Number of minutes after which the Wireguard interface is considered stale, based on its latest handshake timestamp.
- `PING_TIMEOUT_SECONDS` (default 2): Timeout option for the ping attempts.
- `PING_ATTEMPTS` (default 1): Number of ping attempts.
- `FORCE_RESTART` (default 0): If set to 1, the Wireguard interface is restarted (via systemd) regardless of its staleness.
- `RESTART_INTERVAL_MINUTES` (default 30): The number of minutes to wait until next restart.
