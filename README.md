# mini-redis-java

A Redis-like key-value server built in **pure Java 21**: no frameworks and no libraries beyond JUnit.
The goal isn't to replace Redis. It's to understand what Redis does: sockets, protocols, concurrency,
expiry, non-blocking I/O, persistence and JVM behaviour. Each of those gets built by hand, broken on
purpose, and then fixed.

## Goals

- Speak enough of the RESP protocol that the real `redis-cli` can talk to this server.
- Learn each concept from the inside: build it, see it fail under stress, explain why it failed.
- Leave a written record (concept notes and ADRs) that can be reviewed later for interviews.
- Keep the work steady and shipped in small pieces: every change goes through an issue and a PR.

## Learning method

Every concept goes through the same loop:

| Step | What it means here |
|---|---|
| **Learn** | Read the spec, docs or source. Write down the 2–3 ideas that matter. |
| **Build** | Implement the smallest working version, with tests. |
| **Break** | Make it fail on purpose: load, races, kill -9, bad input, tiny heap. |
| **Fix** | Fix the cause, not the symptom. Add a test that would have caught it. |
| **Document** | Write a concept note in [`docs/concepts/`](docs/concepts/) (and an ADR if a design choice was made). |
| **Commit** | Small Conventional Commits, PR, merge. |

A concept isn't done until its "Break it" section has a real failure in it.

## Daily workflow

1. **Pick an issue** on GitHub (one roadmap task ≈ one issue).
2. **Branch** from up-to-date `main`:
   ```bash
   git switch main && git pull
   git switch -c feature/resp-parser
   ```
3. **Build + test** until `mvn -B verify` is green.
4. **Break it.** Run the week's "break it" exercise and write down exactly what happened.
5. **Concept note.** Copy [`docs/concepts/TEMPLATE.md`](docs/concepts/TEMPLATE.md) and fill it in.
6. **Progress log.** Add a row to [`docs/progress-log.md`](docs/progress-log.md).
7. **PR.** Push, open a PR that references the issue (`Closes #12`), and let CI pass.
8. **Merge** (squash), delete the branch, and tick the box in [`docs/roadmap.md`](docs/roadmap.md).

## Git conventions

**Branches.** `main` is protected: no direct commits to it, and everything comes in through a PR.

| Prefix | Use for | Example |
|---|---|---|
| `feature/` | New behaviour | `feature/incr-command` |
| `fix/` | Bug fixes | `fix/resp-bulk-string-length` |
| `docs/` | Docs only | `docs/concept-virtual-threads` |
| `chore/` | Build, CI, tooling | `chore/bump-surefire` |

**Commits** follow [Conventional Commits](https://www.conventionalcommits.org/):

```
<type>(<optional scope>): <imperative summary>

feat(resp): parse bulk strings
fix(store): make INCR atomic under concurrent clients
test(server): reproduce lost updates with 100 clients
docs(concepts): add note on ConcurrentHashMap internals
refactor(command): model commands as sealed interface
chore(ci): cache Maven dependencies
```

Types: `feat`, `fix`, `test`, `docs`, `refactor`, `perf`, `chore`, `build`, `ci`.

## Tech stack

- Java 21 (records, sealed interfaces, pattern matching, virtual threads)
- Maven 3.9+
- JUnit 5
- GitHub Actions CI (`mvn -B verify` on every push and PR)
- Tools used along the way: `redis-cli`, `redis-benchmark`, JFR, JDK Mission Control, VisualVM

## Build, run, test

```bash
# Build and run all tests (same command CI runs)
mvn -B verify

# Run the server
java -cp target/classes com.sumit.miniredis.Main

# Talk to it (once Week 1 is done; default port 6379)
redis-cli -p 6379 PING
```

## Folder structure

```
mini-redis-java/
├── .github/workflows/ci.yml   # CI: mvn -B verify on Java 21
├── docs/
│   ├── roadmap.md             # 6-week plan with checkboxes
│   ├── progress-log.md        # daily log
│   ├── concepts/              # one note per concept (Learn → Break → Fix)
│   └── adr/                   # architecture decision records
├── src/
│   ├── main/java/com/sumit/miniredis/
│   └── test/java/com/sumit/miniredis/
├── pom.xml
└── LICENSE
```

## Docs

- [Roadmap](docs/roadmap.md): what to learn and build, week by week
- [Progress log](docs/progress-log.md): what actually happened, day by day
- [Concept notes](docs/concepts/): deep-dive notes, one per concept
- [ADRs](docs/adr/): why the project is built the way it is

## Status

**Week 1 – not started**

## License

[MIT](LICENSE)
