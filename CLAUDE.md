# Stratus

Self-hosted personal cloud: photo backup, calendar, music and video.
Same goals as Nextcloud, radically fewer moving parts.

This repository is the workspace: what binds every Stratus repo, and nothing
that belongs to one of them. Each repo's own half -- its architecture, its
configuration, its abstractions and its tech decisions -- lives in that repo's
own `CLAUDE.md`, so that a change to the code and the paragraph describing it
are the same pull request. Claude Code reads both, since it merges the file from
every parent directory.

Repos in this workspace:

| directory | repo | what it is |
|---|---|---|
| `backend/` | `stratus-backend` | the server: one Go binary, one container |
| `app/` | `stratus-app` | the iOS and Android client: one Kotlin Multiplatform codebase |

## Non-negotiable principles

1. **One binary, one container.** No Redis, no separate web server, no PHP, no
   external job queue. Background work is goroutines inside the same process.
2. **Reuse existing protocols instead of inventing APIs.** Every feature we ship
   should be usable from clients that already exist. The web UI is not an
   exception to this: it is server-rendered HTML for the most universal existing
   client there is. It consumes the same internals as the protocol handlers and
   **must never grow a private JSON API for its own use** -- that is how this
   principle gets broken quietly.

   This principle used to end with "we do not write mobile apps", and
   `stratus-app` is the single exception we have granted it, conditionally.
   Automatic camera-roll backup is the one thing no existing client does for
   free on iOS, and it is what a personal cloud gets judged on. The condition is
   that the app speaks **only standard protocols and stays useful against any
   WebDAV server**, so it can never become the reason to add a Stratus-only
   endpoint here. The day an app feature would be easiest to satisfy with a
   private API is the day this principle is actually on the line.
3. **Pluggable at exactly two seams:** metadata database and blob storage. Nothing
   else gets an abstraction layer "just in case".
4. **Single user for now**, sharing later. Don't hardcode assumptions that block it:
   keep an owner id on records even while it is always the same value.
5. **Minimal dependencies.** Prefer stdlib. Pure-Go / CGO-free build so the image
   can be `scratch` or distroless with a static binary.

## Protocol surface

| Protocol | Use | Clients |
|---|---|---|
| WebDAV | files, photo upload, generic sync | rclone, Finder, Nautilus, FolderSync |
| tus | resumable upload of large files, negotiated | tus-js-client, TUSKit, tus-android-client |
| CalDAV | calendar | DAVx5, Thunderbird, iOS/macOS |
| OpenSubsonic | music | Symfonium, Substreamer, DSub, Feishin |
| HTTP range | direct video/audio streaming | any browser, VLC, mpv |
| Web UI | login, browse, upload, download, rename and delete files today; the calendar later | any browser |
| CardDAV | contacts | *later* |
| DLNA / UPnP-AV | TVs, set-top players | *later* |

\* Finder needs WebDAV class 2 to mount read-write, so the server advertises it
and answers `LOCK`/`UNLOCK` with a token nothing records. Two clients writing
the same file are not protected — they never were — and the real defence against
a lost update is the strong ETag and `If-Match`.

## Working agreements

- **Deferred problems become issues, not conversation.** Anything found and
  consciously left for later — a bug, a gap in the tests, a tech decision that
  contradicts a stated goal — gets a GitHub issue before moving on, filed against
  the repo it affects and added to the project board. Say what the problem is,
  what evidence there is for it, what the options are, and what it blocks. A
  problem that exists only in a chat log is a problem lost.

- **The README is part of the deliverable.** It has to answer two questions
  correctly at every commit: *what works today* and *how do I run it*. A change
  that adds a surface, a setting, a requirement or a limitation is not finished
  until the README says so in the same pull request.

  Concretely it carries: the protocol table with what actually works rather than
  what is intended, the quickstart with anything a first run genuinely needs,
  the configuration variables an ordinary install touches — and the ones it does
  not, kept out of the main table on purpose — and the operational behaviour
  somebody would otherwise discover the hard way, such as what is swept in the
  background or what a lock does and does not guarantee.

  This is not documentation hygiene. A self-hosted project is judged in its first
  five minutes, and a README that promises a protocol that does not answer, or
  an image size that is three times off, costs more trust than the feature it was
  advertising was worth. It has drifted six times already: the SQLite DSN, the
  ffmpeg-optional line, the WebDAV client table, "greenfield" long after it
  stopped being greenfield, the image size — advertised at ~55 MB while the real
  thing is 145 MB, which is the very example this paragraph uses — and that
  client table a second time, listing DAVx5 against photo backup when it does no
  such thing. Twice in the same table: when a row says which clients work, check
  that they do the job the row claims and not merely the protocol.

## Status

Single user, usable from a WebDAV client, and a music library over
OpenSubsonic.

- Workspace: https://github.com/C0piIot/stratus
- Backend: https://github.com/C0piIot/stratus-backend
- App: https://github.com/C0piIot/stratus-app
- Board: https://github.com/users/C0piIot/projects/2

Working:

- Both seams, each with a conformance suite every one of its drivers passes:
  disk and s3 for blobs, sqlite, postgres and mysql for metadata.
- Resumable uploads over tus at `/tus/`, because a `PUT` is all or nothing and a
  large video on a mobile connection never finishes without them. The storage
  port grew a second half for it -- start, append, ask, complete, abort -- that
  both blob drivers honour truthfully, and an upload in progress is a row so it
  survives a restart. No tus client library has been pointed at it yet.
- WebDAV at `/dav/`, behind HTTP Basic with a global rate limit on failed
  logins, mounted only when credentials are configured.
- OpenSubsonic at `/rest/`, browsing by tag and by folder, search, the album
  lists a home screen is made of, streaming, and cover art -- over both of the
  protocol's authentication schemes and sharing that same rate limit. No real
  client has been pointed at it yet, so the README says so.
- `internal/files` holding the blob-plus-row invariant, and a background sweep
  that collects the blobs an overwrite leaves behind.
- A media indexer extracting EXIF, audio tags and video probes, and thumbnails
  made on first request and kept as derived blobs the same sweep collects.
- A web UI at `/`: sign in, walk the tree, download a file, upload one, make a
  folder, rename and delete. The upload streams into the blob store and replaces
  like a PUT does; deleting asks first, because there is no trash bin. The
  session is
  signed rather than stored, keyed by the configured password, so changing it
  revokes every cookie already issued and a restart revokes none.
- Migrations applied at startup, a request log, and a container asserted from
  outside by the smoke suite: static binary, no shell, non-root, hardened
  runtime, data-directory and configuration failure matrices.

Not written yet: CalDAV and the calendar view over it, and sharing. Nor renaming
a folder that has anything in it, which no surface can do: moving a directory is
a rewrite of every path under it. Nor thumbnails of what only ffmpeg can
decode -- HEIC and video -- or anything that remembers what a user did.

`stratus-app` exists but holds no code yet: only the decisions, chief among them
that it negotiates its upload transport -- plain WebDAV `PUT` against any server,
[tus](https://tus.io) where the server offers it, because `PUT` cannot resume and
a large video over mobile data therefore never finishes. The server implements
tus now (stratus-backend#122), so the negotiation has something to find. Note
where the two repos meet: the app uploads HEIC originals and never transcodes, so the
missing HEIC thumbnails above stop being a backlog item and become the first
thing anybody sees.

The board carries a `Priority` field for when, and a `decision` label for the
issues that need a call before anyone can start.
