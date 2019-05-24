# Chatkit mobile coding challenge

We would like you to write a component (or components) to model the state of a
Chatkit Room, present that state to the developer, and incorporate changes to
the state coming in from the backend as well as requested by the developer.

## Entities

This is a description of the entities provided by the backend.

### User

Represents a user of the chat system.

```
User: {
  id: unique identifier for the user
  name: a display name for the user
}
```

### Room

A room is a container for a stream of messages.
Each message in the system may exist in exactly one room.

```
Room: {
  id: unique identifier for the room
  name: a display name for the room
}
```

### Membership

A membership is the link between a user and a room, indicating that the user
is in a room. Memberships are persistent - whether the user is online or
offline, they will remain a member of the room until they are removed.

```
Membership: {
  userId: this user
  roomId: is part of this room
}
```

### Message

A message belongs to exactly one room. It has exactly one sender, which is a
user entity.
Messages have IDs which increase over time, defining the order of messages in
the room.

```
Message: {
  id: ordered numeric identifier
  messageText: text of the message
  userId: ID of the user who created the message
}
```

### Cursor

A cursor point to a specific message in a room, and represents that a
particular user has read all messages up to and including the referenced
message in a particlar room.

```
Cursor: {
  userId: this user
  messageId: has read up to this message
  roomId: in this room
}
```

## Inputs

The client receives three different streams of data relating to a room:

- Messages, a stream of `Message` entities, as they are created by other
  users. Messages created by this client are also received as part of the
  stream.
- Memberships, a stream of events describing which users are currently part of
  the room.
- Cursors, a stream of events describing who has read which messages in the
  room.

Both Memberships and Cursors streams have two classes of events:

- Initial state: when a connection is first made to the backend, the entire
  state for the entity is transmitted
- Deltas: after the initial state is transmitted, changes to the state are
  sent as individual events in real time, e.g. `user_added_to_room`, or
  `cursor_updated`

Because your connection to the backend might be lost during a session, you
must assume that an initial state event can actually arrive at any time, and
should replace any existing state stored for the entity.

### Events

#### Messages

There is no initial state event.

Delta event:

```
NewMessage: {
  Message: the message entity, as described above
}
```

#### Memberships

Initial state:

```
InitialMembers: {
  members: [
    userId: Id of each user who is a member
  ]
}
```

Delta:

```
MemberAdded: {
  userId
}

MemberRemoved: {
  userId
}
```

#### Cursors

Initial state:

```
InitialCursors: {
  cursors: [
    Cursor: a cursor entity as described above
  ]
}
```

Delta event:

```
CursorUpdated: {
  Cursor: a cursor entity as described above
}
```

## Problem statement

These different entities are provided in real time to the client by the
backend. It is your challenge to take these different entity streams, and
construct a useful data model to be consumed by the developer.

The backend is made up of multiple microservices from which the client
receives these entities in real time. However, that means that the streams
received by the client are not synchronised, and so the order of events
received by the client may not be ideal - particular if the client loses its
connection to one of more of the backend microservice.

Your solution should hide these details from the developer and present a
consistent view of the current state of a room and its dependent entities to
the developer.

It should allow the developer to query the current state of the room and
receive consistent results (for example, they should not see a cursor in the
data model for a user who is not a member, or for a message which is not yet
in the model). This might be used for the initial population of the app UI.

It should also allow the developer to be notified of changes to the data
model, so that they can update their representation in the UI.

This is a real problem currently handled (with a lot of room for improvement)
by the Chatkit client SDKs. We plan to move much of this problem to the
backend and provide a more unified view of the data model to the SDK in a
single stream. However, when we begin to add features to support offline
usage to the SDKs, this kind of state consistency management problem will
return, and be much more difficult to avoid, so we think this is still a
reasonably representative test.
