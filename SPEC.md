# soksak-issue-board-spec

An **issue board**: a surface where issues are visible as cards, each carrying a title, a state, and
the detail a human needs to judge it at a glance.

This contract exists so that a producer of issue state (a ledger, a tracker, a loop) can put its
issues on a board **without naming the board**. The producer pins this contract; the board declares
it implements it. Either side can be replaced without touching the other — which is the whole point,
and the reason the producer must never pin the implementer's plugin id.

## Discovery

An implementer declares the contract in its manifest:

```json
{ "implements": ["soksak-issue-board-spec"] }
```

A consumer discovers implementers by contract id alone:

```
sok plugin.implementers '{"contract":"soksak-issue-board-spec"}'
```

and addresses whichever it finds as `plugin.<discovered id>.<command>`. A consumer that hard-codes
an implementer's id has not implemented this contract; it has merely used one board.

**No implementer is a legal state.** A board is a projection, never a source of truth: the consumer
must keep working, unprojected, when nothing implements the contract. A consumer that fails without
a board has made the board load-bearing, which this contract forbids.

## The card

A card is the projection of one issue. The producer owns the issue; the board owns the card. The
board never writes back — a card is a view of a fact, not the fact.

| Field | Meaning |
|---|---|
| `title` | What the issue is, as a human names it. |
| `description` | The detail a human judges it by (who holds it, what evidence it carries). |
| `status` | Where it stands. |

### status

An implementer supports at least these, and means by them exactly this:

| status | Meaning |
|---|---|
| `backlog` | Known, and nobody is working on it. |
| `inprogress` | Someone is working on it right now. |
| `done` | Finished, on whatever terms the producer requires. |

An implementer MAY offer more (`todo`, `review`, …). A consumer MUST NOT require them.

## Commands

An implementer exposes these. Names and shapes are the contract; everything else about the board —
its layout, its views, its extra fields — is the implementer's own business.

### `node.add`

```
node.add { title: string, description?: string, status?: status } → { nodeId: string }
```

Creates a card and returns the id the board knows it by. The id is the **board's** — a consumer must
not construct or predict it, and must not assume it is stable across boards.

### `node.edit`

```
node.edit { id: nodeId, title?: string, description?: string, status?: status } → { ok: true }
```

Updates a card in place. Omitted fields are left alone.

### `node.get`

```
node.get { id: nodeId } → { node } | refusal
```

Reads a card back. A refusal means the board no longer has it — a human deleted it, or the board was
reset. The consumer must treat that as "no card" and be able to make a new one.

## The mapping is the consumer's

The board issues the id; the consumer must remember which issue it belongs to. A consumer that
cannot answer "which card is this issue?" without searching the board by title is not projecting,
it is guessing — two issues may share a title, and a title may be edited on the board.

So: **the consumer stores `issue → nodeId` in its own store**, and re-projects through that mapping.
When `node.get` says the card is gone, the mapping is stale: drop it and project a fresh card.

## What an implementer must not do

- **Never write back to the producer.** A board that edits the issue it was given has inverted the
  contract; the producer's store is the truth and the card is downstream of it.
- **Never require the consumer to know the board.** No implementer-specific field may be mandatory.
