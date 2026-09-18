---
Created: 2026-08-04
Type: Zettl
aliases:
References:
tags:
  - coding
  - work
---
# Supporting SMTP/IMAP Alongside the Gmail API

A guide to how SMTP/IMAP work — sending, backfill, sync, and message structure — and how to restructure a Gmail-API-based backend to support both provider types.

---

## The Fundamental Shift

Three mental-model changes coming from the Gmail API:

1. **SMTP and IMAP are two completely separate protocols with no shared state.** SMTP is only for submitting outbound mail; IMAP is only for reading and manipulating mailboxes. There's no unified "email API" — your provider abstraction will talk to two different servers, often with separate credentials/endpoints, and they don't know about each other.

2. **Everything is raw RFC 5322/MIME messages.** Gmail's API hands you parsed JSON with a `payload.parts` tree and separate `attachmentId`s. With IMAP, the canonical object is the raw byte stream of the message, and you (or a MIME library) parse it. Gmail's JSON structure is actually just a pre-parsed MIME tree — the mental model transfers, you just move the parsing into your own code.

3. **IMAP is folder-centric and stateful.** There's no account-wide message list and no account-wide change feed. You open a session, `SELECT` a mailbox (folder), and everything you do happens within that folder. Sync state is tracked per-folder, not per-account.

---

## Sending (SMTP)

### The protocol flow

- Connect to the provider's submission server: port **587 with STARTTLS**, or **465 with implicit TLS**.
- Authenticate: typically SASL PLAIN/LOGIN with a password or app password, or **XOAUTH2** for providers like Gmail/Microsoft that require OAuth.
- The exchange is essentially: `MAIL FROM`, `RCPT TO` for each recipient, then `DATA` followed by the complete raw message.

### You construct the entire MIME document yourself

Headers, multipart boundaries, base64-encoded attachments, inline images with `Content-ID` headers that your HTML references via `cid:` URLs. Every language has solid libraries for this (Python's `email` package, MimeKit in .NET, `nodemailer` in Node), but it's your responsibility now, whereas Gmail's API let you get away with structured fields for simple cases.

### Gotchas coming from the Gmail API

**Sent folder.** SMTP delivers the message but does *not* save a copy to the Sent folder. Gmail does that automatically even over SMTP, but most providers don't. The standard pattern: send via SMTP, then `APPEND` the same raw message to the Sent mailbox via IMAP with the `\Seen` flag. You need provider-specific knowledge (or heuristics) about whether to do this, or you'll get duplicates on Gmail and missing sent mail elsewhere.

**Message-ID and threading.** You generate the `Message-ID` header yourself. For replies, you must set `In-Reply-To` and `References` headers correctly — this is what threading is built on. There is **no `threadId` in the IMAP world**; threads are something *you* compute client-side from `References`/`In-Reply-To` chains (plus subject heuristics as a fallback). This is probably the single biggest backend restructuring item if your data model currently leans on Gmail's `threadId`.

**Delivery feedback.** SMTP gives a synchronous accept/reject at submission time, but that only means the server took responsibility. Actual delivery failures come back later as bounce messages (DSNs) to the return-path address — i.e., they show up in the inbox as email, which your sync pipeline will ingest like any other message. Also note SMTP has no idempotency: a retry after a timeout can double-send.

---

## Backfill (IMAP)

### Basics

- Connect on port **993** (TLS), authenticate.
- `LIST` to enumerate mailboxes; for each one, `SELECT` it and fetch messages.
- Within a selected folder, messages are addressed by **UID** — a per-folder, monotonically increasing identifier that's stable *as long as the folder's `UIDVALIDITY` value doesn't change*.

### The efficient two-phase fetch pattern

1. **Fetch cheap metadata for everything** — `UID FETCH 1:* (FLAGS INTERNALDATE RFC822.SIZE ENVELOPE BODYSTRUCTURE)` in batched ranges.
   - `ENVELOPE` gives you parsed from/to/subject/date.
   - `BODYSTRUCTURE` gives you the full MIME tree *as metadata* — part types, sizes, filenames, Content-IDs — without downloading any content.
2. **Fetch bodies selectively** — `BODY[]` for the whole raw message, or `BODY[1.2]` to pull just a specific MIME part by its section number. Pull text bodies immediately and defer or skip large attachments, fetching them lazily on demand (roughly analogous to Gmail's separate `attachmentId` fetch).

### Structural differences from Gmail

**Labels vs folders.** Gmail has labels (one message, many labels); IMAP has folders (a message *exists separately* in each folder it appears in). When you point IMAP at a Gmail account, labels appear as folders and the same message shows up as distinct copies with distinct UIDs in each. Cross-folder deduplication is done via the `Message-ID` header — usually reliable, but not guaranteed unique or present. Gmail's IMAP server offers proprietary extensions — `X-GM-MSGID`, `X-GM-THRID`, `X-GM-LABELS` — that restore global IDs and thread IDs; use them when detected.

**No global message ID.** In baseline IMAP, a message's identity is the tuple `(folder, UIDVALIDITY, UID)`. RFC 8474 (OBJECTID) adds stable global IDs, but server support is spotty.

**Connection limits.** Gmail allows around 15 concurrent IMAP connections per account; others allow far fewer. Backfill parallelism needs a per-account connection pool.

---

## Incremental Sync

This is where IMAP demands the most engineering — there's no equivalent of Gmail's `history.list` change feed.

### Baseline sync (no extensions)

Sync state per folder is `{UIDVALIDITY, highest UID seen}`:

- **New messages** — easy: `UID FETCH <lastUID+1>:* (...)`.
- **Flag changes** (read/unread, starred, etc.) — painful: baseline IMAP gives no "what changed" query, so you'd re-fetch `FLAGS` for every known UID and diff.
- **Deletions** — detected by comparing the server's UID set against yours.
- **`UIDVALIDITY` changed on `SELECT`** — happens when a server rebuilds a mailbox. Every UID you've cached for that folder is void; you must fully resync it. Your schema needs to treat this as a first-class event, not an error.

### Extensions that fix most of this

Supported by most serious servers (Dovecot, Gmail, Fastmail, Microsoft 365 in part):

- **CONDSTORE (RFC 7162)** — adds a `MODSEQ` counter. Store `HIGHESTMODSEQ` per folder and ask `UID FETCH 1:* (FLAGS) (CHANGEDSINCE <modseq>)` to get only changed messages.
- **QRESYNC (RFC 7162)** — extends CONDSTORE so a single `SELECT` with your saved state also returns which UIDs `VANISHED` (were deleted) since you last looked.

With both, incremental sync becomes nearly as cheap as Gmail's history API — just per-folder instead of per-account. Detect capabilities via the `CAPABILITY` command and keep a degraded polling path for servers without them.

### Real-time updates

**IDLE** is the push mechanism: a connection parked in IDLE on a folder gets notified of new mail and changes in *that folder only*. The standard production pattern is one connection IDLEing on INBOX plus periodic polling of other folders (with the connection-limit budget in mind).

There's no webhook equivalent to Gmail's Pub/Sub push — your architecture shifts from webhook-driven to holding **long-lived connections per account**, which has real implications for how you shard sync workers.

---

## Message Structure (MIME)

A raw message is headers plus a body, where the body is a tree of MIME parts. Common structures:

```
Simple:            text/plain

HTML email:        multipart/alternative
                   ├── text/plain            (fallback)
                   └── text/html

Inline images:     multipart/related
                   ├── multipart/alternative
                   │   ├── text/plain
                   │   └── text/html          (contains <img src="cid:logo123">)
                   └── image/png              (Content-ID: <logo123>, disposition: inline)

The works:         multipart/mixed
                   ├── multipart/related
                   │   ├── multipart/alternative
                   │   │   ├── text/plain
                   │   │   └── text/html
                   │   └── image/jpeg          (inline, Content-ID)
                   ├── application/pdf         (Content-Disposition: attachment)
                   └── message/rfc822          (a forwarded email, itself a full message)
```

### Classification rules

- `multipart/alternative` — "same content, pick the richest you can render."
- `multipart/related` — bundles a body with resources it references by `cid:`.
- `multipart/mixed` — the container that separates body from attachments.
- A part is an **inline image** if it has a `Content-ID` referenced by the HTML (and usually `Content-Disposition: inline`).
- A part is an **attachment** if disposition is `attachment`, or it has a filename and isn't cid-referenced.

This is exactly the same tree Gmail gives you in `payload.parts` — Gmail just pre-walks it and hands attachments out by ID. Your IMAP normalizer will walk `BODYSTRUCTURE`, classify parts into **{best body, inline assets keyed by Content-ID, attachments}**, and can then present the identical shape your system already uses for Gmail. That's the crux of supporting both: the canonical internal representation you already have is reachable from raw MIME; you're just writing the parser Gmail was running for you.

### Real-world mess to budget for

- Transfer encodings (base64, quoted-printable) and arbitrary charsets to decode.
- RFC 2047 encoded-words in headers (`=?UTF-8?B?...?=`) and RFC 2231 encoded filenames.
- HTML-only messages with no plain part, and vice versa.
- Outlook's `winmail.dat` (TNEF) blobs.
- `multipart/signed` S/MIME wrappers.
- Plenty of outright malformed MIME that strict parsers choke on.

Use a battle-tested MIME library and treat parsing failures as expected inputs, not exceptions.

---

## Restructuring the Backend

The abstraction that works in practice: a **provider-agnostic canonical model with per-provider adapters**. Design these seams deliberately:

### Identity

Canonical message ID must not assume Gmail's global ID. For IMAP it's derived from `(folder, UIDVALIDITY, UID)` with `Message-ID`-based dedup across folders, upgraded to `X-GM-MSGID` or OBJECTID when available.

### Threads

Make threading a service you compute (`References`/`In-Reply-To` graph + subject fallback), and treat Gmail's `threadId` as one input to it rather than the source of truth. This keeps thread semantics consistent across providers.

### Labels vs folders

Decide on one internal concept (most systems pick "labels") and have the IMAP adapter translate: applying a label = `COPY`/`MOVE` between folders, and multi-label messages on plain IMAP either aren't supported or mean multiple copies.

### Change feed

Define an internal event stream — *message added / flag changed / message removed / folder invalidated* — and have Gmail's history API and IMAP's QRESYNC/polling both emit into it. The "UIDVALIDITY changed, folder needs full resync" event has no Gmail analog, so add it now.

### Send pipeline

Compose raw MIME once for both providers (Gmail's API happily accepts raw RFC 5322 too, which lets you unify composition), then provider-specific submission plus the conditional sent-folder `APPEND`.

Drafts are just messages APPENDed to the Drafts folder with the `\Draft` flag — there's no draft object with an update method; editing a draft means append-new-then-delete-old.

### Auth

Support both password/app-password and XOAUTH2, because Gmail and Microsoft 365 have disabled basic auth for IMAP, while generic providers still use passwords.

---

## Footnote: JMAP

JMAP (RFC 8620/8621) is the modern JSON protocol designed to replace exactly these pain points and looks much more like the Gmail API — but adoption is basically Fastmail and a few others, so IMAP/SMTP remains the compatibility target.

---

## Quick Reference: Gmail API → IMAP/SMTP Mapping

| Gmail API concept             | IMAP/SMTP equivalent                                   |
| ----------------------------- | ------------------------------------------------------ |
| `messages.send`               | SMTP submission + conditional IMAP `APPEND` to Sent    |
| `messages.list` / backfill    | `LIST` folders → `SELECT` → `UID FETCH` in ranges      |
| `history.list` change feed    | CONDSTORE/QRESYNC per folder (or flag-diff polling)    |
| Pub/Sub push notifications    | IMAP `IDLE` (per-folder, long-lived connection)        |
| Global message `id`           | `(folder, UIDVALIDITY, UID)` + `Message-ID` dedup      |
| `threadId`                    | Client-side threading from `References`/`In-Reply-To`  |
| Labels                        | Folders (message copied per folder)                    |
| `payload.parts` (parsed MIME) | `BODYSTRUCTURE` metadata + your own MIME parser        |
| `attachmentId` lazy fetch     | `BODY[section]` partial fetch                          |
| Drafts resource with update   | `APPEND` with `\Draft`; edit = append new + delete old |
