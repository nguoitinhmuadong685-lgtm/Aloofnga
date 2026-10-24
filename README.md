# Nebula Proxy - Gophertunnel rewrite

A Go rewrite of the Node/`bedrock-protocol` proxy, on
[gophertunnel](https://github.com/sandertv/gophertunnel) v1.59.0.

## Why

The JS proxy depends on PrismarineJS `minecraft-data`, which ships schemas up to
**1.26.30 (protocol 1001)**. Live servers are on **2168** (1.26.40 through
1.26.44, one protocol number, and from 1.26.44 not one format), so that proxy
runs on a hand-written delta schema, a hypothesis, not a definition, and every
module reading a mis-parsed packet field silently no-ops.

Gophertunnel targets `CurrentProtocol = 2168`, `CurrentVersion = "1.26.44"` natively.
The schema problem does not exist here.

Secondary wins: one static binary instead of a node process per tester, and
packets are **typed structs** rather than name-keyed maps. A mistyped packet
name is a compile error instead of a branch that never fires.

## Layout

```
cmd/nebula-proxy/      the product: entrypoint and module registration
cmd/nebula-bots/       the bots runner: one process per account, many alts
cmd/nebula-gateway/    public front door: resolves the player's instance, then transfers
cmd/nebula-relay/      node-local UDP relay replacing the gateway's Transfer
cmd/pingtool/          diagnostic: ping a server, probe which protocols it accepts
internal/module/       Module interface, Base, Setting, Manager (dispatch)
internal/state/        Game state: player, entity tracker (mutex-guarded)
internal/proxy/        the MITM relay + packet-driven state tracking
internal/modules/      the modules themselves, one package per category
internal/mv/           multi-version support (see below)
internal/xauth/        the proxy's own Xbox login (upstream leg)
internal/config/       config.json load/save, named presets, autosaving
internal/dashboard/    web UI: embedded bundle, REST API, WebSocket push
internal/licence/      startup licence gate (HWID-bound, fails closed)
internal/hudpack/      branded client resource pack served at join
internal/music/        the music folder -> a second resource pack + the library
internal/mcutil/       shared packet/input helpers (text, input flags, block actions)
internal/vendorcheck/  test-only guard over the vendored gophertunnel patches
bots/                  the alt accounts: registry, session, their own modules
scripts/               protodef diffing + registry generation (Node)
```

## Dashboard

`http://127.0.0.1:8080` by default; `-dashboard ""` turns it off.

The client is the **prebuilt React bundle from the JS proxy**, copied into
`internal/dashboard/public` and embedded in the binary with `go:embed`. It is
not rebuilt here; the TypeScript sources live in the other proxy, so the JSON
shapes in `snapshot.go` and the routes in `server.go` are a **fixed contract**.
Renaming a field there shows up as a blank panel in the browser, never as a
compile error, which is why those shapes are explicit structs rather than maps.

Two details the JS server got wrong and this one does not:

- **Enum settings are reported and accepted as labels.** The bundle's `<select>`
  is keyed on the option label, not its index. Storing the raw string made every
  module comparing against an index fall through to its default branch, so
  changing a mode from the dashboard quietly did nothing.
- **Toggles go through `Manager.Toggle`, never `SetEnabled`.** A module whose
  whole effect is emitted when it is enabled needs those packets queued for the
  relay; `SetEnabled` only returns them.

Loopback-bound by default: the API can toggle every module and
rewrite the config, and it has no authentication.

## Music (`internal/music`)

Drop audio files in `music/` (next to the binary; `-music-dir` moves it) and
the proxy serves them as a second resource pack, "Nebula Music", after the HUD
pack. Anything that is not already `.ogg`/`.fsb` is converted once by ffmpeg
into `music/.cache/`; durations come from ffprobe and are remembered in
`.cache/index.json`.

The **Music** module plays them: `/.music play <n|name>`, `stop`, `next`,
`prev`, `pause`, `resume`, `list`, `rescan`, `now`. Settings — Volume,
Shuffle, Loop (Off/One/All), Autoplay — persist like every other module's and
show on the dashboard. Playback is `PlaySound`/`StopSound` against the pack's
`sound_definitions.json`; pause is stop-and-resume-at-offset, which is the
only seek the client honours. The pack version is a hash of the track set, so
a client re-downloads it only when the songs change; an empty folder serves
nothing at all.

## eBag (`bots/runner/ebag.go`)

One alt account is the player's **mobile storage**. Not a database of items: a
real account, signed in, standing in the world, and its **real Minecraft
inventory is the bag**. What it is carrying is what you have, and getting
something out of it means asking it to come over and drop it.

The owner and the storage are never the same session. The owner plays through
the proxy; the eBag is a bot in the runner's registry like any other, and the
eBag is the label saying which one plays that part. `ebag.json` in the runner's
data directory remembers it across restarts, beside `bots.json` and `ops.json`.

### One service, three doors

`Registry.EBag()` is the whole implementation. The three places you can reach
it from differ **only in how the caller is authenticated**:

| Door | Reaches it through | Who it proves you are |
|---|---|---|
| `/.ebag` in game | `misc.EBag` → bot app `/api/proxy/ebag` | licence key + HWID |
| `/ebag` on Discord | `cogs/ebag.py` → bot app `/internal/ebag` | the shared internal token, then the portal account linked to that Discord id |
| the dashboard tab | `views/ebag` → bot app `/api/r/ebag*` | the dashboard session cookie |

`bot_app._EBAG_OPS` is the one table mapping an op name onto the runner's
route, and `test_every_door_maps_onto_the_same_runner_routes` holds all three
to it. A rule written in a door instead of in the service is a rule the other
two do not have, which is how the JS proxy ended up with three different ideas
of what "drop" meant.

### States

`offline`, `connecting`, `reconnecting` and `idle` are read off the session, so
a bag that dropped out says so even if the queue thinks it is busy. `moving`,
`bursting`, `following` and `dropping` are held by the worker while it is
steering. `unassigned` is nobody having picked an account yet, and it is a
state rather than an error because it is the normal first screen.

### The queue

One worker goroutine, a buffered channel, and a **steering lock**. Every door
enqueues and returns: a command handler runs on the relay's own goroutine and
an HTTP handler on the listener's, and neither may wait on a trip across a
world. Each job carries a deadline and one retry, and a retry is only taken
for a failure worth repeating — a trip the server put a bot back from, a drop
the world never showed. A refusal is not retried.

**Follow does not queue.** It is a mode with a ticker that re-aims while
nothing else is steering (`TryLock`, so a queued job always wins). Queueing a
move every two seconds would fill the queue the moment the owner walked out of
range.

### Nothing is assumed to have worked

- A **trip** is driven by the existing `SMTP` module — the same glide, and the
  same `Burst` setting the player has on `/.smtp` — and then the distance is
  checked. Arriving is a measurement, not the absence of an error.
- A **drop** is not acknowledged the way an item stack request is, so it is
  confirmed two ways: the server sending the slot back changed, or the stack
  appearing as an item entity beside the bag. Neither arriving is reported as
  *unconfirmed*, not as success. An eBag that says it handed something over and
  did not is worse than one that admits it is unsure.
- **Emptying the bag** is 36 of those in a row, so it stops at the job's
  deadline and says how many slots are still full.

### Destructive actions are confirmed

`drop everything` is the only one, and the service refuses it with a pending
token rather than doing it. In game that token is answered by the `/.yes` every
other module already uses; on Discord by a button only the person who asked can
press; on the dashboard by a dialog. The offer expires on its own.

## Multi-version (`internal/mv`)

**1.21.90 through 1.26.42, all thirteen versions are ready.** 1.26.44 has no
entry of its own, because it is gophertunnel's native version, but it is why
two of those thirteen look the way they do: the August 2026 hotfix kept protocol
2168 and changed the format underneath it, so 2168 no longer names one format.
See "1.26.44 kept the number and changed the format" below. The design rests on
two facts.

**Both ends speak one version.** The proxy dials upstream with the same protocol
the client connected on, so a packet read at version V is written back out at
version V. There is never anything to translate, which is why this is hundreds
of lines and not the tens of thousands the general-purpose multi-version
libraries need.

**Unknown packets relay perfectly.** Gophertunnel returns `packet.Unknown` for
any ID absent from a protocol's pool and re-serialises it byte for byte. So a
packet whose layout changed is *excluded* rather than mis-parsed. It still
reaches the other side intact, it is simply invisible to modules.

That leaves only two categories needing real work per version, and the sizes are
measured rather than guessed (`scripts/protodiff.js`, against minecraft-data's
protodef, resolving types transitively so a change in a shared type is caught):

| | 1.21.90 | 1.26.0 | 1.26.20 |
|---|---|---|---|
| handshake packets needing a shim | 4 | 3 | 1 |
| module packets needing a shim | 20 | 16 | 3 |

The handshake column is what gates a version: without `StartGame`, `Disconnect`,
`DimensionData` and `ResourcePackStack`, the connection cannot be established.
The module column is purely additive; each shim buys back one module's
behaviour on that version.

**No older version is advertised until its handshake shims exist.** `Version.Ready`
gates `AcceptedProtocols`, and a test asserts no ready version excludes a packet
gophertunnel's own `Conn` decodes. A half-finished shim that accepts a
connection and then mis-parses `StartGame` strands the player in an empty world
with no error anywhere, strictly worse than declining the version and saying
why.

### What the shims actually are

Three packets, not four. `Disconnect` needs none: the generated diff flags it at
every version, but only `DisconnectFailReason`'s **label map** changed; the base
type is `zigzag32` throughout and gophertunnel reads a plain `int32`, so the
bytes never moved. It is un-excluded rather than shimmed.

`StartGame` is one parameterised type (`startgame.go`) rather than eleven copies
of an eighty-field `Marshal`, because the differences are six present/absent
flags:

| flag | versions |
|---|---|
| editor fields (policy + anonymous drops) | 1.26.30 only |
| `is_chat_logging` | 1.26.30 only |
| server-join-information | none on 1.21.x, short form on 1.26.0, full from 1.26.10 |
| `tick_death_systems` | 1.21.100 - 1.21.124 |
| identifiers early, and in a different order | 1.21.x |
| unsigned spawn `y` | through 1.26.0 |
| signed game-rule ints | through 1.21.100 |

**The 1.26.30 → 1.26.40 step is inferred, not measured.** minecraft-data stops at
1001 and Mojang renumbered straight to 2168, so nothing describes the gap. It was
derived by comparing 1.26.30's protodef against gophertunnel's structs field by
field, which turned up four differences: `is_chat_logging`, the game-rule integer
width, `GatheringJoinInfo`'s Optionals, and `DimensionDefinition.PackID`. That is
why **1.26.30 needs a shim too** rather than being "the same as current", and it
is the weakest link here. It fails loudly at the handshake if wrong.

Round-trip tests prove each shim is self-consistent: every field written and read
at the same offset. They cannot prove the layout matches a real client of that
version; only connecting one does.

### 1.26.44 kept the number and changed the format

Every version above is identified by its protocol number, and that number is
what selects the layouts. That is exactly what the 14 August 2026 hotfix broke:
it shipped on **2168, the same number as 1.26.40 through 1.26.43**, and changed the wire format underneath.
the wire format underneath. Gophertunnel implements it natively from
v1.59.0 and tells the two apart by the client's **GameVersion string**.

So 2168 selects between two formats, and the registry holds **two entries for
it**: gophertunnel's native 1.26.44, and a `1.26.40` entry here for the older
builds. One byte separates them. A `SetScore` remove entry carries its objective
name in an optional; 1.26.44 wraps that in a second, always-present one, so two
presence bytes where every earlier version writes one.

That is not an obscure packet. A remove entry is how a server clears a
scoreboard line, so any server with a live scoreboard emits them continuously,
and the first batch arrives while the player is still on the loading screen.
Reading the new form with the old code takes the outer byte for the inner one
and frames every following field one byte early, until the entry count runs past
the end of the payload.

**This file previously said 1.26.44 needed nothing here**, because the wiki put
the whole 26.4x line on 2168 and a shared number was taken to mean a shared
format. The number was right and the inference was wrong. What it cost was
every 1.26.44 player being served scoreboards in 1.26.40's encoding and dropped
with "An error has occurred" mid-join.

The fix is in two halves and only one of them can live in this repository:
`internal/mv/scoreboard.go` supplies the old encoding to every version below
1.26.44, and **patch 2 in `vendor/PATCHES.md`** makes the listener skip a
same-numbered entry when choosing by number alone, so the shimmed 2168 only ever
serves a client whose version string claims it and the native one keeps the rest.

### 1.26.42 is newer than gophertunnel, and that matters more than the old versions

Some clients report **1.26.42 = protocol 2169**; gophertunnel v1.59.0 ships
structs for **2168 / 1.26.44**. So `2169` is registered too, reusing 2168's
structs; otherwise the proxy refuses a client people actually run. It sits
*below* 1.26.44 rather than above it, so it takes the `SetScore` shim like every
other pre-1.26.44 entry: an empty exclusion set stopped meaning "identical to
the current version" the moment the current version became one 1.26.42 cannot
parse.

(That 2169 came from a pong, not a published table. minecraft.wiki puts the
whole 26.4x line (26.41, 26.42 and **26.44**) on 2168, so a retail 1.26.44
build is served by the native path and never reaches this entry. It stays for
whatever build does send 2169; an unused entry costs one struct.)

It also carries `DialAs: 2168`, and that turns a limitation into a feature.
Lifeboat answers a 2169 handshake with:

```
We haven't updated to this version yet. Please try again later.
Protocol 2169 not supported
```

So a 1.26.42 player **cannot join Lifeboat at all**, proxy or no proxy. The
relay announces 2168 upstream while serving the client as 2169, which gets them
in. Nothing is translated; only the announced number differs, which is sound
here because a patch bump does not change the wire format, and unsound in
general, hence a per-version field rather than a rule about nearby numbers.

Current state: **all thirteen registered versions are ready: 2168 on both
formats (1.26.40 - 1.26.43 shimmed here, 1.26.44 native), 1.26.42's 2169, and
the eleven from 1.26.30 down to 1.21.90 with their handshake shims in place.**
Protocols 2170 – 2189 are pre-registered speculatively on top of that; see
`forward.go`.

One gap is not in the data and has to be checked by hand: minecraft-data has no
1.26.40, because Mojang renumbered from 1001 straight to 2168 at that release.
Every exclusion set is therefore measured against **1.26.30**, and the
1.26.30 → 1.26.40 delta is not covered.

It is also **not empty**. Comparing 1.26.30's protodef `StartGame` against
gophertunnel's struct field by field (`scripts/packetfields.js` lists the
fields): 1.26.30 carries `is_chat_logging` between `server_controlled_sound` and
the server-join block, and gophertunnel has no such field. Every field after
that offset would be read at the wrong position, so 1.26.30 needs a shim like
any other version, and the exclusion sets for older versions are a lower bound
rather than the whole story.

### Overrides convert; they do not replace

The obvious shim (a replacement that IS `*packet.StartGame` but reads the old
layout) is impossible, because `Marshal` is a method on the concrete type and
there is only one of those. So an override decodes into its **own** type and
`Override.ToLatest` / `FromLatest` translate to and from the current-version
type.

That translation is load-bearing rather than tidy-up. Gophertunnel's `Conn`
type-asserts on the concrete types to drive the handshake (`pk.(*packet.StartGame)`
to build `GameData`). A version-specific type reaching that switch matches
nothing, so the connection hangs waiting for a packet it has already received
and discarded, which presents as an unresponsive server. A test asserts every
override supplies both directions, because a one-way override fails far from the
missing function.

## Design differences from the JS proxy

These are deliberate; each closes a bug class that actually bit the JS version.

- **Settings are handles, not strings.** Modules capture `*Setting` at construction
  (`k.sRange = b.RegisterSlider("Range", ...)`) instead of calling `getNum("Range")`
  at use time. In JS a typo returned `?? 0`; a `Range` of 0 is a module that does
  nothing, with no error. Here a misspelt name cannot become a live code path.
- **Enums compare by label, not index.** `s.EnumIs("Teleport")`, not `mode === 1`.
  Inserting an option in the middle used to silently change the meaning of every
  saved config and every call site at once.
- **Packets are mutated in place.** Hooks get a pointer, so there is no `params`
  to thread through and no way for one module to drop another's edit by
  forgetting to return it.
- **Forged packets are collected before `Cancelled` is honoured.** The
  cancel-and-replace idiom is used all over the module set; breaking out first
  throws the replacement away.
- **Every hook is panic-contained.** A panic kills the process in Go, so one bad
  module would drop every player's connection.
- **State is mutex-guarded and copied out.** The JS proxy shared plain objects
  safely because it was single-threaded. That assumption is gone.

## Porting status

| | |
|---|---|
| Core framework | done, settings, manager, dispatch, state, relay |
| Subsystems | **all built**, dashboard, licence gate, autosave, multi-version, hudpack |
| Modules | **156 registered** (`scripts/listmodules.js` regenerates the table below from `main.go`) |
| Tests | **1,748 test functions passing** across 20 packages, `go vet` clean |

| Category | Modules |
|---|---|
| Combat (28) | `AimAssist`, `Aimbot`, `AntiBot`, `AntiHit`, `AuraHelper`, `AutoClicker`, `AutoCrystal`, `AutoHVH`, `AutoKill`, `AutoTrap`, `BowSpam`, `Criticals`, `DamageText`, `Hitbox`, `InfAA`, `InfReach`, `KillAura`, `KillAuraH`, `MobAura`, `AboveAura`, `InsideAura`, `InfinityAura`, `Nametags`, `Reach`, `Surround`, `Switcher`, `TPAura`, `TriggerBot` |
| Movement (26) | `AirJump`, `AntiVoid`, `ArrowDodge`, `AutoJump`, `AutoMove`, `AutoSneak`, `AutoSprint`, `CreativeFly`, `ElytraFly`, `Fly`, `FlyLB`, `Freecam`, `Freeze`, `HighJump`, `Jesus`, `Jetpack`, `Jitter`, `MotionFly`, `NoClip`, `NoSlow`, `Phase`, `Speed`, `Spider`, `SpiralMove`, `Step`, `Velocity` |
| World (39) | `AirPlace`, `AutoEZ`, `AutoHub`, `AutoMine`, `BaseBot`, `DeathWaypoints`, `Diam`, `Diam2`, `DiamondFinder`, `ESP`, `FullBright`, `GlyphESP`, `ItemESP`, `ItemTags`, `NoFall`, `NoRender`, `Nuker`, `OreMiner`, `PortalFinder`, `SMTP`, `SMTransfer`, `SMTeleport`, `Scaffold`, `StashFinder`, `StorageESP`, `StorageTracers`, `Top`, `Tracers`, `WartESP`, `XRay`, `ChestTP`, `SpawnerTP`, `WaterTP`, `StashTP`, `TPMine`, `Hub`, `Spawn`, `ItemTeleport`, `MobTP` |
| Player (30) | `AntiCrystal`, `AntiHunger`, `AutoArmor`, `AutoEat`, `AutoFish`, `AutoGaps`, `AutoSwitch`, `AutoTool`, `AutoTotem`, `BlockReach`, `CameraTP`, `ChestStealer`, `Clip`, `DeathTP`, `EnemyTP`, `EnemyTPv2`, `FriendTP`, `Friends`, `NoHurtCam`, `Offhand`, `PacketMine`, `PlayerCoords`, `PlayerCord`, `PlayerInfo`, `PlayerTP`, `PopCounter`, `SpeedMine`, `SurfaceTP`, `TP`, `TargetInfo` |
| Misc (34) | `ACDetector`, `AntiDesync`, `AntiLagBack`, `ArrayList`, `AutoJoin`, `Banner`, `Chat`, `ClickGUI`, `CommandList`, `Commands`, `CoordLeaker`, `DeviceSpoofer`, `DisableWarning`, `Disabler`, `Dupe`, `FastRegen`, `FriendJoin`, `Godmode`, `JoinSplash`, `Loader`, `ModWarning`, `NameProtect`, `NoPacket`, `PacketCounter`, `PacketLogger`, `PlayerCounter`, `PlayerJoin`, `QuickPanel`, `SeqTPspam`, `Spammer`, `Timer`, `Timer2`, `Twerk`, `Visualize` |

### Near-identical modules share an implementation

`MobAura`, `AboveAura` and `InsideAura` differ only in **where** they claim to
be standing, so they are one `positionalAura` plus an aim function each,
rather than three near-copies.

In the Node proxy these were separate files that drifted, and the same bug
targeting by tracker insertion order instead of by distance, so the aura
reliably hit whichever entity spawned first, had to be found and fixed in
each of them independently. `auras_test.go` runs the shared behaviour tests
across all three in a loop, so a regression cannot hide in one variant.

### Behaviour notes for recent features

- **`Friends`**: attacking a friend is cancelled by the shared friend filter,
  and clicking the SAME person in the Friends panel a second time takes them
  back off the list, a toggle, not an add-only button.
- **`DeathWaypoints`**: a death is also *inferred* on servers that never say
  you died, a respawn with a position reset and full-health restore is
  recorded as a death (see `state/death.go`), so the waypoint appears on
  servers whose respawn handshake skips the obvious packets.

### Position spoofing is applied by the relay, not by a module

Bedrock evaluates an interaction from the position the CLIENT reports, so an
out-of-reach attack lands if you claim, for the tick carrying it, to be
standing next to the target. `state.SetSpoof` records the claim; the **relay**
applies it to the outgoing `PlayerAuthInput`, after the module chain.

In the Node proxy this lived in a separate module the reach modules
"required". With that module switched off, every reach module silently did
nothing, the single most-reported symptom of the whole set. Driving it from
session state means a module that asks for a spoofed position gets one, with
no ordering or enabled-ness to get wrong.

Two bounds keep it survivable, and both are in `spoof.go` rather than in each
module:

- **The claim expires.** It lasts the number of inputs it was set for (two:
  one carrying the attack, one covering a server that processes it on the
  following tick). A standing lie is one a server can check.
- **`MaxSpoofDistance` clamps it.** Past the server's movement tolerance the
  claim is not believed and the player is corrected back, a visible
  rubber-band and no hit, which is worse than a shorter claim. Clamping
  approaches a distant target over several ticks instead of failing silently.

The tracked player position is never moved to the spoof: state must keep
describing where the player really is, or every module reading it computes
ranges from a position that exists for one tick.

### Item stack requests are not fire-and-forget

`ChestStealer` moves **one stack at a time and waits for the server's
`ItemStackResponse`** before the next. Each request carries an id the server
answers and a `StackNetworkID` naming the exact stack instance; send one for a
stack the server has already moved, or reuse an id, and it is rejected, after
which the client's inventory view and the server's have diverged and the player
sees items that are not there.

Request ids are negative and decrease by two, which is the convention the
server validates against. Bursting requests is faster and is the version that
desynchronises.

This is why `AutoArmor` was deliberately scoped to hotbar-only when it landed:
the sequencing was worth doing once, properly, rather than twice badly.

### Block names are resolved by hashing, not by a lookup table

Modern Bedrock derives a block's network id by hashing its state, name plus
properties, as canonically serialised little-endian NBT, rather than indexing
a palette. That makes the id **computable from a name**, which is the only way
a proxy can answer "which id is diamond ore": the palette never crosses the
wire.

`StartGame.UseBlockNetworkIDHashes` says which regime the server is in.
`Chunks.RuntimeIDFor` refuses to answer in index mode, because a computed id
there is a number that matches nothing, or matches the wrong block, silently.
`OreMiner.Available()` surfaces that, so "no ores nearby" and "this cannot
work here" are distinguishable.

The hash is **FNV-1a 32-bit**, and the serialisation is hand-rolled rather than
run through the `nbt` package because it must match one exact byte layout:
empty-named root compound, `name` string, `states` compound with keys
**sorted**, two end tags. Different key order or a named root gives a different
hash. `blockhash_test.go` pins the algorithm against the published FNV-1a
32-bit test vectors rather than against its own output.

This also upgrades air: in hash mode it is computed exactly and the
uniform-section voting heuristic becomes a fallback for index-mode servers.

### The ESP family reports first and draws second

A proxy cannot render in the world. `ESP`, `ItemESP` and `StorageESP` expose
`Summary()` / `Nearby()` for the dashboard and
chat to read, and that readout is the primary output: it works through walls and
on a server that filters particles.

`StorageESP` adds a third route on top of those two: `SetSink` attaches an
overlay transport, and it publishes a `SquareFrame` of world-to-screen projected
2D squares, one per container, coloured by type, dark-bordered, scaled by
distance. With no sink attached the projection is never run, so the module costs
nothing but the block-entity scan. That is a large part of why the Node ESP
modules were inert, they existed, emitted packets, and nothing ever drew them.

The boxes and ropes are the second-class half, and every one of them is
particles placed along an edge, because `SpawnParticleEffect` is the only
drawing primitive the protocol offers. They are off by default and capped in
every direction (`Max Boxes`, `Max Tracers`, density, refresh), see
`internal/modules/world/render.go` for what a box actually costs.

**Tracers are the exception:** with **New Tracers** on, the line is drawn as a
guardian beam, one invisible guardian entity per target, so the client's own
renderer draws it, depth-tested and smooth, instead of a particle rope. The
particle path remains for anchors that are not entities (storage tracers on
block entities). See `internal/modules/world/beam.go`.

Subsystems: settings, manager/dispatch, session state, relay, inventory + item
registry, **containers** (armour/offhand/chests), **chunk decoding**,
**block-entity NBT**, world/line-of-sight, player list, input-flag helpers,
config persistence. Nothing is now blocked on missing infrastructure.

**Out of scope by request:** `FastEat`, and the crasher/freezer family.

### Toggle through the Manager, never SetEnabled directly

A module's enable/disable hook returns an `Emit`, packets with the direction
spelled out. `Manager.Toggle` queues it; the relay's tick loop drains it and
sends each side to the right connection.

This started as a bug I reproduced from the JS proxy while claiming to have
fixed it. `Base.SetEnabled` *returned* the packets, and every call site dropped
them, so `Fly`'s disable-restore went nowhere, leaving the client believing it
could still fly. A returned value nobody consumes is the original bug wearing a
type signature. The unit test passed the whole time, because it called
`SetEnabled` and inspected the return: it validated the unit while the
integration was broken.

Direction has to be explicit because there is no context to infer it from. In a
packet hook it is implied, a packet forged while handling server→client goes
to the client. An enable hook has no such position, and an ability restore sent
upstream tells the server about an update it never issued.

`pending_test.go` covers it, including the death-auto-disable path, which is
precisely where a dropped restore is invisible.

### Cross-module state lives on the session, never in a package

`AntiBot` publishes its verdict by installing a predicate on `state.Game`, and
the combat modules read it from there. The first attempt used a package-level
handle and was wrong twice over: **one proxy process serves many sessions**, so
a package variable is shared by all of them and one player's AntiBot would
judge another player's targets. It also made the test suite order-dependent
a leftover instance from one test silently flagged a later test's players, which
is how it was caught.

Anything a module needs to publish to its siblings belongs on the session
state. `isolation_test.go` guards it.

### Priority is behaviour, not decoration

The manager dispatches highest-first, and some modules depend on it.
`Criticals` (70) must run after `KillAura` (100) and `AutoClicker` (95),
because it reacts to the attack transaction they produce, inverted, it would
mark the input before the attack that needs marking exists. There is a test
asserting that ordering, so a later priority change cannot break it silently.

`KillAura` is the reference implementation: target selection, range/wall-range,
mob filtering, interval throttling, multi-target, rotation modes, the bounded
rotation window, and weapon switching against the item registry.

## Chunk decoding

gophertunnel exposes `LevelChunk.RawPayload []byte` and decodes nothing below
it, so the sub-chunk format is implemented here. `df-mc/dragonfly` has the
canonical decoder but arrives attached to a whole Minecraft server, and its
runtime IDs only align if its protocol target matches ours.

Three details that are easy to get wrong, each covered by a test:

- **Packed indices never straddle a 32-bit word.** A word holds
  `floor(32/bits)` and the rest is padding. Treating the words as one flat bit
  stream passes for 1/2/4/8 bits and fails on 3/5/6.
- **Block order is x-major**: `(x<<8)|(z<<4)|y`. Read y-major it yields terrain
  that looks plausible and is rotated, much harder to notice than absent.
- **`yIndex` is signed.** Worlds start at y=-64; losing the sign stacks every
  section at the wrong height.

### Identifying air

Runtime IDs are hashes of block-state NBT in modern Bedrock and the full
palette never crosses the wire, so there is no table to look air up in. Air is
learned instead: a uniform section, palette of exactly one entry, **above
y=64** is air with near-certainty, confirmed across several sections.

The height restriction is load-bearing, and leaving it out was a real bug
caught by test. A player in a cave sees many uniform *solid* sections, and
enough of them outvote a plain count, at which point stone is classified as
air and every wall turns transparent.

Until air is known, nothing is reported solid: before that every block would
look opaque, and a visibility check built on that reports the whole world
blocked, silently disabling every aura.

### Known limits

- **`UpdateBlock` wins over decoded terrain**, deliberately, it arrives after
  the chunk, so it is strictly newer. The other order keeps reporting a wall
  someone already mined through.
- Line of sight still **fails open** on undecoded columns. A false "visible"
  costs one attack the server may reject; a false "hidden" silently disables
  every aura.
- Particles are not depth-tested the way a rendered outline would be, so a box
  behind a wall is drawn and then hidden by the wall. The drawing shows what is
  in line of sight; the readout is what sees through it. (Tracers with
  **New Tracers** on are the exception: they are drawn as client-rendered
  guardian beams, which depth-test properly.)
- **`AutoTotem` shipped once container tracking landed.** It was held back
  deliberately: the offhand is its own window, and a version written before
  that could not check whether a totem was already equipped, so it would have
  either spammed requests forever or done nothing at all. Deferring it beat
  shipping a module that appeared in the list and did neither.

## Config

Reads and writes the same `config.json` shape as the Node proxy, so settings
carry across. Unknown top-level keys are preserved on save: the Node config
holds much this port does not model, and dropping it would make running the Go
proxy once a destructive operation on a file the other proxy still reads.

Saves are write-then-rename. This file holds every tuned setting, and a crash
during an in-place write leaves a truncated document that fails to parse on the
next start.

Ported modules carry forward the fixes made in the JS version, nearest-target
selection (not tracker insertion order), correct `WallRange` semantics,
projectile filtering, and the bounded rotation window.

## Build and run

```bash
go build ./cmd/nebula-proxy
./nebula-proxy -listen 0.0.0.0:19132 -remote play.lbsg.net:19132
```

```bash
go test ./...
```

### Flags and environment

| Flag | Default | Meaning |
|---|---|---|
| `-listen` | `0.0.0.0:<ListenPort from config>` | client-facing RakNet address |
| `-remote` | `<TargetHost>:<TargetPort from config>` | upstream server to connect to |
| `-config` | `config.json` | config path (created on first save if absent) |
| `-debug` | off | verbose (Debug-level) logging |
| `-dashboard` | `127.0.0.1:8080` | dashboard address; empty string disables it |
| `-auth-cache` | `auth-cache/token.json` | where the Xbox Live token cache lives |
| `-no-auth` | off | skip Xbox auth on the LOCAL listener only |
| `-offline` | off | do not sign in to Xbox Live at all (works only on servers that allow it) |

| Environment | Meaning |
|---|---|
| `NEBULA_LICENSE_KEY` | licence key; overrides the config's `"licenseKey"` |
| `NEBULA_API_URL` | licence server base URL (default: the production portal) |
| `NEBULA_DEV_NO_LICENCE` | `1` skips the licence gate entirely, development only, loudly logged |

## The free plan (`internal/licence/plan.go`)

A free licence is a **proxy key with one column different**. The Discord bot
issues it with `/licence grant plan:Free` or `/key gen plan:Free`, DM'd like any
other key; the difference is that `keys.plan` says `free`, and `/validate`
returns that in the `group` field the proxy has always read and never used.

**The plan comes from the server and nowhere else.** Not config.json, not a
flag, not the dashboard. An entitlement the machine it restricts can edit is
not an entitlement. It is re-read on every five-minute revalidation, so an
upgrade lands without a restart and a downgrade cannot be outlived.

### What free gets

The cheat half of the plan is `freeModules` in `internal/licence/plan.go`:
`AimAssist`, `AirPlace`, `AntiBot`, `AutoArmor`, `AutoClicker`, `AutoEZ`,
`AutoEat`, `AutoFish`, `AutoHub`, `AutoLog`, `AutoTool`, `AutoTotem`,
`BlockReach`, `CamTP`, `CameraTP`, `Criticals`, `DamageText`, `Disabler`,
`ESP`, `EnemyTPv2`, `FlyLB`, `Friends`, `FullBright`, `HitEffects`, `Hitbox`,
`ItemESP`, `ItemTags`, `KillAura`, `MobAura`, `MobESP`, `NoClip`, `NoHurtCam`,
`PortalESP`, `Reach`, `SMTransfer`, `SpawnerESP`, `TP`, `TPMine`, `TargetInfo`
and `TriggerBot`.

`Friends` and `FullBright` are force-enabled at construction, so the plan never
had to lock them: they were open by accident. Naming them in the free set makes
that a decision rather than a side effect, and keeps them free if either is
ever un-forced.

It also keeps **the client surface** (`freeAlso`): the HUD family, chat, the
command handler, the join and warning messages — what a free user needs to
drive the modules at all. The menus themselves are paid: ClickGUI and the
QuickPanel sit in `lockedMisc` with the rest of the premium Misc list —
`ACDetector`, `ArmorHUD`, `AutoJoin`, `CheatDetector`, `Craft`, `DisablerCC`,
`Keystrokes`, `Music`, `NameProtect`, `Smelt`, `Stats`, `TpLog`, `Visualize`,
`Visuals` and the rest — so the free plan is driven from `/.` commands and
the HUD, not the forms.

Everything else in Combat, Movement, Player and World is locked, plus every
Misc module `lockedMisc` names. The cheat categories lock by default, so **a
module added to one later is locked without anybody listing it** — the
direction a mistake should fall. Misc is the other direction on purpose: an
unlisted one stays open, so `lockedMisc` has to be kept current by hand or a
new premium utility ships free.

### Where it is enforced

`Base.SetEnabled` refuses, not just `Manager.Toggle`. `SMTP`, `SurfaceTP`,
`Goto` and `TPTesting` all switch *themselves* on out of their own command
handler and never pass through the manager, so gating the manager alone would
have left `/.smtp 1 2 3` as a working way to run a module the plan excludes.
The manager still owns the message, because it is the only side with somewhere
to send one.

That one choke point covers every door: the ClickGUI, the quick panel, chat
commands, the dashboard's toggle route, presets, and config restore — a
hand-edited `"enabled": true` in config.json goes through `Manager.Toggle` and
is refused like everything else. Each surface says the same thing, and none of
them reports a refused module as switched on; `ApplyPlan` also switches off
anything that was already running when the plan narrowed.

Hidden modules are skipped — they are plumbing, not features. A *forced*
module is the subtler case: `DisablerCC` is paid but still force-arms on
CubeCraft for a paying user, and the host lists in `internal/config` know
nothing about plans. The lock wins underneath the flag instead — `SetForced`
refuses while a `PlanLock` is set, and `ApplyPlan` strips a force it locks,
putting it back if the plan widens again — so a free join on CubeCraft gets
the upgrade notice, not the bypass.
`TestNothingTheConfigCanForceOnIsAPaidModule` walks a free session through
every host policy and asserts no paid module comes out of it forced or
enabled.

### Free is the proxy, and only the proxy

`database.validate_key` refuses a free key for any product but `proxy`. That is
one check rather than a fourth product name, and it covers the injector, the
watchdog and the bots runner together — including the download routes, which
validate with no product name at all. The client explains the refusal as
`free_plan_proxy_only` rather than "invalid key".

### The join banner

`§bNebula §8-> §ffree`, where a paid licence counts down. The countdown answers
"when do I lose this"; on a free plan the answer is never, and what the player
wants to know there is which plan they are on.

## Licensing

The proxy runs a **startup licence gate** (`internal/licence`): HWID-bound,
fails closed, an unlicensed process never listens, dials, or serves the
dashboard. After startup it re-validates on a five-minute poll with a
killswitch; a revocation saves the config, logs, and exits rather than serving
another packet. The key is resolved from `NEBULA_LICENSE_KEY` first, then
`"licenseKey"` under `"proxy"` in `config.json`. The closed-beta allowlist is a
server-side concern (off by default on the portal); the client-side gate is
always on unless `NEBULA_DEV_NO_LICENCE=1`.




