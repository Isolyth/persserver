I'm thinking about a 'personal server', as in like a software object not a literal server. It started when thinking about a task app I'm making and the server for it - why not have it do more? I mean why not? Right now I'm thinking the stuff that makes the most sense is:

Tasks API so various clients can get tasks, polling is probably fine? Can cache tasks local and poll the server when tasks are 'due', to refresh the local cache as well as verify task to notify about still exists, etc on this pattern
Lists! Literally just lists of objects.
Just direct file access? For small files like markdown. Attach to this and then use obsidian or access directly

What could make sense but also not end up bloating this into a 'these should really be separate'?

(⌐■_■)

The unifying question is: what makes something belong in a personal server versus being its own app? My answer would be — it belongs if it's small structured state that multiple clients need to agree on. That's tasks, lists, notes, and a handful of things you haven't listed yet. It doesn't belong if it needs its own storage engine, its own query language, or its own long-running compute.

Things that fit that test well:

Events / calendar-ish items. Tasks with a start time and a duration are basically the same object with different projections. If you're already doing due-date polling, you have 80% of the machinery. Keep it dumb — no recurrence rule engine at first, just RRULE strings you store and let clients interpret, or a materialized "next N occurrences" table.

Key-value / config store. Sounds boring but it's the thing you'll want constantly: "what's my current focus mode", "reading position in this doc", client preferences that should sync. A namespaced KV with last-write-wins is like 40 lines and kills a dozen future one-off features.

Bookmarks / link inbox. It's a list, but with fetch-on-save for title and maybe a cached readable extract. The only reason it's not just "a list" is the enrichment step, which is small.

A generic event log / journal append endpoint. Anything timestamped you want to shove somewhere: weights, moods, "took meds", "started task X". One table, (timestamp, stream, payload_json). Everything time-series-ish becomes a client concern rather than a server feature. This is probably the highest leverage thing on the list because it absorbs infinite future ideas without schema churn.

Webhooks / a job queue for notifications. Not a feature so much as the plumbing that makes tasks actually useful. Though see below.

Things I'd push back on:

Full-text search over everything. Tempting, and it feels unifying, but it drags in an index, a tokenizer, staleness problems, and reindex jobs. If you're doing direct file access, let Obsidian or ripgrep own search. Revisit only when you have enough data that grepping is actually slow.

Sync/CRDT for the files. Direct file access over WebDAV or plain HTTP is fine. The moment you want real conflict resolution, you're writing Syncthing. Use Syncthing.

RSS reading, media library, photos, anything with blobs. Blobs have completely different lifecycle needs (dedup, thumbnails, streaming ranges, backup cost) and they'll distort your storage decisions for the structured stuff. Separate service, maybe same box.

Push notifications as a server feature. Emitting a webhook or an ntfy POST is fine. Owning APNs/FCM credentials, per-device tokens, retry, and delivery receipts is a real project. Delegate to ntfy or Gotify.

Automation/rules engine. "When task overdue, do X." Seductive, becomes a DSL, becomes a debugger. If you want this, run it as a separate process that polls your own API — which also proves the API is good.

On the polling question: polling is fine, but add an updated_since cursor and a monotonic revision number per object from day one. It costs nothing now and it's miserable to retrofit. A single SSE endpoint that just emits "something in namespace X changed, go re-poll" is also cheap and lets you drop poll frequency without building real push.

The one structural decision I'd make early: are tasks/lists/events genuinely separate tables, or are they one item table with a type discriminator, arbitrary JSON body, and a small set of indexed common fields (id, type, created, updated, due, parent, tags)? The second gives you the "why not have it do more" property almost for free — new object types are a client concern. The cost is that you can't do interesting relational queries server-side, which for a single-user personal server you probably never needed.

Or why not just have these all be separate?

(￣ー￣)

Because "separate" is doing a lot of work in that sentence, and the answer depends which separation you mean. There are at least four axes and they don't have to move together:

separate schemas/tables — yes, almost certainly, or at least separate namespaces
separate modules/routers in one codebase — yes, obviously good
separate processes — probably not
separate repos, deploys, databases, auth systems, backup jobs — almost certainly not

The thing you actually pay for when you split at the process level, for a single user:

Auth becomes n problems. One personal server, one token, done. Five services and now you either have five token stores or you're standing up an identity provider so you can log into your todo list. That's the classic self-hosting death spiral.

Cross-object references stop working. A task that points at a list item, a note that's attached to a task, a log entry tagged with a project. In one process these are a foreign key and a join. Across processes they're a dangling ID you have to resolve with an HTTP call that might fail, and there's no referential integrity, so you get orphans forever. This is the real cost and it shows up later than you expect.

Transactions vanish. "Complete task and append to log and bump the project's updated_at" is one write in one process. Across three services it's a distributed write with partial-failure states you will not handle.

Ops multiplies by n. n systemd units, n backup targets, n TLS terminations, n sets of migrations to remember. On NixOS this is less bad than most places, but backup and restore-testing still scale linearly and you'll skip them.

Client config multiplies by n. Every client needs five base URLs instead of one.

What you get in exchange — independent deploys, independent scaling, blast-radius isolation, polyglot freedom — is mostly worthless at one user. You will never scale lists independently of tasks.

So the useful version of "why not separate" is: build them as separate modules behind one process, one database, one auth boundary, one deploy. Keep the boundaries clean internally — no module reaching into another's tables, talk through function calls that look like they could become HTTP calls. Then if something genuinely wants to leave later (the blob-heavy thing, the thing you want to open to a friend, the thing written in a different language), the seam already exists and extraction is a day's work rather than a rewrite.

The counter-case, to be fair: if a component has genuinely different failure or resource characteristics — a media transcoder that pins CPU, a scraper that gets rate-limited and wedged, something you want to restart constantly while developing — separate process is right, because you don't want it taking your task list down with it. That's a real criterion. "These are conceptually different nouns" isn't.

What's the existing version of this mega-app?

Thought for 9s

(¬‿¬)

Several, at different points on the spectrum:

The protocol answer: CalDAV/CardDAV/WebDAV. This is literally the standardized version of your idea — tasks (VTODO), events (VEVENT), contacts, and files, all over one HTTP server with one auth. Radicale or Baïkal is ~one config file and you get a huge client ecosystem for free. The problem is it's a 1990s data model: iCalendar is a text format with a parser everyone implements slightly differently, arbitrary structured objects aren't really a thing, and querying is awful. But if you only wanted tasks + lists + files, this already exists and you'd be reinventing it.

Nextcloud. The maximalist version. Files, tasks, calendar, contacts, notes, bookmarks, and a hundred apps, one auth, one DB. It's exactly the thing you're describing, taken to its conclusion — which is why it's also the cautionary tale: PHP monolith, heavy, slow, and the app quality varies wildly. Worth studying as "what does this look like at year ten."

Anytype / Tana / Fibery / Notion. The "one item table with a type discriminator" idea, productized. Everything is an object with a type and properties; tasks and lists and notes are just views over the same substrate. Anytype is the closest to your case since it's local-first and self-hostable. This is the strongest prior art for the data model question specifically.

Trilium / Logseq / Obsidian+plugins. Notes-first, then everything else gets built as a projection of notes. Given you already mentioned Obsidian, this is the path of least resistance — but the ceiling is low once you want real API access from non-note clients.

Sandstorm, YunoHost, Cloudron. These answered "why not separate" with: keep them separate, but unify auth, deploy, and backup. Sandstorm's grain model in particular is the most interesting attempt at making per-app isolation not cost you a login per app. It mostly failed commercially, which is itself informative.

remoteStorage, Solid, AT Protocol PDS. "Personal server as a substrate, apps are just clients" as an explicit thesis. Solid is the most academically pure and the least usable; a PDS is the most currently-alive version of the idea. Useful for the conceptual framing even if you don't adopt any of them.

Urbit. The maximal version of your instinct, taken past the point of sanity. Included because it's the one that most directly says "a personal server should be one persistent object that does everything."

Home Assistant. Not personal-data, but the best-executed example of the architecture you're describing: single process, single auth, event bus in the middle, integrations as modules, everything speaks to one state machine. If you want a model for how to structure the internals so modules don't tangle, it's a better read than Nextcloud.

The gap you'd actually be filling: nothing in that list is small, API-first, single-user, and modern-schema at once. CalDAV is small but ancient, Nextcloud is modern-ish but enormous, Anytype has the right model but is an app rather than an API you build clients against. That's a real hole — which is a decent argument for building it, and also the reason to keep it deliberately tiny so you don't drift toward the Nextcloud end.

(๑•̀ㅂ•́)

Then the design question stops being "which features" and becomes "what's the smallest kernel a plugin can be written against." Everything else is a plugin, including the tasks app you started from.

The kernel is basically three things: a storage primitive, an event bus, and a routing table. Storage: plugins get a namespace and a handle, they don't touch each other's data directly. Events: plugins emit and subscribe, and this is the only way they compose — no direct calls, no imports. Routing: plugins register HTTP handlers under their namespace. That's it. If you find yourself adding a fourth kernel concern, be suspicious.

The event bus is the important one. It's what makes "why not have it do more" not turn into spaghetti, because a plugin that wants to react to task completion subscribes to tasks.item.completed and never learns that the tasks plugin exists. It's also what gives you your updated_since cursor and your SSE change-notification endpoint basically for free — those are just the bus, projected.

The load-bearing decision is what a plugin actually is, and there are three answers with very different consequences:

In-process, same language. Fastest to build, zero IPC, shared transaction scope. Cost: every plugin can crash your server, and "plugin" means "you recompile." For Rust this means either compiling plugins in or dealing with dynamic loading, which is unpleasant.

WASM components. The fashionable answer and genuinely a good fit here — sandboxed, hot-loadable, language-agnostic, small. You define the host interface in WIT (storage, emit, subscribe, register-route) and that interface is your kernel. The cost is that the component model tooling is still sharp-edged and async host functions are annoying. But if you want "hyper-minimal kernel, everything else is a plugin" to be literally true rather than aspirational, this is the one that enforces it.

Out-of-process, plugins are just services that register. The kernel becomes a router plus a message bus, and a plugin is anything that can speak the protocol — including a shell script. Maximum flexibility, and it means "install a plugin" can mean "point at a URL." Cost: you're back to distributed writes, and the kernel can't offer transactions across plugins. But for single-user this might be fine, and it's by far the easiest to develop against.

Given Iroh is in the picture, the third is interesting, because the transport is already the thing you'd use for plugin↔kernel as well as client↔kernel. A plugin and a client become nearly the same object, distinguished only by whether they register routes. That's an elegant collapse if you can stomach it.

On Iroh specifically: the thing to get right early is that it's a transport, not an auth system. Node identity gives you "which key is this", not "what may it do." You want a capability model on top — per-client tokens scoped to plugin namespaces and verbs, so your phone's task widget can't read your journal. Bolting that on later is painful, and it's the single most likely thing to get skipped in a minimal design. Also: Iroh gives you QUIC streams, so change-notification wants to be a stream rather than SSE-over-HTTP, and you probably want the HTTP surface to be a shim over the real protocol rather than the primary interface — otherwise you'll design everything around request/response and the P2P path becomes second-class.

Things to consciously refuse to put in the kernel: search, notification delivery, auth policy (mechanism yes, policy no), any schema knowledge about what a task or list is, migrations for plugin data, and cron. Cron especially is tempting — your due-date polling wants it — but a plugin that emits tick events on a timer is a plugin, not a kernel feature.

The test I'd apply: can you write the tasks plugin, delete it, and have the server still start and be useful to a different plugin that knows nothing about tasks? If yes, the kernel is actually a kernel. If the server has a tasks-shaped hole in it, you built a task app with an extension mechanism.

A goal, but I'm unsure if it's possible: could the whole thing be made entirely, 100%, stateless?

(・_・;)

Not 100%, no — but the interesting part is exactly which piece is irreducible, because it's smaller than you'd think.

The hard floor: something has to durably hold bytes that survive process death, and something has to serialize concurrent writes. Those are the two irreducible bits of state. Everything else — routing tables, plugin registry, subscriptions, caches, sessions, cursors — can genuinely be made derivable or client-held.

So the honest goal is a stateless kernel over an external store, which is a real and achievable thing, and materially different from "stateful server." The kernel process holds nothing you'd cry about losing; kill -9 it at any moment, restart, and it's identical. That gets you almost every property you actually want from statelessness: trivial restarts, no migration of in-memory structures, no "it's been up 40 days and something is weird," testable by construction, and you can run two of them.

How far you can push each piece:

Plugin registry and routing — derive it. Plugins register on connect; the table is rebuilt from whoever's currently attached. Nothing persisted. If the kernel restarts, plugins reconnect and the table reassembles. This is the Erlang-ish answer and it works well.

Subscriptions — same, they live and die with the connection. Client reconnects, resubscribes, and asks "what have I missed since cursor X" — which is a query against the store, not kernel memory.

Auth — fully stateless is achievable with signed capability tokens: the token carries its own scope, kernel verifies a signature and doesn't look anything up. The catch is revocation, which fundamentally needs either state or short expiry. Short expiry plus a refresh path that does touch the store is the usual dodge.

Cursors and read position — push to the client entirely. This is the big one and it's very doable if your store gives you a monotonic sequence.

The event log — here's where it gets nice. If your store is append-only with a total order, the log is the state, and the kernel becomes a pure function from (log, request) to (new entries, response). Change notification is just "tail the log from N." Everything a plugin knows becomes a projection it can rebuild. This is the design where "stateless" stops being aspirational and becomes structurally true.

Where you can't escape:

Write serialization. Two clients complete the same task simultaneously; someone must decide the order. You can delegate this to the store — SQLite's write lock, Postgres, a log with CAS-on-append — and then the kernel is stateless, but the system is not. Delegation is legitimate; pretending it's absent is not.

Timers. A due-date reminder means something must remember to wake up. You can push this outward: clients hold their own local schedules and poll, which fits your original polling instinct exactly and is genuinely stateless server-side. But then reminders only fire when a client is awake, which may or may not be acceptable. If you need server-side firing, that's a stateful process, and it should be a plugin, not the kernel.

Plugin private state. A plugin that caches or maintains a projection has state. You can require plugins to be pure functions over the log, which is beautiful and also a real constraint on what plugins can be. Probably worth making the default rather than the rule.

Given Iroh, one more wrinkle: connections themselves are state, and a P2P transport with NAT traversal has meaningful session state — handshakes, hole-punched paths, key material. You can call that transport-layer and not application-layer, and that's fair, but if a kernel restart drops every connection and every client has to re-traverse, "stateless" is doing some marketing work.

The version I'd aim for: kernel is a pure function, state lives in an ordered append-only log, clients hold their own cursors, timers are a plugin's problem. That's about as close as the idea can get without becoming a lie, and it's a genuinely clean architecture rather than a compromise.

What about stateless with a separate 'poker' that 'pokes' every amount of time and processes any 'persistent' type things? Then you can run the thing on kubes/workers/whatever, the poker is tiny and just a clock smacking an endpoint, etc

(￣▽￣)ノ

That's the right shape, and it's a well-trodden one — it's how Cloudflare Workers people, Lambda people, and honestly most cron-driven systems handle exactly this. The poker is a dumb clock, the kernel stays a pure function, and "did anything become due" turns into a query rather than a set of in-memory timers. Good instinct.

The thing that makes it work is that the poker must carry no knowledge and no state. It knows one URL and one interval. It doesn't know what tasks are, doesn't track what it processed last, doesn't retry intelligently. Everything it might want to remember — last processed timestamp, which reminders already fired — lives in the store. The moment the poker knows something, you've just relocated the stateful server rather than eliminated it, and it's now a stateful server you're pretending is a cron job.

The corollary is that the endpoint has to be idempotent and re-entrant. Poker fires twice because a retry overlapped, or two pokers exist because Kubernetes did a thing, or a poke takes 90s on a 60s interval — all of these must be harmless. Concretely: the tick handler queries "what's due and unfired as of now," and marks fired via a conditional write, so a duplicate tick finds nothing to do. Never "process everything since my last run," because that requires the poker to know when its last run was.

Where this design actually bites, worth knowing up front:

Resolution vs. cost. Your minimum reaction latency is the tick interval. Minute-granularity reminders need a 60s tick, which is 1,440 wakeups a day mostly finding nothing. Fine at your scale, and you can make the empty case a single indexed query returning zero rows, which is nearly free. But if you ever want second-granularity, this shape stops being appropriate and you need real timers.

Long jobs. If a tick's work exceeds the interval, you need either a lock (state) or work that's naturally chunked and idempotent so overlap is fine. Prefer the latter — have the tick process a bounded batch and return, and let the next tick pick up the rest. Self-limiting, no coordination.

Missed ticks. Machine asleep, deploy, network partition. Because the handler is "what's due now" rather than "what happened since," a gap self-heals on the next tick — things just fire late. This is the main reason the query-based framing beats the log-scanning framing, and it's worth being deliberate about.

Clock skew across workers if more than one thing can tick. Use the store's clock as authoritative, not the poker's.

For your specific stack, the poker can be almost insultingly small: a systemd timer hitting curl, a Kubernetes CronJob with a busybox image, or on NixOS a systemd.timers unit in three lines. Resist making it a service. If you ever find yourself adding config to the poker, that's the signal something belongs in the kernel or in a plugin.

One extension that keeps the purity: let the tick be fanned out by the kernel to plugins as a tick event rather than the poker knowing about endpoints. Poker hits /tick, kernel emits, every subscribed plugin does its own due-check. New plugin with periodic needs? Subscribes. Poker never changes. That preserves your "delete the tasks plugin and nothing breaks" test.

And the honest asterisk: this doesn't make the system stateless, it makes the compute stateless and pushes all statefulness into the store plus an external clock. Which is exactly what you want for running on workers — but the store is now doing real work, and it's the thing you actually have to think hard about, back up, and not lose.

So basically it just turns into a wrapper for specific operations on some database

(￣ー￣)ゞ

Yes. And that's not a deflating conclusion — that's the destination. Nearly everything in this space is that, and the ones that pretend otherwise are usually worse for it. Once you accept it, the design questions get much sharper, because they're all now about where the wrapper adds value over just handing clients a database connection.

The real answers to "why not just use the DB directly":

Authorization. A database gives you row-level security if you fight it; a wrapper gives you capability tokens scoped per plugin namespace and verb, which is the thing you actually wanted. This is probably the single strongest justification for the wrapper existing at all.

A stable contract. Clients bind to an API, not a schema. You can restructure tables without breaking your phone.

The event log and change notification. Databases are bad at "tell me what changed since cursor N, over a P2P connection, without polling." That's genuinely yours to build.

Reachability. Exposing Postgres to the internet is a bad time; exposing an Iroh endpoint that speaks a narrow protocol is fine. The wrapper is your network boundary.

Cross-plugin composition. The event bus is the piece with no database equivalent. That's the actual product.

The thing worth sitting with: if it's a wrapper over a database, then the schema is the architecture, and all the earlier questions collapse into one. Single item table with type discriminator versus per-plugin tables — that's now the decision, and it determines whether plugins can be added without touching the kernel. Same-database plugins can share transactions and join; separate-database plugins can't, and you're back to distributed writes. Migrations become the plugin lifecycle problem you refused to put in the kernel, and you'll have to put something there, probably "plugins own their namespace and run their own migrations at registration."

Also, once framed this way, the answer might be smaller than you think. SQLite plus a thin HTTP layer plus the event bus gets you very far, and things like PostgREST, Supabase, Datasette, and CouchDB are all points on the "wrapper over a DB" spectrum that already exist. Worth a look — not to adopt necessarily, but because if PostgREST plus a tick endpoint plus row-level security covers 80% of it, the remaining 20% (the bus, Iroh, the plugin model) is your actual project, and it's much smaller and more interesting than building the whole thing.

The version of this that goes wrong is when the wrapper starts accumulating logic that should've been in the client or the schema — validation rules, computed views, business logic about what a task means. Keep it a wrapper. The discipline is that if you can't explain what a given endpoint adds over a parameterized query plus an auth check, it probably shouldn't exist.