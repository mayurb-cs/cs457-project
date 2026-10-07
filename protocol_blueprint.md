# CS 457 Sprint 1 — Application Protocol Blueprint

**Student Name:** Mayur Bhat

**Date:** 2026-10-06

**Course:** CS 457 - Computer Networks

**Game:** Co-op Hangman (2 players)

**Target Server:** `server.mayur.edu` : TCP port `5050`


## 1. Transport

| Item | Decision |
|---|---|
| Transport | TCP |
| Port | `5050` |
| Serialization | UTF-8 JSON, one object per line |
| Framing | Newline delimited |
| Max frame size | 4096 bytes including the `\n` |
| Protocol version | `1` |
| Authority | The server is authoritative for all game state. Clients only send intent (`GUESS`, `REMATCH`). The server will validate and broadcast the result. |

### Game rules
- The server picks one shared hidden word per round. Both players work on the same word as a team.
- Turns alternate between Player 1 and Player 2. The server chooses who goes first at random.
- On their turn, a player guesses either a single letter or the full word.
  - Letter in the word -> every position of that letter is revealed to both players.
  - Letter not in the word, or wrong full-word guess -> +1 strike (strikes are shared).
  - Correct full-word guess, or all letters revealed -> team wins.
  - 6 strikes -> team loses.
- Every completed turn (including a timed-out turn) increments `turn_count`. The team's score is the number of turns used; lower is better.
- A turn times out after 30 seconds. A timeout passes the turn and counts as a turn, but does not add a strike.
- Out-of-turn guesses never change game state. The server discards the move and replies to the sender with `ERROR` code `NOT_YOUR_TURN`.


## 2. Message Framing Example

```text
{"type":"LOBBY_WAIT","v":1,"players":["alice","bob"],"needed":0}\n{"type":"GAME_START","v":1,"round":1,"word_length":7,"max_strikes":6,"turn_timeout":30,"players":[{"id":1,"username":"alice"},{"id":2,"username":"bob"}],"first_turn":2}\n{"type":"STATE_UPDATE","v":1,"masked":"_______","guessed":[],"wrong":[],"strikes":0,"turn_count":0,"current_turn":2,"last_move":null}\n
```

## 3. Message Envelope

| Field | Type | Required | Description |
|---|---|---|---|
| `type` | string | yes | Message type name |
| `v` | integer | yes | Protocol version |


## 4. Message Type Definitions

| # | Type | Direction | Target server state | Description |
|---|---|---|---|---|
| 1 | `JOIN` | C -> S | LOBBY | Request to join with a username |
| 2 | `WELCOME` | S -> C | LOBBY | Join accepted, assigns player ID |
| 3 | `LOBBY_WAIT` | S -> C | LOBBY | Lobby roster, players still needed |
| 4 | `GAME_START` | S -> C | GAME_START | New round begins, word length, turn order |
| 5 | `GUESS` | C -> S | PLAYER_TURN | Guess a letter or the whole word |
| 6 | `STATE_UPDATE` | S -> C (broadcast) | PLAYER_TURN, CHECK_GAME_END | Authoritative board snapshot |
| 7 | `GAME_OVER` | S -> C (broadcast) | GAME_OVER | Round result and final score |
| 8 | `REMATCH` | C -> S | CLEANUP | Vote to play another round |
| 9 | `ERROR` | S -> C | any | Request rejected, with reason |
| 10 | `PING` / `PONG` | C <-> S | any | Liveness heartbeat |
| 11 | `DISCONNECT` | C <-> S | any | Graceful leave or server shutdown |


### 4.1 `JOIN` (Client -> Server)

| Field | Type | Description |
|---|---|---|
| `username` | string | 1–16 chars |


### 4.2 `WELCOME` (Server -> Client)

| Field | Type | Description |
|---|---|---|
| `player_id` | integer | `1` or `2` |
| `username` | string | Accepted username |


### 4.3 `LOBBY_WAIT` (Server -> Client, broadcast to lobby)

| Field | Type | Description |
|---|---|---|
| `players` | array of string | Usernames currently in the lobby |
| `needed` | integer | Players still required to start |


### 4.4 `GAME_START` (Server -> Client, broadcast)

| Field | Type | Description |
|---|---|---|
| `round` | integer | Round number |
| `word_length` | integer | Number of letters in the hidden word |
| `max_strikes` | integer | Strikes allowed before losing |
| `turn_timeout` | integer | Seconds per turn |
| `players` | array of object | `{ "id": int, "username": string }` for both players |
| `first_turn` | integer | `player_id` who guesses first |


### 4.5 `GUESS` (Client -> Server)

| Field | Type | Description |
|---|---|---|
| `seq` | integer | Client-chosen counter, increases by 1 per guess. |
| `kind` | string | `"letter"` or `"word"` |
| `value` | string | `letter`: exactly 1 char `a`–`z`. `word`: only `a`–`z`, length equal to `word_length`. |


### 4.6 `STATE_UPDATE` (Server -> Client, broadcast to both players)

| Field | Type | Description |
|---|---|---|
| `masked` | string | Word with unrevealed letters as `_` |
| `guessed` | array of string | Correct letters guessed so far |
| `wrong` | array of string | Letters and words that were wrong |
| `strikes` | integer | Strikes used |
| `turn_count` | integer | Completed turns this round |
| `current_turn` | integer | `player_id` whose turn it is now |
| `last_move` | object or null | see below |

`last_move` object:

| Field | Type | Description |
|---|---|---|
| `player_id` | integer | Who acted |
| `kind` | string | `"letter"`, `"word"`, or `"timeout"` |
| `value` | string or null | The guess |
| `correct` | boolean | Whether the guess hit |


### 4.7 `GAME_OVER` (Server -> Client, broadcast)

| Field | Type | Description |
|---|---|---|
| `result` | string | `"WIN"`, `"LOSE"`, or `"ABORTED"` |
| `reason` | string | `"WORD_GUESSED"`, `"ALL_LETTERS_REVEALED"`, `"MAX_STRIKES"`, `"PLAYER_DISCONNECTED"`, `"PLAYER_QUIT"` |
| `word` | string | The hidden word, revealed |
| `turn_count` | integer | Turns used |
| `strikes` | integer | Strikes used |
| `solved_by` | integer or null | `player_id` who made the winning guess or `null` |
| `rematch_timeout` | integer | Seconds to send `REMATCH`, `0` if a rematch isn't possible (`ABORTED`) |


### 4.8 `ERROR` (Server -> Client)

| Field | Type | Description |
|---|---|---|
| `code` | string | Machine-readable code |
| `message` | string | Human-readable text the client can display |
| `ref_seq` | integer or null | `seq` of the rejected `GUESS` if applicable |
| `fatal` | boolean | `true` if the server will close the connection after sending |


| Code | Fatal | When |
|---|---|---|
| `LOBBY_FULL` | yes | `JOIN` while 2 players are already seated |
| `USERNAME_TAKEN` | yes | `JOIN` with a name already in the lobby |
| `INVALID_USERNAME` | yes | Username fails length or character rules |
| `NOT_YOUR_TURN` | no | `GUESS` from the player who isn't `current_turn` |
| `INVALID_GUESS` | no | Malformed letter or word |
| `ALREADY_GUESSED` | no | Letter was guessed earlier this round |
| `WRONG_STATE` | no | Message not valid in the current state |
| `BAD_MESSAGE` | no | Frame isn't valid UTF-8 JSON |
| `UNKNOWN_TYPE` | no | `type` isn't defined |
| `UNSUPPORTED_VERSION` | yes | `v` does not match |
| `FRAME_TOO_LARGE` | yes | No `\n` within 4096 bytes |


### 4.9 `DISCONNECT` (Client <-> Server)

| Field | Type | Description |
|---|---|---|
| `reason` | string | `"QUIT"`, `"IDLE"` (server dropped an inactive player), `"SERVER_SHUTDOWN"` |
