# nuTube

_The algorithm, on the device_

---

**Download:** there are no releases yet. Build from source (installation and usage, below) and sideload `nutube.apk`. Google Play is not an option - see the unresolved question below.

---

## description

nuTube is a local-first way to explore video platforms, starting with YouTube. Nothing is hosted here: nuTube is a pure Android client that pulls from sources it does not own. The recommendation algorithm lives on the device. There is no remote ranking service - nuTube keeps a small index locally and ranks it with simple, inspectable rules, so every card can tell you why it surfaced. Every search you run becomes a term, following a channel is just one more term, and the index is whatever those terms have gathered. If it works for YouTube, the shape extends: every source plugs in behind one generic `FeedItem`, and a single local ranker decides what to show across all of them.

<details open>

<summary>unresolved: ads, views, and who pays for the video</summary>

**This is an open design question, not a settled position. Read it before building on this.**

nuTube reaches YouTube through NewPipeExtractor, which speaks InnerTube - the private API the YouTube site and apps use - rather than the YouTube Data API. That means no key, no quota, and no account. It also means three things that have not been resolved:

- **Ads are not shown, and cannot practically be.** The extractor returns content stream URLs directly; YouTube's ads are separate streams plus tracking beacons that must fire against Google's ad infrastructure. Skipping them is not a feature this app implements, it is what not implementing ads looks like. Building them would not help either - ad revenue only flows when an impression is reported and trusted, so the result would be the burden of ads with none of the payout.
- **Views very likely do not count.** View counting depends on playback progress pings (`videostats_playback_url` and friends in the player response) that this client does not send. A creator gets no credit for a view through here. That is the same for every keyless client, and it is the part of this design that is hardest to feel good about.
- **Google Play is not an option.** Third-party YouTube clients are removed for circumventing monetization, independent of the terms-of-service question. Distribution is sideload, F-Droid or Obtainium. Note also Android's developer verification rollout: enforcement began 2026-09-30 in Brazil, Indonesia, Singapore and Thailand and expands globally in 2027, after which unverified apps on certified devices install only through an advanced flow with a 24-hour wait. `adb install` is unaffected.

**How this is handled today.** Playback is a per-platform setting, and **embed mode is the default**. Embed mode loads YouTube's own IFrame player - the sanctioned embedding API - and gets out of its way: no JS bridge, no `enablejsapi`, nothing that reads or rewrites what the page loads. Its ads run, its view is counted, its creator is paid. Native mode is opt-in per platform under Settings, and its description states plainly what it takes away.

That does not make the app submittable to Google Play - review looks at what an app can do, not what it defaults to - and it does not settle the question. It just puts the choice with the person who is actually party to YouTube's terms.

</details>

<details>

<summary>features</summary>

- **Search across sources.** A query fans out through `SourceRegistry` to every registered platform concurrently and folds the results into the local index. A source that fails is dropped, so a rate-limited platform degrades instead of breaking the feed.
- **A feed that explains itself.** Every card names the single rule that did the most to rank it, and carries its upload date.
- **Terms.** Every search is saved as a term and credited with what it surfaced. The Terms tab lists them: tap one to re-run it, or remove it to drop it and everything only it was holding.
- **Following.** The Follow button in the player writes a `channel:` term, which pages through a channel's entire back catalogue. There is no separate subscription list.
- **Two playback modes, per platform.** Embed mode (the default) runs the platform's own player. Native mode adds hardware decode, HD from merged video and audio tracks, and picture-in-picture when you leave the app.
- **Mini player.** Backing out of a native video docks it above the tab bar so it keeps playing while you browse; tap to expand, X to stop.
- **Share to index.** Sharing or opening a YouTube link from any other app adds it to the index.

Three tabs sit at the bottom - Feed, Terms and Settings. Closing a video uses the system back gesture rather than a button.

</details>

<details>

<summary>subsystems</summary>

## how ranking works

Scoring is a set of separable rules in [`core/.../ranking/`](core/src/commonMain/kotlin/eco/src/nutube/core/ranking/), each returning 0..1 plus a sentence explaining itself. `Ranker` takes a weighted sum for the order but takes the *explanation* from whichever single rule contributed most, so a card names the actual reason it beat the others rather than summarising arithmetic.

- `QueryOverlapRule` - overlap with the search box. Stands down entirely when the box is empty, letting the affinity rules decide.
- `ChannelAffinityRule` - how often you have opened videos from that channel, scaled against your most-watched channel rather than an absolute count.
- `TitleTermRule` - subject matter learned from titles you have opened, as unigrams and bigrams over content words. Averaged across the title's own grams so a long title cannot win by containing more words.

The only training signal is opening a video. Counts live in `affinity.json` as two maps of integers - nothing latent, readable by a person, which is the whole premise of the algorithm living on the device.

Adding a signal - watch time, recency, co-occurrence, an on-device embedding - means adding an entry to `RULE_REGISTRY`, not editing a scoring function.

## how the index gets built

**Every search you run is saved as a term**, and the term is credited with whatever it surfaced. Videos reachable from a surviving term stay when a term is removed, and videos added by hand - a shared link - are never evicted, because no term put them there.

Following a channel is the same mechanism. `channel: <name>` asks a platform for that channel's uploads rather than running a keyword search, and pages to the end of the back catalogue rather than stopping at a page. There is no second storage, and the crawler will treat a follow exactly like a search.

Channel identity is exact on purpose. A display name is not unique - several channels are called "Arc" - so `channel:` accepts an `@handle`, a channel id or a channel URL and resolves those directly, falling back to a name search only when given nothing better. Follow always stores the canonical channel, and follow state is matched by channel identity rather than by term text, since the same channel can legitimately be written several ways.

The term list is the input a crawler will eventually re-run on a schedule to keep the index fresh without anyone searching. The crawler is not written yet; today terms only run when you search, tap one, or follow a channel.

## playback

Embed mode loads the platform's IFrame player in a WebView and leaves it alone; the page owns its own player, so leaving the app pauses it instead of entering picture-in-picture.

Native mode plays through ExoPlayer. HD works by taking YouTube's separate video-only and audio-only tracks and merging them with `MergingMediaSource` - the muxed stream YouTube also offers stops at 720p and is kept only as a fallback.

## adding a platform

Implement `VideoSource` and register it in `NuTubeApp.onCreate`. Everything above the interface - the index, the ranking, the feed, the player - is written against `FeedItem` and never learns which platform an item came from.

</details>

<details>

<summary>layout</summary>

Kotlin Multiplatform, split so that a second platform is an added target rather than a rewrite.

- **[`core/`](core/)** - The multiplatform module, `commonMain` only: `FeedItem`, `LocalIndex` (the on-device store), `ranking/` (the rules and the `Ranker`), `TermBank`, `VideoSource` (the contract every platform implements), `SourceRegistry` (which platforms are plugged in), `Settings` and `PlaybackStreams`. Nothing here touches an Android or JVM API; it uses okio rather than `java.io` for exactly that reason. Adding `iosArm64()` or `jvm()` to `core/build.gradle.kts` is the whole change needed to run this logic elsewhere.
- **[`app/`](app/)** - The Android application: the Compose UI (`ui/`), the embed and ExoPlayer players, and `sources/youtube/`, which implements `VideoSource` over NewPipeExtractor.
- **[`gradle/libs.versions.toml`](gradle/libs.versions.toml)** - Every dependency version, in one place.
- **[`run.sh`](run.sh)** - Build, install and launch on an emulator or a connected phone.
- **`nutube.apk`** - The last debug build, copied to the root by `run.sh` (git-ignored).

</details>

<details>

<summary>installation and usage</summary>

## building

Needs JDK 17 and the Android SDK. No Android Studio required - the Gradle wrapper does everything. `local.properties` points at the SDK and is git-ignored, so it needs writing once per machine:

```sh
echo "sdk.dir=$HOME/Android/Sdk" > local.properties
```

Then:

```sh
./run.sh --apk        # build only; drops nutube.apk in this directory
./run.sh              # boot the emulator, install, launch, tail logs
./run.sh --url <link> # launch through a VIEW intent, to exercise the deep link
./run.sh --help       # every flag
```

`run.sh` uses whatever device is already connected, so with a phone plugged in it installs there instead of booting the emulator. `NUTUBE_SDK` and `NUTUBE_AVD` override the SDK path and the emulator image.

## installing

```sh
adb install nutube.apk
```

`nutube.apk` sits at the project root rather than under `app/build/` so that file-sync tools which skip build directories (Syncthing, here) can still carry it to a phone.

</details>

<details>

<summary>history</summary>

## why Kotlin, and what happened to the Godot version

nuTube started as a Godot prototype inside [praxis](https://github.com/0-5788719150923125/praxis), under `axis/nutube/godot/`; it was removed in `fd187a1c` and survives in that repository's history. It worked, and the two things people worry about with a game engine - idle battery drain and UI lag - both turned out to be solvable. Godot can genuinely stop redrawing when idle (`OS.low_processor_usage_mode` plus `RenderingServer.viewport_set_update_mode(..., VIEWPORT_UPDATE_DISABLED)` on the root viewport takes GPU draw calls to zero, measured).

What killed it was everything an aggregator needs that lives on the JVM:

- **Playback.** Godot's `VideoStream` has exactly one subclass, `VideoStreamTheora`. YouTube serves DASH with separate VP9/AV1/H.264 video and audio. ExoPlayer handles it, hardware-decoded, on a surface Android can hand straight to the display controller.
- **Extraction.** oEmbed gives a title and an author and nothing else - no search, no channel listings. Real extraction means running YouTube's player JavaScript to decipher stream URLs, and GDScript has no JS engine. NewPipeExtractor does it with Rhino.
- **Background indexing.** Android has had no background execution without `WorkManager` or a foreground service since API 26, and neither is reachable from GDScript. A crawler that runs while the app is closed has to be Kotlin.

Each of those alone forced JVM code into the project. Together they meant Godot would have been a UI shell over a Kotlin app.

</details>

## limitations

- **The background crawler is not built.** `WorkManager` is a dependency but unused; terms only run when you search, tap one, or follow a channel.
- **No background audio.** The manifest and permissions are in place, the `MediaSessionService` is not.
- **Persistence is JSON files** - `index.json`, `terms.json`, `settings.json` and `affinity.json` in the app's files directory.
- **The ranker's only signal is opening a video.** Watch time, recency and embeddings are future rules.
- **NewPipeExtractor is a Java library**, so the YouTube source can only ever live in a JVM target. An iOS build would share the index, the ranker and the registry, but would need its own extraction path and its own player. That is a real limit on how much Kotlin Multiplatform buys here, and it is worth knowing before betting on it.
