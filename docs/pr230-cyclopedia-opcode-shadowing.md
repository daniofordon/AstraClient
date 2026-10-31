# Review note: Cyclopedia character info opcode shadowing

Draft for [Mateuzkl/AstraClient#230](https://github.com/Mateuzkl/AstraClient/pull/230)
("Port Cyclopedia character info to AstraClient"). This finding comes from
reviewing the server counterpart,
[forgottenserver-downgrade-1.8-8.60#345](https://github.com/Mateuzkl/forgottenserver-downgrade-1.8-8.60/commit/6f49cfb888f7253b4791a73c7755ec90a9e81f20),
against client `main` at `80c59ec7`.

Status: not posted yet. The text below the line is the comment body, ready to
copy into the PR.

---

## `0xDA` is still registered in Lua, so the new `parseCyclopedia` is never reached

The client dispatches each server opcode into Lua **before** the C++ switch:

```cpp
// src/client/protocolgameparse.cpp:186
const bool handledByLua = callLuaField<bool>("onOpcode", opcode, msg);
...
if (handledByLua) {
    ...
    continue; // the switch below is skipped entirely
}
```

`ProtocolGame:onOpcode` returns `true` for every opcode present in
`opcodeCallbacks` (`modules/gamelib/protocolgame.lua:7-11`), and
`modules/game_protocol/protocol.lua` still registers a handler for the same
opcode this PR starts handling:

```lua
-- modules/game_protocol/protocol.lua:31
CyclopediaCharacterInfo = 0xDA,

-- modules/game_protocol/protocol.lua:500
registerOpcode(ServerPackets.CyclopediaCharacterInfo, function(protocol, msg)
	local type = msg:getU8()
	if g_game.getProtocolVersion() >= 1215 then   -- false at 860
		local error = msg:getU8()
```

Because `registerProtocol()` runs on `onEnterGame` / `onPendingGame` /
`onGameStart` and the module is `autoload: true`, this registration is always
active during gameplay. Consequences:

* None of the new types (3, 4, 5, 6, 7, 8, 11, 13, 14, 15) reach the rewritten
  `ProtocolGame::parseCyclopedia`, so the new `onAstra*` callbacks are never
  invoked and the `mods/game_cyclopedia` adapters added here stay idle.
* The Lua handler still parses the **pre-#345 layout**: it does not consume the
  error byte, reads item summary as `u16 id + u32 amount` and has no outfit
  colour / familiar block. Against a server that ships
  `forgottenserver-downgrade-1.8-8.60#345` that desynchronises the frame
  silently — there is no length validation on that path.
* The `>= 1215` gate is dead code in this build: `init.lua` sets
  `version = 860` and the server accepts only 860
  (`CLIENT_VERSION_MIN` / `CLIENT_VERSION_MAX`), so the error byte is never
  skipped.

Worth noting the UI side is already wired in `main`: the character tabs call
`g_game.requestCyclopediaData(...)`
(`mods/game_cyclopedia/classes/character.lua:294,314-317`) and
`g_game.updateCharacterTitle(...)`
(`mods/game_cyclopedia/classes/titles.lua:200,210,222`), but both are no-op
stubs from the `gameNoops` table (`modules/corelib/globals.lua:538-586`,
`g_game[name] = g_game[name] or noop`) until this PR adds the bindings. So this
PR is what makes those tabs do anything — provided it is not shadowed.

Two ways to resolve it:

**Option 1 — let C++ own `0xDA`** (consistent with the rest of this PR). Remove
the Lua registration, i.e. the `CyclopediaCharacterInfo` entry in
`ServerPackets` and the whole `registerOpcode(...)` block:

```diff
-	CyclopediaCharacterInfo = 0xDA,
```

```diff
-  registerOpcode(ServerPackets.CyclopediaCharacterInfo, function(protocol, msg)
-	local type = msg:getU8()
-	if g_game.getProtocolVersion() >= 1215 then
-		local error = msg:getU8()
-		...
-	end
-	if type == 0 then -- Basic Information
-	...
-  end)
```

**Option 2 — keep Lua in charge.** Port the new layout into the Lua handler
(error byte, item summary with `tier` and name, outfit colour / familiar blocks)
and call the same signals from there, leaving `parseCyclopedia` unused.

`0x81` needs nothing extra: no Lua registration exists for it, so
`sendCharacterTitleSelection` reaches the C++ parser as intended.

Are you planning to take option 1, and should this land together with
`forgottenserver-downgrade-1.8-8.60#345`? Without that pairing, current `main`
clients keep the old achievements layout while the new server sends the error
byte, which breaks the (otherwise working) achievements window.
