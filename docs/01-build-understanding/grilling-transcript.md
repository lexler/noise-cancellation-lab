# Grilling transcript: stash

**User:** I have too many browser tabs open and I forget what I saw. When I want to remember something I email it to myself, which is ridiculous, and then I cannot find it again. I tried bookmarks instead, but they don't help because I can't remember why I cared to start with, and can't search by my notes or additional context. I need a place where I can send a link with a short note on why it was interesting, and later search for it by keyword and get the link back.

I'm thinking it should support several input channels, and I might add more channels in the future:

- A Chrome button. It pops up, has the URL and title filled in, asks me what exactly I want to remember, maybe tags, and saves.
- A chat message. Telegram probably, but it does not have to be Telegram, it should be able to support different channels. Sending a message to something, that is the idea.
- Possibly a command line tool

Everything is saved to a new web app I'll host for this, and I probably want a simple interface for looking through added items and also searching by keywords. Search from the command line would be nice too.

Tags, maybe, and maybe automated to some degree. If search covers my note and the page title, tags might not be needed at all, I am not sure.

The job of the input channel will just be to capture, and one core owns the store and the search. The core is hosted, so the phone works and I do not run anything on the computer. The web page for searching lives in the same place as the core, and the command line talks to the hosted core without a local store.

Interview me relentlessly about every aspect of this application until we reach a shared understanding. Walk down each branch of the design tree, resolving dependencies between decisions one-by-one. For each question, provide your recommended answer.

Ask the questions one at a time.

If a question can be answered by exploring the codebase or existing documentation, explore the codebase instead. Don't update the document and don't start building it.

**Agent:** I'll first look at what's already in the repo so I don't ask about things that are already decided there.

The repo is a starter: an empty C# console app on .NET 10 with NUnit tests, an empty docs/ folder, and a README that calls it a learning-hour starter. So I'll take C# / .NET 10 as the language for the core unless you say otherwise, and I won't ask about that.

Let's start at the root, since most other decisions hang off it.

Question 1: Who uses this?

Is it strictly a single-user system, just you, with one identity everywhere? Or should the core be built with the notion of multiple users, even if you are the only one for now?

My recommendation: single user. No accounts, no user table, no per-user filtering. Every channel authenticates with a shared secret, a long random API token the core knows, and that is the entire security model. It cuts a lot of complexity out of the store, the search, and every channel. If a second person ever wants it, they run their own instance.

**User:** just me

**Agent:** Good, single user, one shared token.

Question 2: What exactly is one saved item?

I have so far: a URL, the page title, your note on why it was interesting, maybe tags, and implicitly a timestamp. Two things need a decision.

First, is the URL required? A Telegram message might be just a thought with no link. Should the core accept note-only items, or reject anything without a URL?

Second, is the URL a key? If you save the same URL twice, is that two items, or does the second save update the first?

My recommendation: URL is optional, so the same core works as a plain memory dump too, and a message that happens to contain no link still gets kept. And the same URL saved twice is two separate items. Deduplication sounds tidy but it destroys information, such as the two different reasons you found the page interesting a month apart. The search page can group by URL later if duplicates ever bother you.

**User:** i like optional url but I don't like same url saved twice being two separate items. If I'm trying to add something that I already added, I think we should at least attempt to find the error like that. And show me that I already have something like that. Maybe I get an option to update it at the time.

**Agent:** So capture becomes a conversation: the core checks first, shows you what it already has, and you choose. That shapes the API for every channel, so let me pin down the matching rule.

Question 3: What counts as "the same" item?

Options, from strict to loose:

1. Exact URL string match only.
2. Normalized URL: ignore http vs https, www, trailing slash, fragment, and known tracking parameters like utm_*. So a link copied from a newsletter matches the one you saved from the browser.
3. Normalized URL, plus a "looks similar" check on title and note text, shown as a softer "you may already have this" hint.

My recommendation: option 2 as the hard match, which triggers the "already saved, update or keep both?" flow. Skip option 3 for now. The fuzzy check is exactly what the search page is for, and doing it at capture time slows down the one moment you want to be fast. If note-only items with no URL ever collide, nothing checks them, and that is fine.

**User:** agreed on normalized url, so 2

**Agent:** Question 4: When you choose "update", what happens to the old note?

You save a URL, the core says "you already have this, with note X from March." You pick update. Does the new note replace X, or get appended to it as a second entry with its own date?

This also decides whether an item holds one note or a list of notes, which the store and the search page both need to know up front.

My recommendation: append. An item is a URL plus a list of dated notes. Replacing throws away the March reason, and "I found this interesting twice, for these two reasons" is exactly the kind of thing you want to see later. The channels stay simple: they only ever send "URL, note, tags," and the core decides whether that becomes a new item or a new note on an existing one. Editing or deleting an old note happens on the web page, not through capture.

**User:** I think it fills in existing note in an editable box so you can edit the message and add or remove something from it if you want to

**Agent:** So an item has one note, and update means the channel shows the existing note pre-filled, you edit it, and the whole thing is saved back. Clean for the Chrome popup, the web page, and the CLI, which can open your editor with the old text.

Question 5: How does the duplicate flow work in a chat channel?

Telegram cannot pre-fill your input box. So when you send a link the core already has, the bot can only reply with text. Options:

1. Bot replies "Already saved on 12 March: 'old note'. Send the full new note to replace it, or /keep to leave it." Your next message replaces the note.
2. Bot replies the same way, but your next message is appended to the old note, since retyping the old one on a phone is painful.
3. Bot just says "already saved: 'old note'" and does nothing. Editing happens later on the web page.

My recommendation: option 2, but only in chat. The core keeps the rule "update replaces the note in full." The Telegram channel handles the awkwardness itself: it shows the old note, takes your reply, joins old plus new, and sends the joined text as the replacement. That keeps the core's API uniform across channels while giving the phone a workable flow. The bot also copies the old note as a plain message so you can long-press, copy, and edit it if you want a real rewrite.

**User:** let's do 2

**Agent:** That settles the item: optional URL (normalized as key), title, one note, timestamps, and maybe tags. Tags are the last open field.

Question 6: Tags. In, out, or later?

You said search over note and title might make tags unnecessary. Three ways to go:

1. No tags at all. Search covers note, title, and URL. Done.
2. Free-form manual tags, optional on every channel, searchable like any other word.
3. Automatic tags, where the core asks an LLM to suggest a few tags from the page and your note.

My recommendation: option 2, kept deliberately cheap. A tag is just a word you type, stored as a list of strings, with no tag management screen. Search treats tags as more words to match, so the search engine does not change. The reason to have them at all is not classification, it is that "#recipe" is a shorter and more reliable search term than whatever you happened to write in the note. Automatic tags are worth trying later as a plug-in step, since the item shape already has the field, but not before you know whether you ever type tags yourself.

**User:** let's remember that we want tags but let's not do it in the mvp just yet

**Agent:** Noted: tags out of the MVP, but the item keeps room for them.

Question 7: What should search feel like?

Search is the whole point, so let me pin down the behaviour before choosing a store, since the store follows from this. Options:

1. Plain keyword match: every word you type must appear somewhere in note, title, or URL. Case-insensitive, prefix-tolerant so "recip" finds "recipes". Results newest first.
2. Ranked full-text search: same input, but results are ordered by relevance with stemming, so "running" finds "run".
3. Semantic search: embeddings, so "that thing about slow tests" finds a page whose note says "flaky CI".

My recommendation: option 1 for the MVP. With a personal collection of a few thousand items, all words must match plus prefix tolerance is precise and predictable, and newest-first ordering is what you want when you half remember "I saw this last week." Stemming and ranking rarely matter at this scale. Semantic search is the interesting upgrade, but it changes the store and adds an external dependency, so it should be a second phase once the plain version has shown you where it falls short.

**User:** agreed 1

**Agent:** Question 8: Where does the core run?

Everything is hosted, so this decides the store, the deploy, and how the Telegram bot receives messages. Options for a .NET app:

1. A small VPS you already own, running the app in Docker behind Caddy for HTTPS. Store is a SQLite file on disk, backed up by copying the file.
2. A container platform like Fly.io or Railway. Same Docker image, they handle HTTPS and the domain, SQLite on a persistent volume.
3. A managed platform like Azure App Service with a managed database.

My recommendation: option 2, Fly.io, with SQLite on a volume. It is one Dockerfile, one fly deploy, HTTPS for free, a public URL that Telegram can call as a webhook, and it costs a few dollars a month or nothing. SQLite is the right store for one user with plain keyword search: no separate database service, backups are one file, and the search from question 7 is a LIKE query or SQLite's built-in FTS5 later if you want it. If you already run a VPS for other things, option 1 is equally fine and I would pick whichever you already log into.

The one thing I would rule out is a managed database. It is the biggest cost and operational item in the whole design and buys nothing here.

Do you already have a VPS or a platform account?

**User:** let's do 2 yes

**Agent:** Fly.io with SQLite on a volume.

Question 9: How does each channel prove it is you?

The core is on a public URL. Three callers need handling:

- Chrome extension and CLI: send the shared API token in an Authorization header. You paste the token once into the extension options and once into a CLI config file.
- Telegram: the bot receives a webhook from Telegram's servers. The core checks the webhook secret Telegram signs with, and then only accepts messages from your own chat ID. Anyone else who finds the bot gets ignored.
- The web search page: this one is open. Options are a login form that sets a cookie, HTTP basic auth, or a bookmarkable URL with the token in it.

My recommendation: as above for the extension, CLI, and Telegram. For the web page, a single password field on first visit that sets a long-lived cookie. It is a few lines, works on the phone, and you log in once per device. Basic auth is even less code, but phones handle it badly and you cannot log out. Token-in-URL leaks into history and screenshots.

Do you accept that, and is the web page password the same string as the API token or a separate one?

**User:** for web page i want gmail login

**Agent:** Google sign-in it is: the core does the OAuth dance with Google, accepts exactly one email address, yours, and sets a session cookie. That needs a Google Cloud OAuth client once, and nothing else changes.

Question 10: Should the Chrome extension ride on that same login?

Since the extension runs in the same browser you sign into the web page with, it can call the core with your session cookie and skip the pasted token entirely. Options:

1. Extension uses the cookie. If you are not signed in, the popup shows a "sign in" link that opens the web page, and after that it just works. No token in the extension at all.
2. Extension has its own pasted API token, like the CLI, fully independent of the web session.

My recommendation: option 1. It removes a setup step and a secret from the extension, and it means the two browser surfaces share one identity. The CLI keeps the pasted token, since it has no browser. So the core ends up with three ways in: Google session cookie for browser callers, bearer token for the CLI, and the Telegram chat ID check.

**User:** yes that works

**Agent:** Question 11: What is the capture API between a channel and the core?

The duplicate flow needs a two-step exchange. Options:

1. Channel first calls "lookup by URL", then decides to create or update. Two round trips on every save, even when there is no duplicate.
2. Channel calls "save" once. The core answers either "created" or "conflict, here is the existing item." Only in the conflict case does the channel show the existing note and follow up with an "update item X" call.
3. Channel calls "save" with a flag "replace if exists" and the core silently overwrites. No conversation.

My recommendation: option 2. The common path is one call, the duplicate path is one more, and the decision stays with you rather than being made silently. Concretely the core exposes about four operations: save, update item, search, and list recent, plus delete for the web page. Every channel, including the web page itself and the CLI, uses only those. The Chrome popup gets the duplicate check for free the moment it opens, by calling lookup-by-URL for the current tab, so it can show "already saved" before you even type.

That last bit means the popup does use a lookup call after all, but only on open, not on save. Is that fine?

**User:** can we have some kind of protection on insert to database? then we won't need to implement this via an api, we could just react to that with the suggestion to update right away. is that what you meant in 2 or no? will need to research our db supports that

**Agent:** Yes, that is exactly what I meant, and SQLite supports it. A unique index on the normalized URL column, and SQLite treats NULLs as distinct in unique indexes, so any number of note-only items with no URL coexist fine. The core does a plain insert; if the database rejects it with a uniqueness violation, the core loads the existing item and answers "conflict, here it is." No pre-check, no race, and the rule lives in one place. The popup's lookup-on-open is just a convenience read on top of that.

Question 12: How does the Telegram channel turn a message into an item?

A chat message is free text. The bot has to find the URL, the note, and later the tags. Options:

1. Any message: the first URL found becomes the URL, everything else is the note. No URL means a note-only item. Save immediately, reply with a short confirmation.
2. Same, but if the message is only a URL with no other words, the bot asks "why is this interesting?" and waits for your next message as the note.
3. Structured commands like /save <url> <note>.

My recommendation: option 2. Forwarding a link from another app usually produces a bare URL, and the bot asking for the reason is the moment that makes this system better than emailing yourself. If you reply with /skip or nothing within a while, it saves with an empty note. Fetching the page title for a Telegram save is the core's job, not the bot's, since the bot has no browser. That is the next question.

**User:** 2 yes but the bot asks 'add note?', not 'why is this interesting?'

**Agent:** Got it, the prompt is "add note?".

Question 13: Where does the page title come from when the channel does not have it?

The Chrome popup has the title for free. Telegram and the CLI have only a URL. Options:

1. The core fetches the page when it receives a URL without a title, reads the <title> tag, and stores it. Done synchronously during save, with a short timeout.
2. Same, but in the background after the save, so the save reply is instant and the title appears a moment later.
3. No fetching. Items saved from Telegram or the CLI have no title unless you type one.

My recommendation: option 1. A title is searchable text you did not have to type, which is a big part of the value, and a two-second fetch during a Telegram save is invisible. If the fetch fails or times out, the item is saved without a title and the bot's confirmation says so. Sites that block bots or need login will simply come back empty, and that is acceptable. Background fetching is a later optimisation if the delay ever annoys you.

One consequence: the core now makes outbound HTTP calls, which the tests will need to fake.

**User:** 1

**Agent:** Question 14: What does the web page do, and how is it built?

Scope first. I have: a search box, a list of results newest first, each showing title, link, note, and date, and the ability to edit a note and delete an item. Anything else you want in the MVP? For example, "open the link and mark it as read", or a "random item" button.

Then the build. Options:

1. Server-rendered HTML from the .NET app, a form with a search box, and plain links. Editing is a small form per item. No JavaScript build step.
2. Same, plus HTMX for search-as-you-type without a page reload.
3. A separate frontend app in React or similar, talking to the JSON API.

My recommendation: scope as listed, nothing more. And option 1, one static page: the same JSON API the channels use, with a few lines of vanilla JavaScript that fetch results as you type and render them. No framework, no build step, shipped from the same app. It stays honest about the web page being just another channel over the API, and it works on the phone. If the page ever grows past one screen, that is the moment to reconsider.

**User:** Might need to filter by date. So filter by date and search I think. the rest like you said

**Agent:** Date filter it is: a from/to range on the search API, and the page shows it as two date fields next to the search box.

Question 15: What is the command line tool?

Two things to decide: what it does, and what it is made of.

Commands I have in mind: add <url> [note], which opens your editor for the note if you give none and handles the duplicate flow by opening the editor with the old note pre-filled, and search <words> [--since date], printing one line per hit with date, title, and URL. Configuration is a URL and token in a file under your home directory.

Build options:

1. A second C# console project in this solution, installed as a dotnet global tool. Shares the item model and API client with the core, tested with NUnit like everything else.
2. A shell script wrapping curl and jq. Twenty lines, no build, but the duplicate flow and editor handling get ugly fast.
3. Python or Go.

My recommendation: option 1. It lives beside the core, the API client it uses is the same one the tests use to drive the core, and you already have the toolchain. The shell script is tempting but the editor-and-conflict flow is where it would fall apart.

Is that command set enough, and do you want anything on the phone side of the CLI, or is it laptop only?

**User:** ask me one question a time only

**Agent:** Understood.

Question 15: Is the CLI a second C# console project in this solution, installed as a dotnet global tool?

Recommendation: yes. It shares the API client with the core's tests, uses the toolchain you already have, and handles the editor-and-duplicate flow far better than a shell script over curl would.

**User:** ok let's do a separate one - like a client to our backend

**Agent:** Question 16: Is this the CLI command set for the MVP?

- add <url> [note]: saves. With no note given, opens your editor. On a duplicate, opens the editor with the existing note pre-filled and updates on save.
- search <words> [--since <date>] [--until <date>]: prints one line per hit with date, title, and URL, newest first.
- Configuration: core URL and API token in a file under your home directory.

Recommendation: yes, exactly these two commands and nothing else. List, edit, and delete stay on the web page.

**User:** i'd rather have a list

**Agent:** Question 17: Does "list" mean a third command, list [--since <date>], that prints recent items newest first without a search term?

Recommendation: yes, and make it the same code path as search with an empty query. So the CLI has add, search, and list, and the core has one search endpoint that takes optional words and an optional date range.

**User:** yes third command

**Agent:** Question 18: What does the Chrome popup show when it opens on a page you have not saved yet?

Recommendation: URL and title pre-filled but editable, a note box with the cursor in it, and a Save button. Enter saves, Escape closes. After saving it shows "saved" for a second and closes itself. Nothing else: no tags field until tags arrive, no search box, since the web page is one click away.

**User:** ok

**Agent:** Question 19: And when the popup opens on a page you already saved?

Recommendation: the popup does a lookup on open and, if the URL is known, shows "saved on 12 March" with the existing note pre-filled in the editable box. The button reads Update instead of Save. So the duplicate flow in the browser is invisible: you never see a conflict, the popup just opens in the right mode.

**User:** ok

**Agent:** Question 20: Does the Telegram channel run inside the core's process, or as its own deployable?

The design says channels only capture and the core owns the store. That is a code boundary. It does not have to be a process boundary.

Recommendation: same process, same Fly app. The Telegram adapter is a webhook endpoint in the core that talks to the core's capture operation through the same interface the HTTP API uses, never touching the store directly. One deploy, one URL, one set of secrets, and Telegram's webhook needs a public HTTPS URL which the Fly app already has. If a channel ever needs a different runtime, it can move out, because it only ever spoke to the capture interface.

**User:** same process works and is simple

**Agent:** Question 21: When the bot asks "add note?", where does the waiting state live?

Between the bare URL and your reply, someone has to remember which item the reply belongs to. Fly restarts the app on every deploy, so in-memory state can vanish mid-conversation.

Recommendation: no waiting state at all. The bot saves the item immediately with an empty note, then asks "add note?". Your next non-URL message within, say, ten minutes updates that most recent item's note. The "most recent item awaiting a note" pointer is one row in the database, so a restart loses nothing, and the worst case is an item saved without a note, which is still better than losing the link.

**User:** ok

**Agent:** Question 22: How is the data backed up?

SQLite on one Fly volume means one disk. Volumes can be lost.

Recommendation: a daily export the core does itself. A scheduled job inside the app writes the whole collection as a JSON file and uploads it to a bucket, Cloudflare R2 or a similar S3-compatible one, keeping the last thirty. Plus an "export everything as JSON" endpoint on the API, so the CLI can pull a copy to your laptop whenever you want. Fly's own volume snapshots exist too, but I would not rely on them alone. The JSON export also doubles as the migration path if the store ever changes.

**User:** don't need this yet for mvp, let's plan it for later. plan your recommendation for something in backlog

**Agent:** Noted for the backlog: daily JSON export to a bucket plus an export endpoint.

Question 23: Is delete a hard delete?

Recommendation: yes, the row is gone. Soft delete only matters if you might undelete, and with a single user clicking a button on the web page, a confirm dialog covers that. One caveat: a hard delete frees the URL, so saving it again later creates a fresh item, which is what you would expect.

**User:** yes

**Agent:** Question 24: Is the core built as hexagonal, with the domain in the middle and every I/O behind a port?

Concretely: a core library with the item model, URL normalization, the capture and search operations, and ports for the store, the title fetcher, and the clock. Adapters around it: SQLite store, HTTP title fetcher, the JSON API, the Telegram webhook, the static web page. Tests drive the core through in-memory fakes and run in milliseconds, with a small number of adapter tests against a real SQLite file and a stubbed HTTP server.

Recommendation: yes, and grown outside-in with TDD, starting from a walking skeleton that saves one item and searches for it through the HTTP API. This repo is a learning-hour starter, so I assume that is the point, but tell me if you want something looser.

**User:** yes I was thinking hexagonal will fit this well

**Agent:** Question 25: What is the project layout in the solution?

Recommendation: rename away from the ActivePartner placeholder to whatever you want to call the product, and have these projects under csharp/:

- Core: domain, ports, operations. No framework references.
- Web: the ASP.NET host with the SQLite adapter, JSON API, Google login, Telegram webhook, and the static search page. This is what gets deployed to Fly.
- Cli: the command line client. References nothing but an HTTP client.
- Core.Tests, Web.Tests, Cli.Tests.

The Chrome extension is plain JavaScript in a chrome-extension/ folder at the top level, since it has no build.

Do you have a name for the product yet?

**User:** do you have any ideas for names?

**Agent:** A few, grouped by flavour:

- Keep or Kept: the verb is the whole product. Short CLI: keep add, keep search. Google has a Keep, but for a private tool that hardly matters.
- Stash: same idea, no name clash, and stash is a nice CLI word.
- Pin: what the Chrome button does.
- Remind, Recall: about the moment you get the link back rather than the moment you save it. recall search flaky tests reads well.
- Later: honest about what it is for. "Send it to Later."
- Tabless: a joke about the origin story, and it would make you smile every time you closed a tab.

My pick is Recall. It names the part you care about, searching, not the part that was already easy, saving. The CLI reads as a sentence, the bot can be "Recall bot", and it does not collide with anything you type daily.

Which one, or do you want another round?

**User:** stash is nice

**Agent:** Stash it is: projects Stash.Core, Stash.Web, Stash.Cli and their tests, CLI command stash.

Question 26: How is the walking skeleton defined?

The first end-to-end slice, before any channel exists, sets the shape of everything after it.

Recommendation: the Web app running locally with SQLite, no login yet, exposing save and search over JSON. One acceptance test starts the app in-process, saves an item with a URL and note, searches by a word from the note, and gets the URL back. Then a second test saves the same URL again and expects the conflict answer. Once that passes and deploys to Fly, the channels get added one at a time in this order: web page, Chrome extension, CLI, Telegram, with Google login just before the first deploy that exposes the page.

**User:** that should work yes

**Agent:** Question 27: How does the code get to Fly?

Recommendation: manual fly deploy from your laptop for the MVP, with a Dockerfile in the repo that builds and publishes the Web project. No CI pipeline yet. The tests run locally before you deploy, and one person deploying by hand a few times a week does not need automation. GitHub Actions deploying on push to main goes in the backlog next to backups. Secrets, that is the Google OAuth client, the API token, and the Telegram bot token, live in Fly secrets, never in the repo.

**User:** manual works for now but let's add github cli pipeline in the backlog

**Agent:** Backlog: GitHub Actions pipeline.

Question 28: How is the Chrome extension installed?

Recommendation: loaded unpacked from the repo folder in developer mode, on your machine only. No Chrome Web Store listing, no packaging, no signing. Updating it means pulling the repo and clicking reload on the extensions page. The core's URL is a single constant in the extension's options page so the same code works against local and Fly. Publishing to the store is a backlog item that only matters if you want it on a second machine without the repo.

**User:** yes it's a personal app so that's fine

**Agent:** Question 29: What are the exact URL normalization rules?

The unique index depends on these, and changing them later means reindexing.

Recommendation: lowercase the scheme and host, treat http and https as the same, drop a leading www., drop the fragment, drop a trailing slash, remove query parameters whose name starts with utm_ plus fbclid and gclid, and sort the remaining query parameters. Keep everything else, including the path case, since many sites are case-sensitive there. The original URL is stored untouched and is what you get back; the normalized form is only the key.

**User:** go with recommendation

**Agent:** Question 30: Which timestamps does an item keep, and in what timezone?

Recommendation: two, created and last updated, both stored as UTC. Date filters in search compare against created, since "when did I see this" is the question you ask. The web page and CLI show them in your local time. No timezone setting anywhere in the core.

**User:** Always store in UTC but display in berlin timezone

**Agent:** Display fixed to Europe/Berlin regardless of device, stored as UTC.

That was the last open branch. Here is the shared understanding as I have it.

Product: Stash, a single-user link-and-note memory. C# on .NET 10, hexagonal, grown outside-in with TDD, hosted on Fly.io with SQLite on a volume.

Item: optional URL, stored as typed plus a normalized key with a unique index on it, title, one editable note, created and updated timestamps in UTC shown in Berlin time. Tags have a place in the model but are not in the MVP.

Duplicate rule: insert fails on the unique index, core answers "conflict, here is the existing item," and the channel offers an update with the old note pre-filled. In Telegram the reply is appended to the old note.

Search: all typed words must appear in note, title, or URL, case-insensitive, prefix-tolerant, optional created-date range, newest first. Empty query is list.

Core API: save, update, lookup by URL, search, delete. Title is fetched by the core when a channel does not supply one, synchronously with a short timeout.

Auth: Google sign-in restricted to your email for the web page and, via the same cookie, the Chrome extension. Bearer token for the CLI. Telegram webhook secret plus your chat ID.

Channels:

- Web page: one static page, vanilla JavaScript over the JSON API, search box, date range, edit note, delete with confirm.
- Chrome extension: loaded unpacked, lookup on open, Save or Update mode, URL and title pre-filled.
- CLI: stash add, stash search, stash list, editor for notes, config file in home.
- Telegram: runs inside the core process, first URL becomes the URL, bare URL saves immediately and asks "add note?", next message within ten minutes fills the note, pointer kept in the database.

Delivery: walking skeleton is save plus search over JSON with a conflict test, then web page, Chrome, CLI, Telegram, with Google login before the first public deploy. Manual fly deploy with a Dockerfile, secrets in Fly.

Backlog: tags, automatic tags, daily JSON backup to a bucket plus export endpoint, GitHub Actions deploy, semantic search, Chrome Web Store packaging.
