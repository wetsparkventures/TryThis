# TryThis: plan

Written 2026-11-06 for Jack Biltis from his brief of the same day. "TryThis" is the working name
(see §12 before launch). Audience: Jack, deciding what to build first, and later whoever builds it.

## 1. The idea, in Jack's words

A project "that allows users to store their favorite movies, books, songs, podcasts, youtube clips,
comedians, music bands, tv shows, restaurants, plays, art, architecture, clothing, snacks, ebay
items, cars, travel, presents, etc and the system will recommend other items that they would like.
The system makes its money from reference links to amazon and other online locations but does not
limit its recommendations based on revenue. It also builds custom youtube, spotify, and instagram
playlists based on these preferences. Users can also add movies, shows, goals (catch up on every
Elvis movie, Oscar winning movie...) or educational goals (learn to fly, get a working knowledge of
classic art or the theory of relativity) for the system to curate items and build quizzes."

## 2. What exists and what is different

Every recommender today lives inside one category and one store: Netflix knows your films, Spotify
your music, Goodreads your books, Amazon what you bought from Amazon. Cross-category apps
(TasteDive, Likewise) stop at "if you like this band you may like this film" and sell placement.
Nothing holds a person's whole taste in one place, explains itself, and turns "I want to see every
Best Picture winner" or "understand relativity" into a plan with a quiz at the end.

Three things make TryThis different, and all three are product rules, not slogans:

1. **One taste, every category.** A favorite is a favorite whether it is a film, a restaurant or a
   pair of boots. The engine learns *why* (dry humor, 1970s, Americana, Italian, minimalism) and
   carries that across categories.
2. **Revenue never touches ranking.** Recommendations are ranked first; affiliate links are attached
   afterward, by a separate step that cannot see or change the order. The code enforces this (§6.4),
   and every money link is labeled.
3. **Goals become curricula.** A completionist goal ("every Elvis movie") expands from a real catalog
   into a checklist with progress. An educational goal ("working knowledge of classic art") becomes
   an ordered syllabus of books, videos, courses and visits, with quizzes that track what stuck.

## 3. Who it is for and what they do

| Person | What they do in TryThis |
|---|---|
| The collector | Logs favorites as they come up: "that was great, save it." Rates, adds a line on why. |
| The stuck one | Asks "what should I watch / read / eat / listen to next" and gets a short list with reasons. |
| The planner | Sets a goal, gets a plan, checks items off, takes quizzes, sees progress. |
| The gift giver | Opens a friend's shared profile (or builds one from what they know) and asks for presents. |
| The playlist listener | Presses one button and a Spotify or YouTube playlist appears in their own account. |

## 4. Scope

### 4.1 Kinds of item

One catalog, one `Kind` table (the single source for the category list; nothing else hard-codes it).
Each kind names where its items come from and what a favorite of that kind can link to.

| Kind | Catalog source (search and details) | Money link |
|---|---|---|
| Movie, TV show | TMDB (free API, attribution required) | Amazon (buy/rent), JustWatch-style "where to stream" |
| Book | Open Library + Google Books | Amazon, Bookshop.org |
| Song, album, band | Spotify Web API (search and metadata), MusicBrainz as fallback | Amazon Music, merch later |
| Podcast | Podcast Index (free, open) + Apple Podcasts lookup | none (free content) |
| YouTube clip, channel | YouTube Data API | none |
| Comedian | Wikidata + YouTube specials + Spotify albums | Ticket links later (Ticketmaster affiliate) |
| Play, musical | Wikidata + IBDB-style lookups | Ticket links later |
| Restaurant | Google Places API | OpenTable affiliate, Uber Eats later |
| Art, architecture | Wikidata + museum open APIs (Met, Art Institute of Chicago, Rijksmuseum) | Prints and books on Amazon |
| Clothing, snack, eBay item, present | Amazon Product Advertising API, eBay Browse API | Amazon Associates, eBay Partner Network |
| Car | NHTSA vPIC (makes and models), Wikidata | Autotrader/Edmunds affiliate later |
| Travel | Wikidata places, Google Places | Expedia or Booking affiliate |
| Anything else | Free-text item the user types; the engine still extracts facets | Skimlinks (turns ordinary store links into affiliate links) |

Phase 1 ships the first six rows (film, TV, books, music, podcasts, YouTube) plus restaurants; the
rest arrive in phase 3 (§9). "Anything else" exists from day one so no favorite is refused.

### 4.2 Favorites

A favorite is: an item, a rating (1 to 5, optional), a free-text "why I like it" (optional but
prompted, because it is the richest signal), tags the user chooses, and the date. Users can also mark
**dislikes** and **already have / already seen** so recommendations skip them. Import: Letterboxd,
Goodreads and Spotify "liked" exports in phase 2.

### 4.3 Recommendations

"What next?" per kind, or across kinds ("surprise me"). Each recommendation shows **why** ("because
you liked *The Sting* and *Ocean's Eleven*: heist, 1970s, ensemble"), **where** (stream, buy, visit,
listen) and three buttons: save it, not for me, already know it. Every answer feeds back into the
taste profile.

### 4.4 Playlists

- **Spotify**: creates a playlist in the user's own account (OAuth, `playlist-modify-private`),
  named and described by TryThis, refreshed weekly if the user opts in.
- **YouTube**: same, through the YouTube Data API (`playlists.insert`, `playlistItems.insert`).
- **Instagram**: Instagram has no API for creating collections or guides for a user (Guides were
  removed in 2023; saved collections are read-only to third parties). What TryThis can do: build a
  list of accounts, hashtags and reels to follow from the taste profile, show it as a page the user
  taps through to Instagram, and post a shareable "my taste" card to the user's own feed with their
  permission. The plan calls this an **Instagram guide** and does not promise a playlist.

### 4.5 Goals and quizzes

Two goal types, same shape (an ordered list of items with status):

- **Completionist**: "every Elvis movie", "every Best Picture winner", "every Cormac McCarthy
  novel", "every Michelin-starred restaurant in Arizona". The engine resolves the set from a real
  catalog (Wikidata holds Oscar winners, filmographies and bibliographies), orders it (release date
  by default), marks what the user already logged, and tracks progress.
- **Educational**: "learn to fly", "working knowledge of classic art", "the theory of relativity".
  The engine drafts a syllabus in levels (orientation, foundations, depth, practice), each level a
  mix of books, lectures, videos, podcasts, museum visits or, for flying, real-world steps (discovery
  flight, written test), every item a catalog item so it can be saved, linked and bought. The user
  edits the syllabus; it is theirs.

**Quizzes** attach to educational goals: short, generated from the items the user has marked done
(so the questions are about what they read, not a generic test), multiple choice plus one written
answer graded by the model, with spaced repetition of missed questions. Progress is shown per level.

### 4.6 Sharing and gifts

A profile can be shared as a read-only link. Anyone with the link (signed in or not) can ask for
present ideas "for this person", in a price range, for an occasion. The profile owner controls which
kinds are visible on the share.

## 5. Recommendation engine

The engine has four parts, and the first is the one that matters.

1. **Facets.** When a favorite is saved, the model (Claude) reads the item's catalog metadata and the
   user's "why" and emits a short set of facets: genre and style words, era, mood, themes, people,
   places, craft attributes. Facets are stored once per item (`ItemFacet`) and rolled up per user
   (`UserFacet`, a cache with weights; the roll-up is the only thing that writes it). Facets are what
   let a love of film noir surface a jazz album and an art deco building.
2. **Candidates.** For a request, gather candidates three ways: catalog similarity (items sharing
   the user's strongest facets, via pgvector embeddings of item facet text), the model's own
   suggestions (asked for items, then resolved against the catalogs so nothing invented is shown),
   and, once there are enough users, "people whose favorites overlap yours also like" (collaborative
   filtering; phase 3).
3. **Ranking.** Score by facet match, diversity (no five sequels), novelty against what the user
   already logged or rejected, and recency of their taste. The ranker reads taste data only.
4. **Explanation.** The model writes the one-line "because you liked…" from the facets that matched,
   so every reason is traceable to a stored favorite.

Cold start: the first screen asks for five favorites in any kind, then shows recommendations
immediately; the first recommendations are honest about being rough.

## 6. Revenue

### 6.1 Affiliate programs

| Program | Covers | Terms worth knowing |
|---|---|---|
| Amazon Associates (Product Advertising API 5) | Books, film, music, clothing, snacks, presents, anything Amazon sells | Must make 3 qualifying sales within 180 days or the account closes; prices may be cached 24 hours at most; "As an Amazon Associate I earn from qualifying purchases" must appear |
| eBay Partner Network (Browse API) | eBay items, collectibles, cars, parts | Standard; links expire and must be regenerated |
| Bookshop.org | Books, for users who prefer independents | 10% commission |
| OpenTable | Restaurants | Per seated reservation |
| Expedia Group / Booking.com | Travel | Per booking |
| Ticketmaster | Comedians, plays, bands | Per ticket |
| Skimlinks or Sovrn | Everything else with a store page | Converts ordinary retailer links; takes a cut |

### 6.2 What is not for sale

No placement, no sponsored items, no selling or renting taste data, no ad network. The business is
the affiliate cut on things people were already recommended for taste reasons. A paid tier (phase 3)
can remove the affiliate links entirely for people who want that, add unlimited goals and family
profiles.

### 6.3 Disclosure

Every money link carries a visible "affiliate link" label and the Amazon sentence appears on every
page that has one (FTC endorsement guides and Amazon's operating agreement both require it).

### 6.4 The firewall, in code

`recommend()` returns an ordered list of items and reasons and has no import from the `money/`
module. `decorate()` then takes that list and attaches links. A test asserts the two modules do not
reference each other, and a second test checks that the order out of `decorate()` equals the order
in. That is the whole guarantee, and it is checkable.

## 7. Architecture

Same shape as the other WetSpark web products (Predictable, YourBallot), so the tooling and the
deploy path are known:

| Layer | Choice | Why |
|---|---|---|
| Runtime | Node 24+, TypeScript | Matches Predictable and YourBallot |
| Web | Server-rendered pages with a small amount of client script; a PWA so it installs on a phone. Native apps later only if the PWA is not enough | One codebase, no app-store gate for v1 |
| Database | Postgres with pgvector (Fly Postgres or Neon) | Relational data plus embeddings in one place |
| Model | Claude (Sonnet 5.5 for facets, explanations, syllabi, quiz generation and grading; Haiku 5.5 for cheap classification) | |
| Hosting | Fly.io app `trythis`, Cloudflare in front | Jack's standard (`Jack/CLAUDE.md`) |
| Sign-in | Email magic link, Google, Apple; password sign-in with a Show Password option | Spotify and YouTube need their own OAuth grants on top |
| Jobs | A worker process on Fly for facet extraction, catalog refreshes, weekly playlists and the 24-hour Amazon price expiry | |
| Secrets | Locally in Keychain items `TryThis <service>`; on Fly as secrets | Working.md rules |

External APIs and their gates: TMDB (free key), Open Library (none), Google Books and Places and
YouTube (one Google Cloud project; YouTube has a 10,000 unit/day quota, enough for thousands of
users), Spotify (developer app; extended quota needs a review once past 25 users, and **Spotify's
developer terms forbid using Spotify content to train AI models**, so Spotify metadata is used only
to identify items and build playlists, never fed into model training), Podcast Index (free key),
Amazon PA-API (needs an approved Associates account first), eBay (developer account, Browse API).

## 8. Data model

Follows `SQL.md`: proper-case names, one capital per word, meaningful keys, each fact stored once.

| Table | Key | Holds |
|---|---|---|
| `Kind` | `KindCode` | The category list (§4.1) and each kind's catalog source. The only place categories are defined |
| `Item` | `KindCode, Source, SourceId` | One catalog entry (a film from TMDB, a place from Google). Title, year, creators, image, metadata JSON, embedding |
| `Person` | `PersonId` (surrogate: nothing natural is stable) | Account, sign-in methods, settings |
| `Favorite` | `PersonId, KindCode, Source, SourceId` | Rating, Why, Stance (like / dislike / have), AddedOn |
| `Facet` | `FacetCode` | The vocabulary of taste words, grown by the model, merged by hand when it drifts |
| `ItemFacet` | item key + `FacetCode` | Weight |
| `PersonFacet` | `PersonId, FacetCode` | Weight. A **cache** rolled up from Favorites by one job; nothing else writes it |
| `Recommendation` | `PersonId, KindCode, Source, SourceId, ShownOn` | Score, Reasons JSON, Outcome (saved / rejected / known / ignored) |
| `Goal` | `GoalId` | PersonId, Title, GoalType (completionist / educational), Status |
| `GoalItem` | `GoalId, Sequence` | item key, Level, Status, DoneOn |
| `Quiz` | `QuizId` | GoalId, Level, CreatedOn |
| `QuizQuestion` | `QuizId, Sequence` | Prompt, Choices JSON, Answer, source item key |
| `QuizAnswer` | `QuizId, Sequence, PersonId, AnsweredOn` | Given, Correct, NextReviewOn |
| `Playlist` | `PersonId, Platform` | PlatformPlaylistId, LastBuiltOn, AutoRefresh |
| `MoneyLink` | item key + `Merchant` | Url, FetchedOn (Amazon rows expire after 24 hours) |
| `MoneyClick` | `PersonId, item key, Merchant, ClickedOn` | For reporting what earns; never read by ranking |
| `Share` | `ShareCode` | PersonId, visible kinds, CreatedOn, RevokedOn |

## 9. Phases

| Phase | Delivers | Done when |
|---|---|---|
| **0. Plan** (this document) | Folder, repo, plan, name check | Jack approves the plan and the first-phase scope |
| **1. Taste, recommendations and playlists** | Sign-in; add favorites in film, TV, books, music, podcasts, YouTube, restaurants and free text; facets; "what next" per kind and across kinds with reasons; Amazon and Bookshop links behind the firewall; shareable profile; Spotify and YouTube playlists in the user's own account (Jack widened phase 1 to include them, 2026-11-06); PWA on Fly | Jack and the family use it for two weeks, the recommendations are worth acting on and a playlist lands in Spotify and YouTube |
| **2. Goals** | Instagram guide page; completionist goals from Wikidata; educational goals with syllabi and quizzes; Letterboxd, Goodreads and Spotify imports | A goal of each type runs end to end, quiz included |
| **3. Every kind, and money** | Remaining kinds (art, architecture, comedians, plays, clothing, snacks, eBay, cars, travel, presents); gift mode; eBay, OpenTable, travel and ticket affiliates; collaborative filtering; paid tier; app-store wrappers if wanted | Public launch under the final name |

Each phase is a few weeks of sessions, not months; phase 1 is the one to size carefully and ship.

## 10. Risks and rules

- **Platform terms.** Spotify (no AI training on its data, playlist creation only for the signed-in
  user), YouTube (API services terms: no stored data beyond 30 days except user-authorized, quota),
  Amazon (sales minimum, caching limit, disclosure), Google Places (no storing results beyond
  place IDs). Each integration gets a one-paragraph terms note in `INTEGRATIONS.md` before it ships.
- **Privacy.** Taste data is personal (it can reveal religion, politics, health, sexuality). Plain
  privacy policy, no third-party trackers, export and delete on demand, 13+ only.
- **Model honesty.** Every recommended item must resolve to a catalog entry; nothing invented is
  shown. Syllabi cite the item they point at. Quizzes are generated only from items marked done.
- **Cost.** Facet extraction is the main model cost: roughly one short call per favorite and one per
  recommendation request. At Sonnet prices that is a few cents per active user per month; the
  worker batches and caches to keep it there.
- **Name.** See §12.

## 11. Identity and WetSpark

A WetSpark product. Portfolio entry (Jack, 2026-11-06): status `lab`, identity `dba` (its own domain later; no
texting campaign needed), public on the site from the start, no announcement. Integrations (Google Cloud
project, Spotify app, Amazon Associates, eBay developer account) are opened under WetSpark's
identity per `WetSpark/INTEGRATIONS.md`, using info@wetspark.com, and recorded there.

## 12. The name

"TryThis" is the working name. Checked 2026-11-06 (the naming rule: check the neighbors):

- trythis.com, trythis.co, trythis.ai and trythisapp.com are all registered; try-this.com too.
- An Italian social "challenge your friends" app called TryThis existed around 2020, and
  "TryThisOn" is a live virtual try-on app in the App Store.
- No USPTO search done yet; do it before launch.

Recommendation: keep TryThis as the folder and working name (like Sandbox), and choose the launch
name at the end of phase 1 when the product is in hand. Candidates can be checked then.

## 13. Decisions (Jack, 2026-11-06)

1. Phase 1 is the wider version: the planned scope plus Spotify and YouTube playlists (§9).
2. Instagram: the guide page stands in for a playlist (§4.4). Agreed.
3. The repo is public.
4. WetSpark portfolio: `lab`, public on the site, no announcement.

## 14. Branches and pull requests

| Repo | Branch | Pull request | State |
|---|---|---|---|
| [wetsparkventures/TryThis](https://github.com/wetsparkventures/TryThis) (public) | main | none (Jack's own repo: commits go straight to main) | plan only |
