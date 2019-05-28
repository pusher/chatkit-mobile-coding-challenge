# Chatkit mobile coding challenge

We would like you to write a component (or components) to model the state of a
Chatkit Room, present that state to the developer, and incorporate changes to
the state coming in from the backend as well as requested by the developer.

This challenge is intended to take between 4 and 6 hours.

## Entities

This is a description of the entities provided by the backend.

### User

Represents a user of the chat system.

```kotlin
data class User(
  val id: String,               // a unique identifier
  val name: String              // a display name
)
```

### Room

A room is a container for a stream of messages.
Each message in the system may exist in exactly one room.

```kotlin
data class Room(
  val id: String,               // a unique identifier
  val name: String              // a display name
)
```

### Room Memberships

The "members" of a room are represented as a set of user ids. Memberships are
persistent - a user is a member of a room even when they are offline, and
remain a member until they leave or are removed.

### Message

A message belongs to exactly one room. It has exactly one sender, which is a
user entity.
Messages have ids which increase over time, defining the order of messages in
the room.

```kotlin
data class Message(
  val id: Integer,              // message ids are ordered
  val messageText: String,      // the content of the message
  val userId: String,           // the id of the sender
  val roomId: String            // the id of the room the message was sent to
)
```

### Cursor

A cursor point to a specific message in a room, and represents that a
particular user has read all messages up to and including the referenced
message in a particlar room.

```kotlin
data class Cursor(
  val userId: String,           // this user
  val messageId: Integer,       // has read up to this message
  val roomId: String            // in this room
)
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

Streams have two classes of events:

- Initial state: when a connection is first made to the backend, the entire
  state for the entity is transmitted
- Deltas: after the initial state is transmitted, changes to the state are
  sent as individual events in real time, e.g. `MemberAdded`, or
  `CursorUpdated`

Because your connection to the backend might be lost during a session, you
must assume that an initial state event can actually arrive at any time, and
should replace any existing state stored for the entity.

### Events

#### Messages

```kotlin
sealed class MessageInput {
  data class InitialState(
    val List<Message>
  )
  data class NewMessage(
    val message: Message
  )
}
```

#### Memberships

```kotlin
sealed class MembershipInput {
  data class InitialState(
    val roomId: String,
    val userIds: Set<String>
  )
  data class MemberAdded(
    val roomId: String,
    val userId: String
  )
  data class MemberRemoved(
    val roomId: String,
    val userId: String
  )
}
```

#### Cursors

```kotlin
sealed class CursorsInput {
  data class InitialState(
    val cursors: Set<Cursor>
  )
  data class Update(
   val cursor: Cursor
  )
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

## Expectations

We expect that you will provide code which satisfies the description above and
is demonstrated using a suite of unit tests. These is no need to provide an
executable or complete app.

Entity models representing data received from the backend are provided, and
should help get started with the inputs to your code. However, they may not
represent the models that you want to expose to the user of the code.

This challenge is difficult, and you may not be able to complete it in the
time suggested. We are most interested in how you design your code in order to
match the requirements, so we suggest that you begin not by coding, but by
understanding the problem space and laying out the design of your solution,
with some notes on why the design is a good one.

A well thought design with an incomplete implementation will be judged better
than a complete solution alone.

## Example session

Here is an example of what a session might look like, expressed as you might
in a unit test.

Comments describe the implications of what is happening, but no assertions are
made about what callbacks are emitted or what state is exposed to the user of
your code, because these decisions are left up to you.

This probably wouldn't make a good unit test, as it is trying to illustrate a
lot of different points in one session, not provide a unit test for you to
fill out.

```kotlin
@test fun sessionTranscript1() {
  val subject = MyStateModel() // an instance of your code

  // (Context, not represented here.) The user has asked to subscribe
  // to the room "lobby", so three streams are being initialised for the
  // messages, members and cursors relating to that room

  // The first backend to respond happens to be the Cursors service
  subject.received(
    CursorsInput::InitialState(
      cursors = setOf(
        Cursor(roomId = "lobby", messageId = 1, userId = "alice"),
        Cursor(roomId = "lobby", messageId = 5, userId = "bob"),
        Cursor(roomId = "lobby", messageId = 3, userId = "carol"),
        Cursor(roomId = "lobby", messageId = 3, userId = "derek"),
      )
    )
  )
  // We know about some cursors, but we do not know if the users are members
  // of the room, and neither do we know about the messages references.

  subject.received(
    MessageInput::InitialState(
      messages = listOf(
        Message(id = 1, roomId = "lobby", userId = "alice", messageText = "Hi!"),
        Message(id = 2, roomId = "lobby", userId = "carol", messageText = "..."),
        Message(id = 3, roomId = "lobby", userId = "derek", messageText = "..."),
        Message(id = 4, roomId = "lobby", userId = "bob", messageText = "..."),
        Message(id = 5, roomId = "lobby", userId = "bob", messageText = "...")
      )
    )

  // Next we receive the initial state from the memberships service
  subject.received(
    MembershipInput::InitialState(
      roomId = "lobby",
      members = setOf("bob", "carol", "derek")
    )
  )
  // Now we know who is a member of the room. Does this affect which cursors
  // we want to expose to the user?

  subject.received(
    MessageInput::NewMessage(
      Message(id = 6, roomId = "lobby", userId = "derek", messageText = "...")
    )
  )

  subject.received(
    CursorInput::Update(
      Cursor(roomId = "lobby", messageId = 6, userId = "bob")
    )
  )
  subject.received(
    CursorInput::Update(
      Cursor(roomId = "lobby", messageId = 6, userId = "derek")
    )
  )

  // We receive a cursor for a user who is not a member.
  // Perhaps there is a race with the membership backend?
  // What (if anything) should we expose to the user?
  subject.received(
    CursorInput::Update(
      Cursor(roomId = "lobby", messageId = 8, userId = "ed")
    )
  )

  // Ah, it was a race, the membership event arrives immediately afterwards.
  subject.received(
    MembershipInput::MemberAdded(
      userId = "ed"
    )
  )

  subject.received(
    MessageInput::NewMessage(
      Message(id = 7, roomId = "lobby", userId = "ed", messageText = "Hi everyone!")
    )
  )

  subject.received(
    CursorInput::Update(
      Cursor(roomId = "lobby", messageId = 7, userId = "bob")
    )
  )
  subject.received(
    CursorInput::Update(
      Cursor(roomId = "lobby", messageId = 7, userId = "derek")
    )
  )
  subject.received(
    CursorInput::Update(
      Cursor(roomId = "lobby", messageId = 7, userId = "ed")
    )
  )

  // We must have temporarily lost the stream from the memberships backend,
  // because we have received a new state snapshot which represents the entire
  // state as we should see it at this point!
  subject.received(
    MembershipInput::InitialState(
      roomId = "lobby",
      userIds = listOf("bob", "carol", "ed")
    )
  )
  // What should the state look like now if it is queried?
  // What how should we communicate the changes to the user of the code?

  // User "carol" has come back online and read some messages.
  subject.received(
    CursorInput::Update(
      Cursor(roomId = "lobby", messageId = 7, userId = "carol")
    )
  )

  // And so on...
}
```
