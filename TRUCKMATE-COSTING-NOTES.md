# TruckMate custom costing — design notes (fork of valhalla/valhalla @ 3.7.0)

Branch: `feature/unprotected-left-costing`. Owner: TruckMate (sharpninja/valhalla).
Purpose: add a custom costing rule that avoids **unprotected left turns** and folds TruckMate's static route friction into the engine. See the TruckMate plan `greedy-popping-alpaca.md`.

## Hard requirement
A left turn is **protected** only if the oncoming approach is gated by an **all-way stop** OR a **traffic signal**. Yield/give-way, two-way stop (oncoming not stopped), and uncontrolled = **unprotected**. Take an unprotected left only when avoiding it would detour more than a configurable distance (default 1 mile; mi/km selectable), realized as an in-engine finite penalty.

## Phase 1 data-fidelity finding (gate result: GO, with conservative fallback)

### Available in stock 3.7.0 graph
- `baldr::DirectedEdge`: `traffic_signal()`, `stop_sign()`, `yield_sign()` — each is the control "at the END of this edge" (i.e., the approach the edge represents, entering its end node). (`valhalla/baldr/directededge.h`)
- `baldr::NodeInfo`: `traffic_signal()`, `intersection()` type, `named_intersection()`. (`valhalla/baldr/nodeinfo.h`)
- Lua tag transform (`lua/graph.lua`) sets directional `forward_stop`/`backward_stop`, `forward_yield`/`backward_yield`, `forward_signal`/`backward_signal` from `direction` / `traffic_signals:direction`.

### NOT available (the dominant risk, confirmed)
- No explicit **all-way vs two-way** stop classification. `graph.lua` does **not** read `stop=all` / `stop=minor` / `all_way`.
- A bare `highway=stop` node with no `direction` sets neither `forward_stop` nor `backward_stop` (none of the Lua branches fire) — so many real all-way stops carry no edge stop flag.

### Consequence for the rule
"Protected by all-way stop" cannot be read directly. It must be **inferred**: a node is an all-way stop if **every inbound approach edge** has `stop_sign()` true. This depends on per-approach OSM tagging quality, which is sparse.

### Decision (accepted in plan; simplified per user 2026-06-27)
The conflict for a left turn is a SINGLE approach (the traffic you cross), not the whole node. Do NOT infer "all-way" by scanning every inbound edge. Instead:
1. Resolve the one **conflicting approach** edge at the turn node — the approach coming from the left / the opposing-through relative to the maneuver (use edge headings / opposing-edge id).
2. Read that approach's directed-edge control:
   - `stop_sign()` true on the conflicting approach -> PROTECTED (this captures all-way stop without node-wide enumeration: the conflicting side is stopped).
   - node `traffic_signal()` true -> PROTECTED.
   - else (no stop, `yield_sign()` only, or untagged) -> UNPROTECTED; apply the avoidance penalty.
- **Conservative fallback:** untagged conflicting approach (bare `highway=stop`, no `direction`) shows no `stop_sign()` -> treated as UNPROTECTED. Never wrongly permits an unprotected left; may over-avoid where tagging is sparse, bounded by the configurable detour distance.
- **Optional precision upgrade (TR-OSMNAV-DATAFIDELITY-035):** augment `lua/graph.lua` + mjolnir to set `forward_stop`/`backward_stop` for bare `highway=stop` and honor `stop=all`/`stop=minor`, improving conflicting-approach stop capture. Tracked, not required for the conservative path.

### Costing access note (for Phase 2)
The decision is LOCAL: one conflicting-approach edge + node signal. The tile-aware `TransitionCost` overload is needed only to fetch that one approach edge (e.g. via opposing-edge id / node edge by heading), not to scan all edges. Confirm the exact 3.7.0 signature when implementing in `sif/`.

## Custom costing option contract (sent by TruckMate under costing_options.{truck,auto})
- `unprotected_left_avoidance_meters` (double; 0/absent disables). TruckMate converts the operator's mi/km setting to meters.
- `enable_static_friction` (bool; default true). When false (Class-4+/over-8' profile), comfort biases (maneuver/toll/ferry/road-class) are not applied. The unprotected-left reject is INDEPENDENT of this flag and always applies.
