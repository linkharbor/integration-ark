# LinkHarbor for ARK

A single-file Python agent that polls an ARK: Survival Evolved or ARK:
Survival Ascended server over RCON and pushes the live state to your
[LinkHarbor](https://linkharbor.com) bio page: a game-server card with
Online/Offline, the number of players and their names, plus join/leave
events for push notifications to your followers.

It uses the RCON port every ARK dedicated server can open, so nothing is
installed on the server. Standard library only, Python 3.10 or newer. ARK's RCON does
not report the player limit; pass it with `--players-max` if you want
"12 / 70" on the card.

Part of the [LinkHarbor integrations](https://github.com/linkharbor/integrations)
family, one repo per game or tool.

## Setup

```bash
git clone https://github.com/linkharbor/integration-ark
cd integration-ark

export LH_INGEST_URL="https://dock.at-harbor.com/v1/ingest/123"   # dashboard: Integrations, Custom (API)
export LH_INGEST_SECRET="…"
export ARK_RCON_PASSWORD="…"                                       # see "Enabling RCON"

python3 agent.py --host 203.0.113.10 --port 27020 --name "The Island PvE" \
  --players-max 70 --address "203.0.113.10:7777" \
  --url "https://discord.gg/xyz" --interval 60
```

Start with `--once -v`. Run it as a service with `integration-ark.service`
(systemd; secrets go into `/etc/integration-ark.env`, `chmod 600`). If the
server runs on the same machine, `--host 127.0.0.1` works even when the RCON
port is firewalled from the outside.

## Events sent

| Event | When | Template variables |
|---|---|---|
| `server.online` | the first successful poll, or the server is back after being unreachable | `{name}`, `{players_online}`, `{players_max}` |
| `server.offline` | the server stopped answering, or rejected the password | `{name}` |
| `player.joined` | a new name appears in `ListPlayers` | `{player}` and the above |
| `player.left` | a name disappears | `{player}` and the above |

The state is sent with every event and every `--heartbeat` minutes (default
10). `--no-player-names` sends counts only. When ARK answers `ListPlayers`
with something other than a player list, the agent logs it and keeps the
previous list for that poll.

Suggested push rule (dashboard, Integrations, Push rules, Custom (API), event `player.joined`): batch window 10 min, title "🎮 {count} players joined", body
"{player_list} are now on {name}", minimum pause 5 min.

## Options

```
--host HOST            server IP or hostname (required)
--port PORT            RCON port (default 27020)
--password PASSWORD    RCON password = ServerAdminPassword (or env ARK_RCON_PASSWORD)
--name NAME            card title (default "ARK server")
--players-max N        player limit shown on the card (RCON does not report it)
--address ADDRESS      address shown on the card, e.g. 203.0.113.10:7777
--url URL              link behind the card's button
--cta TEXT             button label (default "Join server")
--game TEXT            game label (default "ARK")
--image-url URL        banner image (large layout)
--interval SECONDS     polling interval (default 60, minimum 5)
--heartbeat MINUTES    state-only update interval (default 10, 0 = only on change)
--no-player-names      send counts only
--once                 poll once, send, exit
-v                     verbose logging
```

## Enabling RCON

In the server's `GameUserSettings.ini` (under `[ServerSettings]`):

```ini
RCONEnabled=True
RCONPort=27020
ServerAdminPassword=a-long-random-string
```

The RCON password is the `ServerAdminPassword`; put it into
`ARK_RCON_PASSWORD`. Restart the server after editing the file. The agent
opens a new connection every poll and runs `ListPlayers` only; nothing is
changed on the server. RCON is plain TCP without encryption, so keep the port
on a private network or behind a firewall.

## License

[MIT](LICENSE). © 2026 EMI Solutions OÜ.
