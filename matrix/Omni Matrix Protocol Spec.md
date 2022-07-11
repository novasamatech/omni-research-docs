# Establishment of the room

Words MUST, SHOULD and MAY are to be interpreted according to [rfc2119](https://datatracker.ietf.org/doc/html/rfc2119)

Everything below is described in the context of some particular MST account M.

## Definitions
**Omni room** - a matrix Room that follows rules for the name and topic as defined in the "Room creation" section

For each MST account there MUST be a separate Omni room. Each Omni room MUST handle MST operations for all networks.
Omni room MUST be private (invite-only).
M Room refers to any Omni room for the M.

## Room creation

If the user is not a member of any M room and the user sees no invites to any M room he MAY create a new M room.
To do that he MUST perform the following sequence of actions in the exact same order:

1. set the title to `Omni MST | %MST account addess%`
2. send `m.room.topic` state event with the following content:
```json
{
  "omni_extras": {
    "mst_account": {
      "accountName": "MST account name",
      "threshold": 2,
      "signatories": [
        %first signatory address%,
        ...
      ],
      "address": %mst account address%
    },
    "invite": {
      "public_key": %inviter's public key%,
      "signature": %creator.sign(MST address + roomId)%
    },
  },
  "topic": 'Room for communications for' + %mst account address% + 'MST account'
}
```
where `roomId` is the room identifier returned by the matrix server once the room is successfully created.
3. Invite other known signatories to the room

Each address in the definition above MUST be an SS58 address formatted with the `prefix=42`.

`roomId` is included in the signature to avoid possible signature reuse

## Room joining
The invite is considered valid if all conditions below are met:
- the inviting room is the M room
- invite signature is valid

If the user sees one or more valid invites:
- If the user is not a member of the M room then he MUST accept the invite with the lowest score.
- If the user is a member of the M room then he MUST leave the current room and accept the invite with the lowest score if this score is lower than the score of the current user's room.

In the cases above, if there are multiple valid invites to the same room, the user MUST accept one of them. 

## Duplicate resolution strategy
To resolve duplicate rooms we define the following scoring function for the M room:

`score(M room) = alphabetical order of creator's SS58 address formatted with 42 prefix`

In other words, rooms should be compared by the creator's address, which should, in turn, be compared using alphabetical ordering (ABC < BCA)

## Extra invites
If the user is a member of the M room and not all signatories have joined it, the user MAY invite not-yet-joined signatories, if they are known to him.

# Room communication

Everything below assumes that user is a member of M room.
In opposite case below instructions are ignored. 

**Important!** Extrinsic hash value is not reliable and must not be used blindly in the logic where funds are spent

## MST operation initiation
When user initiates on-chain MST he SHOULD send the following event to the M Room:
``` javascript
const eventType = "io.novafoundation.omni.mst_initiated"
const content = {
        "chainId": "0x00", // genesis hash of the network MST sent in
        "callHash": "0x00",
        "callData": "0x00",
        "senderAddress": "5Di3pwfKkaDZsEECgm89Zovxo71k7WADR2WrHvfwrVxRDEey",
        "description": "Description of MST"
}
```
Since `multisig` pallet does not allow multiple MST operations with the same `callHash` to exists, there should not be any clashes.

## MST operation approval
When user approves previously initiated MST operation, he SHOULD send the following event to the M room:
``` javascript
const eventType = "io.novafoundation.omni.mst_approved"
const content = {
        "chainId": "0x00",
        "callHash": "0x00",
        "extrinsicHash": "0x00",
        "senderAddress": "5Di3pwfKkaDZsEECgm89Zovxo71k7WADR2WrHvfwrVxRDEey",
}
```
## MST operation final approval
When the user performs the final approval for the previously initiated MST operation, he SHOULD send the following event to the M room:
``` javascript
const eventType = "io.novafoundation.omni.mst_executed"
const content = {
        "chainId": "0x00",
        "callHash": "0x00",
        "extrinsicHash": "0x00",
        "senderAddress": "5Di3pwfKkaDZsEECgm89Zovxo71k7WADR2WrHvfwrVxRDEey",
}
```
## MST operation cancelling 
When user cancels previously initiated MST operation, he SHOULD send the following event to the M room:
```javascript
const eventType = "io.novafoundation.omni.mst_cancelled"
const content = {
        "chainId": "0x00",
        "callHash": "0x00",
        "extrinsicHash": "0x00",
        "senderAddress": "5Di3pwfKkaDZsEECgm89Zovxo71k7WADR2WrHvfwrVxRDEey",
        "description": "Description of cancellation", // optional
}
```
## MST operation chat message
When user sends the message associated with the MST operation, he SHOULD send the following event to the M room:
```javascript
const eventType = "m.room.message"
const content = {
    "body": "Message body", // the chat message that user has sent
    "msgtype": "m.text",
    "callHash": "0x00"
}
```

## JS SDK code samples

### Create room according to the spec
```javascript
async function inviter() {
    const roomId = await client.createRoom({
        preset: Preset.TrustedPrivateChat,
        name: "Omni MST | 12eLyGvPcMV3JmEieQB9hxm7ej1PooiMVXFLTDfJQaywPNPM",
    })

    await initialStateEvents(roomId.room_id)

    await inviteOthers(roomId.room_id, secrets.others)
}

async function initialStateEvents(roomId) {
    const omniExtras = {
        "mst_account": {
            "accountName": "MST account name",
            "threshold": 2,
            "signatories": [
                "5Di3pwfKkaDZsEECgm89Zovxo71k7WADR2WrHvfwrVxRDEey",
                "5Di3pwfKkaDZsEECgm89Zovxo71k7WADR2WrHvfwrVxRDEey",
                "5Di3pwfKkaDZsEECgm89Zovxo71k7WADR2WrHvfwrVxRDEey"
            ],
            "address": "5Di3pwfKkaDZsEECgm89Zovxo71k7WADR2WrHvfwrVxRDEey" // mst account address
        },
        "invite": {
            "inviter": "5Di3pwfKkaDZsEECgm89Zovxo71k7WADR2WrHvfwrVxRDEey",
            // 64 bytes hex-encoded signature == 128 symbols
            "signature": "0x12345678901234567890123456789012345678901234567890123456789012345678901234567890123456789012345678901234567890123456789012345678"
        }
    }

    const topicContent = {
        "topic": "Room for communications for 5Di3pwfKkaDZsEECgm89Zovxo71k7WADR2WrHvfwrVxRDEey MST account",
        "omni_extras": omniExtras
    }

    await client.sendStateEvent(roomId, "m.room.topic", topicContent)
}

async function inviteOthers(roomId, others) {
    for (const other of others) {
        await client.invite(roomId, other);
    }
}
```

### Get invites
```javascript
function getInvites() {
    return client.getRooms().filter((room) => room.getMyMembership() === "invite")
}
```
### Invite validation
```javascript
function extractValidationInfoFromInvite(room) {
    const eventType = "m.room.topic"
    const strippedStateKey = '' // when user is invited, he only sees stripped state, which has '' as state key for all events

    const topicEvent = room.currentState
        .events.get(eventType).get(strippedStateKey)
        .event

    const omniExtras = topicEvent.content["omni_extras"]
}
```
### Sending events
```javascript
async function sendEvent(roomId) {
    const eventTypeBase = "io.novafoundation.omni"
    const MSTInitiated = eventTypeBase + ".mst_initiated"
    const content = {
        "chainId": "0x00", // genesis hash of the network MST sent int
        "callHash": "0x00",
        "callData": "0x00",
        "senderAddress": "5Di3pwfKkaDZsEECgm89Zovxo71k7WADR2WrHvfwrVxRDEey",
        "description": "Description of MST"
    }
    const eventId = await client.sendEvent(roomId, MSTInitiated, content, null, null)
    console.log(eventId.event_id)
}
```
### Viewing events
```javascript
async function viewEvents(roomId) {
    const events = client.getRoom(roomId).getLiveTimeline().getEvents()

    events.forEach((event) => console.log(event.event.type, event.event.content))
}
```
