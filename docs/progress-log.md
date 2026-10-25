# Progress log

One row per working session. Newest at the bottom. Keep each cell short; the details go in concept notes.

| Date | Week/Day | What I did | Concepts learned | Blockers | Next |
|---|---|---|---|---|---|
| _EXAMPLE_ 2026-10-26 | W1 / D1 | Read RESP2 spec; built `RespParser` for arrays of bulk strings; 6 tests | RESP framing; TCP is a byte stream, so one `read()` ≠ one message | Parser hung on partial input, so the reads need a loop | Handle partial reads; wire `PING` to `redis-cli` |
