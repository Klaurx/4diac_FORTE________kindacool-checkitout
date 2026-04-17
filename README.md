# 4diac FORTE: Use-After-Free in deleteConnection

## Summary

A use-after-free exists in `CResource::deleteConnection` in
`core/src/resource.cpp`. When a `DeleteConnection` management command
targets a gathering data connection using a port-only destination path,
`CGatheringDataConnection` is freed but the FB data input slot is not
nulled. Any subsequent operation touching that slot dereferences freed
memory.

Trigger is deterministic. No race condition. No heap grooming required
to reach the dangling pointer state.

## Affected Files

| File | Location |
|------|----------|
| `core/src/resource.cpp` | `CResource::deleteConnection` |
| `core/src/gatherdataconn.cpp` | `CGatheringDataConnection::disconnect` |

## Impact

Remote unauthenticated attacker. Reachable via the management protocol
on TCP 61499. Multiple post-free dereference paths exist. On targets
without modern mitigations the vtable reuse path leads to controlled
indirect call.

## Contents

```
docs/writeup.md     full technical analysis
poc/concept.md      trigger sequence and expected behavior
patch/fix.diff      proposed fix
repro/README.md     ASAN build and reproduction steps
```
