# Recomp Achievements: list format and channel (version 1)

## The list file

One JSON per recomp. RecompLauncher hosts it, so a fixed rule reaches players without a recomp
update. You don't have to write it alone: send us your recomp and we'll work on it with you.

```json
{
  "format": 1,
  "game": "supermarioworldrecomp",
  "source": "memory",
  "console": "snes",
  "copies": ["sha1:6b47bb75d16514b6a476aa0c73a683a2a4c18765"],
  "versions": ">=1.4.0",
  "voids": ["Cheats/enabled"],
  "achievements": [
    {"id": 1, "title": "Feather Light", "description": "Get a cape", "points": 5, "type": "progression", "trigger": "0xH0019=2"},
    {"id": 2, "title": "Castle Crasher", "description": "Beat the final boss", "points": 50, "type": "win", "event": "beat_bowser"}
  ]
}
```

| Field | Meaning |
| --- | --- |
| `source` | `memory` (rules on console memory), `native` (the game's own achievements, Xbox 360), `events` (the port's code fires them) |
| `console` | `n64`, `snes`, `gb`, `gba`, `psx`, `ps2`, `xbox`, `x360` |
| `copies` | The game copies the rules were checked on, as Easy Setup knows them (`sha1:` or a disc serial). Checked at setup only, not at every start |
| `versions` | Recomp versions the rules were checked on: `>=1.4.0`, `1.4.x`, `1.4.0, 1.4.2` or `>=1.2 <2.0`. On any other version the launcher sends no list that session and the Achievements tab says why |
| `voids` | Boot profile setting keys that make unlocks Casual while on |
| `points` | 5, 10, 25 or 50. Native Xbox 360 gamerscore is converted to this scale |
| `type` | `progression`, `win`, `missable`, or left out |
| `trigger` | An rcheevos condition string (memory lists) |
| `event` | The event name the port fires (code events) |
| `native` | The console's own achievement id (native lists, when titles are overridden) |

Native lists only need `format`, `game`, `source` and `console`. RecompLauncher reads the names,
descriptions and gamerscore from the player's own copy of the game, and leaves out achievements that
need online play so Mastered can be reached offline. A runtime that already reads them can send them
with `native_list` too.

Every achievement shows the same star icon in version 1. Unlocking every achievement in
a game makes it Mastered.

## The channel

RecompLauncher starts the recomp with `RL_SESSION` set to a pipe name
(`\\.\pipe\RecompLauncher-<session>` on Windows, a socket path on Linux and macOS).
Both sides send one JSON object per line. On Windows the pipe is in byte mode and the launcher keeps
a spare instance ready, so a recomp that reconnects gets in; the library waits briefly when the pipe
is busy. A `game` in `hello` that doesn't match the launcher's id is only noted in its dev log.

From the recomp:

| `t` | Fields | When |
| --- | --- | --- |
| `hello` | `sdk`, `console`, `memory`, `game`?, `version`? | Once, on connect |
| `native_list` | `achievements` [{`id`, `title`, `description`, `gamerscore`}] | Right after `hello`, from runtimes that know the game's native achievements (`rl_ach_native_define`) |
| `list_ok` | `rules` | After loading a list |
| `unlock` | `id`, `frame` | A memory rule fired |
| `progress` | `id`, `value`, `target` | A measured rule moved |
| `native` | `id`, `frame`?, `earlier`? | The game unlocked a native achievement (`earlier`: it had it before connecting) |
| `event` | `name`, `frame` | The port fired a code event |
| `save_loaded` | `frame` | A save was loaded |
| `casual` | `reason` | Unlocks from now can't be Ranked |
| `broken` | `id`, `why`? | A rule didn't parse or reads memory the console doesn't have |
| `mem` | `seq`, `hex` | Answer to `peek` |

From the launcher:

| `t` | Fields | Meaning |
| --- | --- | --- |
| `list` | `achievements`, `unlocked`? | The memory rules to check, minus the ones already unlocked |
| `quiet` | | The launcher shows the toast; the runtime hides its own. Always sent while achievements are on |
| `peek` | `address`, `size` (max 256), `seq` | Dev tools: read memory for the memory viewer |

The recomp never talks to accounts or servers. The launcher does the toast, the local
store, syncing and the Ranked checks.

## Ranked rules

An unlock counts on leaderboards only when it came from a live session (inside the session's start
and end, and within 15 minutes after the start or a check-in), on the player's
own save, not in a burst right after `save_loaded`, with no `voids` setting or `casual`
message in that session, and in an order and pace that make sense. Everything else is
kept and shown as Casual.

Memory rules can't fire from a loaded save: rcheevos keeps each rule waiting until it
has been false for at least one frame.
