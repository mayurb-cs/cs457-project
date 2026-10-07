# CS 457 Sprint 1 — Game State Machine Specification

## 1. Server FSM Diagram

```mermaid
stateDiagram-v2
    direction TB

    [*] --> INIT
    INIT --> LOBBY : bind and listen on TCP 5050

    LOBBY --> LOBBY : JOIN accepted (1 of 2) or player leaves, broadcast LOBBY_WAIT
    LOBBY --> LOBBY : JOIN rejected, send ERROR LOBBY_FULL or USERNAME_TAKEN
    LOBBY --> GAME_START : second JOIN accepted (2 of 2)

    GAME_START --> PLAYER_TURN : pick hidden word, assign P1 and P2, pick first turn, send GAME_START
    GAME_START --> GAME_OVER : player disconnects (ABORTED)

    PLAYER_TURN --> EVALUATE_MOVE : GUESS from current player
    PLAYER_TURN --> PLAYER_TURN : GUESS out of turn, send ERROR NOT_YOUR_TURN
    PLAYER_TURN --> CHECK_GAME_END : 30s turn timer expires, turn skipped
    PLAYER_TURN --> GAME_OVER : player disconnects or sends DISCONNECT (ABORTED)

    EVALUATE_MOVE --> PLAYER_TURN : invalid or repeated guess, send ERROR, same player retries
    EVALUATE_MOVE --> CHECK_GAME_END : valid guess, update letters, strikes, turn count

    CHECK_GAME_END --> PLAYER_TURN : round continues, switch turn, broadcast STATE_UPDATE
    CHECK_GAME_END --> GAME_OVER : word solved (WIN) or 6 strikes (LOSE)

    GAME_OVER --> CLEANUP : broadcast GAME_OVER, start 30s rematch vote

    CLEANUP --> GAME_START : both players send REMATCH, new word, round + 1
    CLEANUP --> LOBBY : a player quits, disconnects, or vote times out
    CLEANUP --> [*] : server shutdown, send DISCONNECT to all
```

### Server state descriptions

| State | Description |
|---|---|
| INIT | Create listening socket, set `SO_REUSEADDR`, load word list |
| LOBBY | Accept connections and `JOIN`s until 2 players are seated |
| GAME_START | Choose word, assign Player 1 / Player 2, randomly choose first turn, broadcast `GAME_START` + initial `STATE_UPDATE` |
| PLAYER_TURN | Wait for `GUESS` from `current_turn` with a 30 s timer. Out-of-turn guesses get `ERROR NOT_YOUR_TURN` and change nothing |
| EVALUATE_MOVE | Validate guess format and repeats. Apply letter/word guess, add strike on miss |
| CHECK_GAME_END | Check solved / 6 strikes. Otherwise flip `current_turn` and broadcast `STATE_UPDATE` |
| GAME_OVER | Broadcast `GAME_OVER` with result, revealed word, and turn count |
| CLEANUP | Collect `REMATCH` votes. Drop non-voters, return survivors to LOBBY, or start a new round |

**Disconnects:** In co-op, a round can't continue with one player. A disconnect during any in-game state ends the round as `ABORTED` and the remaining player is returned to LOBBY after `GAME_OVER`. A disconnect in LOBBY just frees the seat.

## 2. Client-Side FSM

```mermaid
stateDiagram-v2
    direction TB

    [*] --> INIT
    INIT --> CONNECTING : user enters server address and username

    CONNECTING --> INIT : DNS failure, connection refused, ERROR LOBBY_FULL or USERNAME_TAKEN
    CONNECTING --> LOBBY : WELCOME received

    LOBBY --> LOBBY : LOBBY_WAIT received, show roster
    LOBBY --> GAME_START : GAME_START received
    LOBBY --> INIT : connection lost

    GAME_START --> WAITING : draw board and scoreboard

    WAITING --> WAITING : STATE_UPDATE, teammate's turn, redraw board
    WAITING --> MY_TURN : STATE_UPDATE with current_turn equal to my player_id
    WAITING --> GAME_OVER : GAME_OVER received
    WAITING --> INIT : connection lost (EOF, reset, or timeout)

    MY_TURN --> MY_TURN : local validation fails or ERROR INVALID_GUESS, prompt again
    MY_TURN --> WAITING : send GUESS to server
    MY_TURN --> WAITING : STATE_UPDATE shows my turn timed out
    MY_TURN --> GAME_OVER : GAME_OVER received (teammate disconnected)
    MY_TURN --> INIT : connection lost

    GAME_OVER --> REMATCH_WAIT : user chooses rematch, send REMATCH
    GAME_OVER --> LOBBY : ABORTED result, LOBBY_WAIT received
    GAME_OVER --> [*] : user quits, send DISCONNECT

    REMATCH_WAIT --> GAME_START : GAME_START received
    REMATCH_WAIT --> LOBBY : LOBBY_WAIT received, teammate left
    REMATCH_WAIT --> INIT : connection lost or DISCONNECT IDLE
```