# Specification

Details about the logic of the original [timer.mob.sh service][timer-repo] and [CLI tool][mob-repo].
These allow to implement an alternative version while still being compatible with the original UI
and mob CLI tool.

The idea is to re-implement the whole toolset step-by-step, starting with the backend, then UI
frontend and finally the mob CLI tool itself.

[timer-repo]: https://github.com/remotemobprogramming/timer
[mob-repo]: https://github.com/remotemobprogramming/mob

- [Backend](#backend)
  - [General notes](#general-notes)
  - [Routing](#routing)
    - [`GET /` index page](#get--index-page)
    - [`GET /events` SSE endpoint for the index page](#get-events-sse-endpoint-for-the-index-page)
    - [`GET /help` help page](#get-help-help-page)
    - [`POST /` join a new room](#post--join-a-new-room)
    - [`GET /<room>` room page](#get-room-room-page)
    - [`PUT /<room>` setting timers and breaks](#put-room-setting-timers-and-breaks)
    - [`GET /<room>/events` SSE endpoint for rooms](#get-roomevents-sse-endpoint-for-rooms)
  - [Rome name generation](#rome-name-generation)
  - [State management](#state-management)
    - [Cleanup procedure](#cleanup-procedure)
    - [Per room state](#per-room-state)
- [Frontend](#frontend)
- [Mob CLI tool](#mob-cli-tool)

## Backend

This section describes the details of the backend, with all details required to create an
alternative server for the UI and mob CLI.

### General notes

Anything noteworthy that applies to the whole backend system.

- Uses UTC throughout, local time zone adjustments are handled by the UI.
- The UI is provided by the backend a set of HTML templates.

### Routing

All routes required to make the backend API work with the UI and the mob CLI.

#### `GET /` index page

The main index page, a template with 3 fields. Original file can be found **[here](https://github.com/remotemobprogramming/timer/blob/ff33cc249cc0ab096e5869f5bb2927ae6d35533c/src/main/resources/templates/index.html)**.

The template fields are:

- `numberOfConnections`: Current active client connections.
- `url`: Current URL of the request.
- `randomRoomName`: Randomly generated new room name.

#### `GET /events` SSE endpoint for the index page

Used on the index page to show currently active users and timers.

- Emits `ACTIVE_USERS_UPDATE` every second, data is an `u64`.
- Emits `ACTIVE_TIMERS_UPDATE` every second, data is an `u64`.

#### `GET /help` help page

Just a simple help page with no interactive functionality. The page is a static HTML file. Original
file can be found **[here](https://github.com/remotemobprogramming/timer/blob/ff33cc249cc0ab096e5869f5bb2927ae6d35533c/src/main/resources/templates/help.html)**.

#### `POST /` join a new room

Used to join a new room with the name from the index page.

- Passes the room name as form data (for example `room=bright-mink-68`)
- Redirects to the room at `/<room>`.

#### `GET /<room>` room page

The page for a specific room, a template with 1 field. Original file can be found **[here](https://github.com/remotemobprogramming/timer/blob/ff33cc249cc0ab096e5869f5bb2927ae6d35533c/src/main/resources/templates/room.html)**.

The template fields are:

- `room`: Data structure for the room containing:
  - `name`: Room name/id.

#### `PUT /<room>` setting timers and breaks

Used to set a timer or break for the given room. Can be triggered through UI as well as the mob CLI.

- The timer type is determined by the field name, either `timer` or `breaktimer`. The value is
  specified in absolute minutes.
- The `user` field is empty when used from the UI but set to the **Git username** when used from
  the mob CLI tool.
- Valid timer range is `0..=1440` minutes.

Example setting a timer for 10 minutes:

```json
{
    "timer": 10,
    "user": ""
}
```

Example setting a break for 15 minutes:

```json
{
    "breaktimer": 15,
    "user": ""
}
```

#### `GET /<room>/events` SSE endpoint for rooms

Use on the room page to show the history and current timer or break time.

- Emits `INITIAL_HISTORY` once in the beginning (without the latest entry).
- Emits `TIMER_REQUEST` whenever an update occurs (also once on page load).
- Emits `KEEP_ALIVE` every **5 seconds** with an empty timer request payload.
- Events seem to be throttled to **1/s**, if hitting break or timer buttons fast, events are
  not coming up as fast and seem to be dropped (on frontend or backend is unclear).

All events emit timer requests which are JSON objects with the following structure:

- `timer` defines the absolute minutes to count down.
- `requested` is an RFC3339 formatted timestamp of when the request was triggered (server side).
- `user` is empty when triggered from the UI, filled when triggered from the mob CLI.
- `nextUser` is the potential next user to be typist.
- `type` is either `TIMER` or `BREAKTIMER`.
  - `TIMER` is a normal countdown timer for the current session.
  - `BREAKTIMER` is a countdown timer as well, but for the break time, not working time.
- All fields except `timer` can be `null` (happens when first entering the room).
  - Technically `timer` can be `null` as well, but is only used for `KEEP_ALIVE` events where the
    data payload is ignored.

Example of a `INITIAL_HISTORY` event payload:

```json
[
    {
        "timer": 15,
        "requested": "2022-01-05T02:27:12.192312Z",
        "user": "",
        "nextUser": null,
        "type": "TIMER"
    },
    {
        "timer": 15,
        "requested": "2022-01-05T02:49:34.133961Z",
        "user": "",
        "nextUser": null,
        "type": "BREAKTIMER"
    }
]
```

Example of a `TIMER_REQUEST` event payload:

```json
{
    "timer": 15,
    "requested": "2022-01-05T02:27:12.192312Z",
    "user": "",
    "nextUser": null,
    "type": "TIMER"
}
```

### Rome name generation

The index page generates random room names when loaded. These auto-generated names have a specific
format that seems to have the goal of being easily memorable and sharable.

- The format is `<adjective>-<animal>-<number>`.
- `adjective` and `animal` are random words picked from pre-defined lists.
- `number` is in the range `10..100`, thus always 2 digits long.
- The room name in API requests is validated against `[a-z0-9-]+`.

### State management

The server keeps track of the active rooms and connections and does certain "householding" tasks
like cleaning up of old history items. This is described as state management.

- All rooms are kept in an in-memory map, reboots don't preserve the state.
- A cleanup run happens every minute, removing old history entries.

#### Cleanup procedure

The following notes describe the details of how the cleanup of the state is done. This is required
to not let the memory usage of the server endlessly grow.

- Entries are considered old if they were created more that 24 hours ago.
- If the room's history becomes empty, an empty timer request is published again.
- Room database seems to be never cleared?

#### Per room state

For each room that is added to the state, additional details about the room are kept in memory, in
addition to just the room name. The following details are managed for each individual room:

- The room's name/id (again in addition to the repository).
  - This seems to be only a detail of the original server, where the structure that manages a room's
    state is passed into templates as well.
- List of previous timer requests, building the history for users that join later and sent over a
  room's event stream on joining.
- The next user is determined by traversing the history, old to new, until a name is found
  that's not the current user (whoever sent the new request).

## Frontend

TODO: write down the details of re-implementing the frontend (for example in WASM).

## Mob CLI tool

TODO: write down all features and details of the mob CLI tool to re-implement it in another
langauge.
