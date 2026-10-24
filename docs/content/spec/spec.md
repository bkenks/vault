# vault — Project Spec

Airtable, but for files. A flat pile of files on disk, each with an ID, and a SQLite database overlaying meaning on top of them.

Folders force every file into exactly one place. `personal/documents` and `business/documents` both exist, files often belong to both, and you only get one copy. vault drops the hierarchy: files live in one pile, and classification becomes rows you can query.

## Scope

This is v1 and it is deliberately tiny. Four commands, two columns, no tests, no integrity checks, no guards. Everything else gets added when it's actually needed.

## Principles

1. **The database describes files. It does not own them.** Finder, Obsidian, `mv`, and any editor stay free to touch files directly.
2. **Files are mutable and ordinary.** No read-only modes, no wrapped editors, no write protocol.
3. **Drift is acceptable.** The database reflects vault state accurately, but if a file changes underneath it, that's fine. Reconcile later, loosely.
4. **IDs solve identity, nothing else.** Two files named `invoice.pdf` need to be distinguishable. That's the whole job.
5. **Start small.** Features earn their way in.

## Layout

```plain
vault/
  .last          # results of the last find, for numbered references
  vault.db       # SQLite
  files/
    <id>/<original-filename>
  out/           # release target
```

Each file gets its own ID-named directory and keeps its original filename inside it. No sidecar metadata — the directory name is the ID, so the mapping is visible with `ls`.

## IDs

ULID: 48-bit millisecond timestamp plus 80 random bits, as 26 characters of Crockford base32. Chosen over random hex because string order equals creation order (so `ls files/` sorts oldest to newest) and the alphabet omits `I`, `L`, `O`, and `U`, which makes typed ID prefixes unambiguous.

## Schema

One table:

```sql
CREATE TABLE files (
  id   TEXT PRIMARY KEY,  -- ULID, matches files/<id>/
  name TEXT NOT NULL      -- original filename
);
```

That's it. A human-friendly name mapped to an ID. Any further structure — a `documents` table, a `domains` table, relations between them — is created by the user in SQL, when they want it.

## Commands

### `vault add [file...]`

Brings files into the vault.

- Copies each file to `files/<new-id>/<original-filename>`.
- Inserts a row into `files`.
- After all copies succeed, prompts to delete the originals.

Flags:

- `--all` — every file in the current working directory, non-recursive, directories skipped.
- `--dry-run` — print what would be added, change nothing.

### `vault find [query]`

Whatever you type gets wrapped. Which wrapper depends on whether the input starts with `SELECT`.

A fragment is treated as a `WHERE` clause:

```plain
vault find "name LIKE '%tax%'"
```

```sql
SELECT ROW_NUMBER() OVER () AS n, id, name
FROM files
WHERE name LIKE '%tax%'
```

A full statement is wrapped for numbering only:

```plain
vault find "SELECT f.id, f.name, d.slug FROM files f JOIN domains d ON ..."
```

```sql
SELECT ROW_NUMBER() OVER () AS n, q.* FROM ( <your query> ) q
```

No argument lists everything.

The fragment form always carries `n` and `id`, which is the point — numbered references and ULIDs come free without you asking for them. The full-statement form always carries `n`, but `id` only if you selected it, and numbered references won't resolve for rows without one.

Results print as a numbered table and save to `.last`. A ULID is 26 characters, so the table prints an 8-character prefix; `.last` holds the full ID.

This is why the project stays small: no query language, no filter syntax, no URI scheme. Structure and search both come from typing SQL.

### `vault open <id|n>`

Takes one argument, either a file ID or a row number from the last `find`. Hands the path to the macOS `open` command, which launches the default app.

### `vault release <id|n>`

Takes one argument, same resolution as `open`: check `.last` for a numbered match first, otherwise look up the ID in `files`. Copies the file to `out/<original-filename>`.

One file at a time. No flags.

## Implementation

- **Go**, single binary.
- **SQLite** via `modernc.org/sqlite` — pure Go, no cgo.
- `.last` as CSV, with the row number being the line number.

## Later

Not now, and not designed yet:

- User-created tables and relations (currently just: write the SQL yourself).
- Saved views — a named query, which is all an Airtable view really is.
- A reconcile command for when the pile and the database disagree.
- Multi-file `release`.
