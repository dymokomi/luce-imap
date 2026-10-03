# luce-imap

An IMAP client in [luce-base](https://github.com/dymokomi/luce-base). It speaks
IMAP4rev1 (RFC 3501) and IMAP4rev2 (RFC 9051), with the extensions mail clients
use: IDLE, MOVE, UIDPLUS, LITERAL+, SASL-IR, SPECIAL-USE, LIST-EXTENDED, ESEARCH,
CONDSTORE (HIGHESTMODSEQ, MODSEQ, CHANGEDSINCE), ENABLE and UTF8=ACCEPT.
The connection is a [luce-tls](https://github.com/dymokomi/luce-tls) `Stream`:
TLS from the start (993), STARTTLS (143), or plain for a local test server.

```luce-base
from luce_imap import imap

var session = try imap.Session.open("imap.example.com", 993, imap.Security.tls)
defer session.close()
try session.login("alice@example.com", password)        # SASL PLAIN or LOGIN
for mailbox in try session.list():                       # with \Sent, \Trash, … roles
    print(mailbox.name)                                   # UTF-8, decoded from modified UTF-7
let state = try session.select("INBOX")                  # exists, uid_validity, uid_next, …
for record in try session.fetch_headers("1:*", "DATE FROM SUBJECT"):
    use(record.uid, record.flags, record.header)          # header bytes for luce-mime
let raw = try session.fetch_message(uid)                 # the whole message, unread
try session.store("7,9", imap.Flags.seen, true)
try session.move("9", "Archive")
if try session.idle(29 * 60 * 1000000000):               # wait for news
    apply(session.expunges(), session.flag_changes(), session.selected.exists)
    session.take_events()
session.logout()
```

Each command's results (mailboxes, fetched records, search hits) are views that
stay valid until the next command. Untagged news that arrives during any command
is kept as events: new EXISTS counts, expunged sequence numbers and flag changes.
Every read and write has a deadline (the session's `timeout`) and can be cancelled
from another thread through a `sync.Cancellation`, so a sync worker can wait in
IDLE and still quit at once. The session never retries. After a failure the
caller reconnects.

| Session | Does |
| --- | --- |
| `open(host, port, security, timeout, cancellation, trust)`, `over(link)` | connect, greet, STARTTLS, capabilities |
| `has(capability)`, `is_secure()`, `text()` | what the server offers; its last error text |
| `login(user, password)`, `login_oauth(user, token)` | SASL PLAIN / LOGIN; SASL XOAUTH2 |
| `list(reference, pattern)`, `status(name)`, `create`, `delete`, `rename` | mailboxes |
| `select(name, read_only)`, `unselect()` | the selected mailbox's state in `selected` |
| `fetch(set, items, changed_since)`, `fetch_headers(set, fields)`, `fetch_message(uid)` | messages by UID |
| `search(criteria)`, `search_text(text)` | UIDs, ascending; text search sends UTF-8 as needed |
| `store(set, flags, add)`, `copy(set, name)`, `move(set, name)`, `expunge(set)` | changes by UID |
| `append(name, data, flags)` | a message added; its UID with UIDPLUS |
| `noop()`, `idle(nanoseconds)`, `has_events()`, `expunges()`, `flag_changes()`, `take_events()` | news |
| `logout()`, `close()` | goodbye |

## Test

```sh
./test.sh          # parser tests and sessions against a scripted loopback server
```

`dev/live.lucb` runs against a real server: `luce-base build dev/live.lucb
--native -o build/live`, then `build/live HOST PORT plain|starttls|tls USER
PASSWORD`. luced-message's `tools/test-server.sh` starts a local GreenMail.

## License

Dual-licensed under Apache-2.0 or MIT, at your option.
