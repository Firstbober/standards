# Comes Unified Messages

**Version: 1.0.0**

Comes Unified Messages is a protocol for exchanging messages over network designed
to be very simple and secure as well.

## Capabilities and limits

- Max single message size is 4096 bytes.
- Rooms are supported
- Authorization is via `Authorization` header
- Encryption via pre-shared password
- People can be invited into a room
- There can be multiple room operators
- Communication is over XML and HTTP/S
- Server implementations are cross-compatible due to shared filesystem formats and layouts

## Filesystem layout

```
ROOT
|
|- info.xml
|- rooms.xml
|- users.xml
|- rooms
   |
   |- my_room
      |- info.xml
      |- messages.xml
   |
   |- csgo
      |- ....
```

## Endpoints

Master prefix for all endpoints is `/_cum`. Meaning that any domain that exposes this prefix
with all the endpoints is a proper `Comes Unified Messages` protocol server.

All endpoints need to be CORS enabled with `*` wildcard to allow 3rd-party websites
to access the contents.

Rate limiting of any endpoint is implementation-dependent and not specified by this protocol.

### POST `/_cum/register`

- Content-Type: `application/x-www-form-urlencoded`
- Form body:
  - `username`: max 24 characters, alphanumeric ASCII and underscore, no whitespace
  - `password`: max 128 characters, alphanumeric ASCII, no whitespace
- Responds:
  - `201`: Account successfully created
  - `400`: Invalid form body

### GET `/_cum/*/**.xml`

- Content-Type: `application/xml`
- Headers:
  - `Authorization`: base64 encoded string `username:password`
  - `Range`: which part of the (decompressed) file to send, e.g. `bytes=0-499`. Offsets address the uncompressed content even though the response stream is returned gzip-compressed.
- Carries g-zipped data
- Responds:
  - `200`: Whole resource available and returned
  - `206`: Partial resource available and returned
  - `404`: If resource cannot be found
  - `416`: If range cannot be satisfied
  - `401`: Unauthorized to access this content
  
Application may wait on this endpoint with `Range` set only to the start.
This will allow the client to get notified about latest file change after
specified range start. The server holds the request for up to 120 seconds
(the default timeout) waiting for a change, after which it responds with the
current content.

### POST `/_cum/*/**.xml`

- Content-Type: `application/xml`
- Headers:
  - `Authorization`: base64 encoded string `username:password`
- Body:
  - An XML data to be appended to the specified file
- Responds:
  - `200`: Appended successfully
  - `404`: If resource cannot be found
  - `401`: Unauthorized to modify this content
  - `400`: Malformed
  
## File schema

### /info.xml

```xml
<?xml version="1.0" encoding="utf-8" ?>
<cum:info>
    <protocol>1.0.0</protocol>
    <responsible>any@example.com</responsible>
</cum:info>
```

- `protocol`: semver version of Comes Unified Messages protocol
- `responsible`: e-mail address of the host

### /rooms.xml

```xml
<?xml version="1.0" encoding="utf-8" ?>
<rooms>
    <room name="my_room" creation_time="unix_timestamp_utc" private="false" />
    <room name="csgo" creation_time="unix_timestamp_utc" private="true" />
</rooms>
```

This file may contain many `room` entries under `rooms` root.

Every `room` has:
- `name`: max 32 characters, alphanumeric ASCII and underscore, no whitespace.
- `creation_time`: unix timestamp in seconds (UTC).
- `private`: `true` or `false` value indicating if permission is required to interact with this room
- The `private` value is mirrored as `<private>` in the room's `info.xml`; the server keeps both in sync, and either may be treated as the source of truth.

### /users.xml

```xml
<?xml version="1.0" encoding="utf-8" ?>
<users>
    <user name="adam" password="argon2id_hash" creation_time="unix_timestamp_utc" />
</users>
```

A server-only file used to store users.

Every `user` has:
- `name`: max 24 characters, alphanumeric ASCII and underscore, no whitespace.
- `password`: Argon2ID hashed password. Password itself: see endpoint `/register`.
- `creation_time`: unix timestamp in seconds (UTC).

### /rooms/room_name/info.xml

```xml
<?xml version="1.0" encoding="utf-8" ?>
<cum:room>
    <name>room_name</name>
    <description>My awesome room</description>
    <private>true</private>
    <encryption>none</encryption>
    <operators>
        <user name="adam" />
    </operators>
    <authorized>
        <user name="adam" />
    </authorized>
</cum:room>
```

This file describes the room.

Every `<cum:room>` has:
- `name`: max 32 characters, alphanumeric ASCII and underscore, no whitespace.
- `description`: max 128 characters, UTF-8.
- `private`: `true` or `false`, whether room is public or invite-only.
- `encryption`: enum of `none`, `rot67` and `aes-256-gcm`
- `operators`: list of operators which can modify this room
- `authorized`: list of authorized users to read this room

You may send the following to the `/_cum/*/**.xml` (POST) endpoint:
- `<cum:room>` containing `<name>`, `<description>` and `<encryption>` to **create** a room. This creates the room directory together with its `info.xml` and an empty `messages.xml` if they do not yet exist; it never updates an existing room.
- `<authorized>` with a `<user>` inside to append a user to the authorized list.
- `<operators>` with a `<user>` inside to append a user to the operators list.

### /rooms/room_name/messages.xml

```xml
<?xml version="1.0" encoding="utf-8" ?>
<cum:messages>
    <message>
        <user name="adam" />
        <creation time="unix_timestamp_utc" />
        <attachments>
            <image url="https://" />
            <binary url="https://" />
        </attachments>
        <message>
            content
        </message>
    </message>
</cum:messages>
```

This file describes all messages in given room.

Every `<message>` has:
- `user`: for specifics reference previous schemas
- `creation`: for specifics reference previous schemas
- `attachments`: empty or full of:
  - `image` with `url`. Informs about image type attachments.
  - `binary` with `url`. Informs about binary type attachments.
- `message`: UTF-8 plaintext when `<encryption>` is `none`; otherwise bytes as Base64 (the encrypted payload).

You may send a `<message>` (containing its inner `<message>` content and `<attachments>`) to the `/_cum/*/**.xml` (POST) endpoint to append a new message at the end of the file.

## Encryption

Comes Unified Messages protocol specifies two types of encryption:
- `rot67` which is special case of caesar cipher with rotation of the alphabet by 67 places.
- `aes-256-gcm` which contains `Nonce(12 bytes) + Ciphertext + AuthTag(16 bytes)` as Base64. The 32-byte key is derived from the pre-shared password using Argon2id (memory cost 64 MiB, iterations 3, parallelism 4), using the room name encoded as UTF-8 as the salt.

For `aes-256-gcm`, users must securely exchange the pre-shared password out of band; the salt is derived from the room name, so no additional value needs to be shared.
