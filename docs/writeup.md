# Technical Analysis: Use-After-Free in CGatheringDataConnection::disconnect

## 1. Invariant

Every delegating connection must guarantee that after `disconnect()`
returns, no FB interface slot holds a pointer to it or to any object it
owns. The mechanism is `CFunctionBlock::connectDI(portId, nullptr)`,
which writes `nullptr` into the raw `CDataConnection*` slot.

`CDataConnection::disconnect()` upholds this:

```
// dataconn.cpp
if (dstGatheringConnection->empty()) {
    paDstFB.connectDI(dstPortId, nullptr);
    delete dstConnection;
}
```

`CGatheringDataConnection::disconnect()` does not.

## 2. Inheritance

```
CDataConnection
  CDelegatingDataConnection<T>    isDelegating() -> true  [final]
    CGatheringDataConnection      isGathering()  -> true  [final]

CMemberDataConnection  : CDelegatingDataConnection<CIEC_ANY>
```

`CGatheringDataConnection` inherits `isDelegating() -> true`.
This is the property that causes `resource.cpp` to call `delete dstCon`
after `disconnect()` returns.

## 3. Root Cause

```
// gatherdataconn.cpp
EMGMResponse CGatheringDataConnection::disconnect(
    CFunctionBlock &paDstFB,
    std::span<const StringId> paDstPortNameId)
{
    for (const auto [member, connection, memberName] : mGatheringData) {
        connection->disconnect(paDstFB, memberName);
        if (connection->isDelegating()) {
            delete connection;
        }
    }
    mGatheringData.clear();
    return EMGMResponse::Ready;
    // connectDI(port, nullptr) never called.
    // paDstFB DI slot still holds `this`.
}
```

Caller in `resource.cpp`:

```
if (CConnection *dstCon = getInputConnection(paDstNameList)) {
    const EMGMResponse retVal =
        dstCon->disconnect(*dstFB, {dstIt, paDstNameList.cend()});
    if (retVal == EMGMResponse::Ready && dstCon->isDelegating()) {
        delete dstCon;   // object freed
    }
    return retVal;
    // DI slot on dstFB is now dangling
}
```

## 4. How getInputConnection Returns the Gathering Connection Itself

The trigger requires the destination path to contain only the port name
with no member suffix. `getInputConnection` calls `getMemberConnection`
on an empty subspan. The base class returns `this`:

```
// dataconn.h
virtual CDataConnection *getMemberConnection(
    const std::span<const StringId> paMemberName)
{
    if (paMemberName.empty()) {
        return this;
    }
    return nullptr;
}
```

So `dstCon` in `resource.cpp` is the `CGatheringDataConnection` itself,
not one of its members.

## 5. Execution Sequence

```
precondition:
    two CreateConnection commands establish member connections to the
    same struct-typed DI port on DST.
    DST DI slot = CGatheringDataConnection* (live)
    mGatheringData = [member1 -> CMemberDataConnection,
                      member2 -> CMemberDataConnection]

trigger:
    DeleteConnection(destination = DST.IN)   // port only, no member suffix

    getInputConnection([DST, IN])
      getDIConnection("IN")         -> CGatheringDataConnection*
      getMemberConnection(empty)    -> this

    dstCon = CGatheringDataConnection*

    dstCon->disconnect()  [virtual -> CGatheringDataConnection::disconnect]
      delete CMemberDataConnection (member1)
      delete CMemberDataConnection (member2)
      mGatheringData.clear()
      return Ready
      // DI slot not touched

    dstCon->isDelegating() -> true
    delete dstCon

    DST DI slot = dangling pointer
```

## 6. Post-Free Dereference Paths

### Path A: Second DeleteConnection on same port

```
getInputConnection([DST, IN])
  getDIConnection("IN")       -> dangling ptr
  getMemberConnection(empty)  -> UAF read (virtual dispatch on freed object)
dstCon->isDelegating()        -> UAF read (vtable)
delete dstCon                 -> double-free
```

### Path B: CreateConnection to same port

```
// funcbloc.cpp::connectDI
CDataConnection **conn = getDIConUnchecked(paDIPortId);
if (*conn && paDataCon != *conn) {   // UAF read
```

### Path C: FB execution

```
// readInputData
readData(i, *var_IN, mDIConns[i]);
// mDIConns[i] is the dangling slot
// UAF read/write; corruption propagates to FB outputs
```

### Path D: Monitoring or query iteration

```
iterates DI connections -> dereferences dangling ptr -> UAF read
```

## 7. Exploitation Notes

The management protocol gives the attacker direct control over allocation
timing and object type via `CREATE` commands. After `delete dstCon` the
region is free. A subsequent `CREATE` of a FB or connection of known type
can cause the allocator to reuse it. If the attacker controls the vtable
pointer written into the reused region, the next call to `isDelegating()`
on Path A or Path D becomes a controlled indirect call.

On embedded targets running FORTE without ASLR or full RELRO this is
not a theoretical path.

## 8. Fix

```
// gatherdataconn.cpp
EMGMResponse CGatheringDataConnection::disconnect(
    CFunctionBlock &paDstFB,
    std::span<const StringId> paDstPortNameId)
{
    for (const auto [member, connection, memberName] : mGatheringData) {
        connection->disconnect(paDstFB, memberName);
        if (connection->isDelegating()) {
            delete connection;
        }
    }
    mGatheringData.clear();

    const TPortId dstPortId =
        paDstFB.getFBInterfaceSpec().getDIID(paDstPortNameId.front());
    paDstFB.connectDI(dstPortId, nullptr);   // restore invariant

    return EMGMResponse::Ready;
}
```

Fixing it here keeps the invariant local to the class. A fix in
`resource.cpp` after `delete dstCon` would stop the immediate UAF but
leave the invariant unaddressed at the class boundary.
