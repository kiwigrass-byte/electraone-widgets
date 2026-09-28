# Electra One v5.0.0f — beta findings

Device under test: mk2, hw revision 3.0, serial EO2-4312746f.
Updated 4.1.4 → 5.0.0f on 2026-09-28. Probed over USB SysEx from Linux (ALSA).

## 1. Update

- Online update from 4.1.4 succeeded.
- **Chrome crashed on the host during/right after the update.** Controller was
  unaffected and came back on 5.0.0f. Worth reporting — no diagnostics captured.

## 2. Runtime info is far richer than documented

`F0 00 21 45 02 7E F7` — the docs say "only the information about free memory is
included at the present time". On 5.0.0f it returns:

```json
{"freeRam":31824304,"uptime":54412,"midiDrops":276,"callbackDrops":0,
 "callbackRuns":0,"timerLoad":2,"timerSkipped":1526,"scriptAborts":0,
 "timerErrors":0,"remoteCoalesced":0,"remoteDrops":0,
 "modes":{"lowSensitivity":false,"midiLearn":false,"debug":false,
          "timer":true,"screenshot":false},
 "stats":{"win":54,"since":54306,"cpu":[92,97],"loop":[663,384],
          "lat":{"router":[0,0,0],"midiMap":[0,0,0],"uiMap":[0,0,0]},
          "mem":{"heap":31824304,"peak":1794048,"lua":148,
                 "stack":[37,"Application Thread"]},
          "q":{"in":0,"io":0,"dev":0,"host":0,"rtr":0,"map":0,"cmd":0},
          "over":{"timerSkip":1526,"timerErr":0,"abort":0,"midiOut":276,
                  "midiIn":0,"ioTmo":0,"cb":0,"map":0,"remote":0,"router":0,
                  "paintLate":192,"paintMax":141}}}
```

This is the "system statistics" surface for agent workflows. **The docs page for
it is stale.** A machine-readable schema for this payload would be worth having.

There is a `modes.screenshot` flag — screenshot mode appears to be a runtime
state, not just a web-app action.

## 3. `schedule` module — verified on device, undocumented publicly

`docs.electra.one/developers/luaext.html` still serves the 4.0 text ("you must
have Firmware version 4.0 or later") and documents no `schedule` module.

Enumerated from the device:

| Call | Behaviour (observed) |
|---|---|
| `schedule.after(ms, fn)` | one-shot |
| `schedule.every(ms, fn)` | repeating; returns a **number** handle |
| `schedule.cancel(handle)` | returns `true` |
| `schedule.now()` | milliseconds since boot |
| `schedule.stats()` | see below |
| `schedule.whenNotes(...)` | **undocumented, signature unknown** |

`schedule.stats()` → `{capacity=32, errors=0, pending=0, ran=0, whenNotes=0,
worstOverrun=0}`. So **32 concurrent jobs**, with error and overrun counters.

## 4. `timer` is NOT removed in v5

Contrary to what the beta thread implies, the whole `timer` table is still
present on 5.0.0f: `disable, enable, getBpm, getClockBpm, getContendedTicks,
getFailedTicks, getLoad, getMaxDurationUs, getPeriod, getPeriodUs,
getSkippedTicks, isEnabled, isSuspended, onTick, setBpm, setClockBpm, set…`

Nothing breaks. Migration to `schedule` is a choice, not a repair.

**The reason to migrate is composability.** `timer.onTick` is a single global
entry point per preset, so two animated widgets cannot coexist. `schedule`
allows 32 independent jobs. For third-party Lua modules sharing one preset,
that is the difference between one widget at a time and composing freely.

## 5. Third-party modules already exist in the protocol

The Commit Files command (`F0 00 21 45 04 2D`) accepts `"location": "modules"`
and `"type": "luaModule"` — extracted from the editor bundle at a 4.x baseline.
The provisioning mechanism has a foothold in the protocol already.

## 6. Lua REPL over SysEx works well

`F0 00 21 45 08 0D <ascii lua> F7` executes without persisting; `print()` output
comes back on the CTRL port as SysEx. This is the whole agent feedback loop in
one call, on Linux, with nothing but ALSA.

## 7. Non-ASCII Lua source fails silently over SysEx — the big one

Any Lua source containing a byte above 0x7F cannot be sent to the controller
over SysEx. MIDI forbids data bytes >= 0x80, so the message is malformed.

**The failure is completely silent**: no ACK, no NACK, no log line, no response
of any kind. The agent sees nothing and cannot tell the difference between
"executed and printed nothing" and "never arrived".

Reproduced deterministically:

```
print("plain ascii")          -> "lua: plain ascii"
-- dash — here            -> (nothing)
print("after em dash")
-- accent été                  -> (nothing)
print("after accent")
```

Then across the widget library: every file containing a non-ASCII byte was
silent, every cleaned file answered. 9 of 20 widgets were affected; after
cleaning 214 characters across 29 files, all of them syntax-check on device.

**Why this matters for the AI workflow specifically:** language models emit em
dashes, curly quotes, arrows and middots constantly, in comments and in UI
strings. An agent writing Lua for Electra will produce unsendable source on a
regular basis and get no feedback telling it why. This is probably the single
highest-value error message the firmware could add.

Suggestions, in order of cheapness:
1. NACK with a reason when the SysEx payload contains a byte >= 0x80.
2. Document the 7-bit constraint on the Upload/Execute Lua pages.
3. Consider accepting an escaped or 7-bit-encoded transport for Lua source, so
   display strings can legitimately contain degree signs, flats and so on.

Open question we could not answer from here: whether the device font renders
non-ASCII glyphs at all once the source arrives by another route (web editor).

## 8. Occasional silent response loss

Independently of the above, the REPL occasionally drops a reply: the same
request repeated immediately afterwards answers normally. The device's own
counters report `midiDrops: 276` and `over.midiOut: 276` at idle. An agent loop
needs a retry, and would benefit from a transaction id echoed on every reply.

## 9. Library migration to the scheduler (done)

All 9 animated widgets moved off `timer.onTick` to `schedule.every`, each with
its own named callback (`cubeLfoTick`, `stepSeqTick`, `arpTick`, …) instead of
a single shared global. `send-lfo`, which starts and stops at runtime, now
keeps the handle and uses `schedule.cancel`. No `timer.*` call remains in the
library. All 20 widgets syntax-check on 5.0.0f.
