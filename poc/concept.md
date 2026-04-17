# Conceptual Trigger Sequence

## Prerequisites

A FORTE instance with a FB type that has a struct-typed data input port.
`GEN_STRUCT_DEMUX` from the `convert` module satisfies this when built in.
The management port must be reachable (TCP 61499).

## Protocol

4diac management protocol: XML over TCP. No authentication.

## Sequence

### Step 1: Establish two source FBs

Create two FBs whose output ports are type-compatible with individual
members of the destination struct type.

### Step 2: Create destination FB with struct DI port

Create a `GEN_STRUCT_DEMUX` instance configured for the target struct type.
This FB has a single struct-typed DI port (`IN`).

### Step 3: Establish gathering connection (two CreateConnection commands)

Connect the first source output to `DST.IN.<member1>`.
Connect the second source output to `DST.IN.<member2>`.

After both commands:
- DST DI slot for `IN` holds a live `CGatheringDataConnection*`
- `mGatheringData` contains two `CMemberDataConnection` entries

### Step 4: Trigger (one DeleteConnection command)

Send a `DeleteConnection` with destination set to `DST.IN` only, with
no member suffix. This is the path that causes `getInputConnection` to
return the `CGatheringDataConnection` itself via `getMemberConnection`
on an empty subspan.

After this command:
- Both `CMemberDataConnection` objects are freed inside `disconnect()`
- `CGatheringDataConnection` is freed by `resource.cpp`
- DST DI slot for `IN` holds a dangling pointer

### Step 5: Trigger UAF read

Any of the following causes a dereference of the dangling slot:

- Send a second `DeleteConnection` to `DST.IN`
- Send a `CreateConnection` targeting `DST.IN`
- Send an event that causes DST to execute (fires `readInputData`)
- Send a monitoring or query command that iterates DI connections

## Expected Behavior on ASAN Build

```
==ERROR: AddressSanitizer: heap-use-after-free
READ of size 8
    #0 CDelegatingDataConnection<...>::isDelegating()
    #1 CResource::deleteConnection(...)
```

On a release build without sanitizers the process either continues with
corrupt state or crashes on the next vtable dispatch through the dangling
pointer, depending on heap state at the time of the subsequent allocation.
