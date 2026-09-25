---
name: onboard
description: List a browser-playable space (game, world, tool, wonder) on Velven, the community marketplace for spaces built with AI. Use after deploying such a project when the creator says "list on Velven", "submit to Velven", "put this on Velven", "publish my game/space/world to the leaderboard", asks how to get plays and a ranking for something built with AI, or wants to claim or verify a space that is already on Velven ("claim my space", "it says unclaimed", "verify I made this"), or asks to "add a leaderboard", keep "high scores", "sign in players", "save progress" across devices or use the "Velven SDK" in a space. Handles the device-code login (the creator clicks one link and presses Approve), puts the ownership proof on the site, fills provenance and the page text from the project, submits the URL or claims the existing listing, adds the leaderboard, sign-in and saves through the SDK when the space keeps a score or progress, and replies with the Velven link and badge.
---

# Velven

Velven (https://velven.ai) is the community marketplace for spaces built with AI.
A space is anything interactive in a browser. It stays where it is hosted; Velven
lists it, counts plays and ranks it. You list it for the creator: no API key is
ever copied by hand. Full docs: `curl -s https://velven.ai/docs/agent.md`.

## When to use

- You just deployed a browser-playable project that was built with AI (any tool)
  and the creator wants it listed, or asks about a leaderboard, plays or ranking.
- Not for: backend services, CLIs, non-interactive pages, or work not built with AI.

Set `VELVEN=https://velven.ai` (or the base URL of a self-hosted instance).
Your shell may not keep variables between commands, so set `VELVEN` and
`TOKEN=$(cat ~/.config/velven/token)` in each command that uses them.

## Step 1: reuse or obtain a token

Tokens live at `~/.config/velven/token` (mode 600). Check first:

```bash
TOKEN=$(cat ~/.config/velven/token 2>/dev/null)
curl -s "$VELVEN/api/agent/me" -H "authorization: Bearer $TOKEN"
# 200 {"handle":"mara","profile_url":"https://velven.ai/mara"}  -> skip to step 3
# 401 -> delete the file and log in below
```

Login: one unauthenticated call returns a link for the creator and a code for
you. The code goes straight to a file; never print `device_code`.

```bash
login=$(curl -s -X POST "$VELVEN/api/agent/login" \
  -H "content-type: application/json" \
  -d '{"agent_name":"Claude Code"}')
(umask 077 && mkdir -p ~/.config/velven && \
  printf '%s' "$login" | jq -r '.device_code // empty' > ~/.config/velven/device_code)
printf '%s' "$login" | jq 'del(.device_code)'
# 201 {"user_code":"K7PX-4MDQ",
#      "verification_url":"https://velven.ai/agent/approve?code=K7PX-4MDQ",
#      "expires_in":900,"interval":5}
```

Use your own name as `agent_name` (max 60 chars). A 429 `rate_limited` means
this address opened too many logins; wait 15 minutes rather than retrying in
a loop.

## Step 2: the creator approves, you poll

Tell the creator, in one short message:

> Open https://velven.ai/agent/approve?code=K7PX-4MDQ and press Approve.
> The code on the page should read K7PX-4MDQ. If you are not signed in to
> Velven yet, the same page signs you in and asks for a handle. I will wait.

Then poll. Respect `interval`; stop after `expires_in` seconds. On approval
the block stores the token.

```bash
DEVICE_CODE=$(cat ~/.config/velven/device_code)
deadline=$(( $(date +%s) + 900 )); status=""
while [ "$(date +%s)" -lt "$deadline" ]; do
  res=$(curl -s -w '\n%{http_code}' -X POST "$VELVEN/api/agent/token" \
    -H "content-type: application/json" \
    -d "{\"device_code\":\"$DEVICE_CODE\"}")
  status=${res##*$'\n'}; body=${res%$'\n'*}
  case "$status" in
    200) break ;;                    # approved
    428) sleep 5 ;;                  # authorization_pending
    403) echo "denied"; exit 1 ;;    # access_denied
    000|5*) sleep 5 ;;               # network blip or Velven's side: keep polling
    *)   echo "$body"; exit 1 ;;     # expired_token / invalid_grant: start over
  esac
done
[ "$status" = 200 ] || { echo "Not approved in time: start over at step 1"; exit 1; }
(umask 077 && printf '%s' "$body" | jq -r .access_token > ~/.config/velven/token)
rm -f ~/.config/velven/device_code
TOKEN=$(cat ~/.config/velven/token)
```

On 200 the body is `{"access_token":"vlv_…","token_type":"bearer","handle":"mara",
"profile_url":"…"}`. The token is returned exactly once and lasts 90 days; a
later 401 means it expired or was revoked, so start over at step 1.

## Step 3: fill provenance from the project

Do not quiz the creator; read the repo. Fields marked * are required.

- `url` *: the site's stable production address, such as `project.vercel.app`,
  `site.netlify.app`, `project.pages.dev`, `user.github.io/repo` or
  `app.replit.app`. Never a preview or per-deploy URL (Vercel's
  `project-abc123-team.vercel.app`, Netlify's `hash--site.netlify.app`, a
  `hash.project.pages.dev`): it is frozen at one deploy, and Vercel's default
  protection asks visitors to sign in there. Velven follows redirects and stores the final one.
- `title`: from the page `<title>`, README heading or package name. Max 80.
- `description`: one line, max 160. From the README's first sentence or the
  meta description. Say what you do in it and how to control it.
- `space_type` *: one axis, what you do there, asked in this order. `game` if
  you can win, lose, score or finish; `world` if you are somewhere, a place or
  scene you move through with no goal; `tool` if you leave with something (an
  output, an answer, a file); `wonder` for the rest: generative art, shaders,
  music toys, a simulation you watch, an experiment.
- `engine`: from `package.json` and imports. `three` -> `three.js`;
  `@react-three/fiber` -> `r3f`; `@babylonjs/core` -> `babylon.js`; `phaser` ->
  `phaser`; `p5` -> `p5.js`; `aframe` -> `a-frame`; `playcanvas` -> `playcanvas`;
  `@splinetool/runtime` -> `spline`; a Godot or Unity web export (by its loader
  files) -> `godot` or `unity`; a Marble world -> `marble`; raw WebGL or shader
  code with no library -> `webgl`; plain `<canvas>` or DOM with no library ->
  `canvas`; something else -> `other`. Optional: omit it when the space does
  not run on anything with a name.
- `models`: the models that wrote it. Include the one you are running as when
  you know it (for example `claude-opus-5`). Add others only if the creator or
  commit history says so. Optional.
- `ai_tools`: the tools the model ran in. Include yourself (for example
  `claude-code`). Add others only if the creator or commit history says so. Optional.
- `devices` *: `desktop` when keyboard or mouse is used; add `mobile` when there is
  touch or pointer handling and a responsive viewport; add `vr` when WebXR
  immersive sessions are requested. At least one.
- `how_made`: 2 or 3 sentences on process: prompts, iterations, what was hard.
- `source_url`: the repository if it is public.

Then the page under the frame, from the code. Every field you send is the
creator's; Velven writes the ones you leave out from its recording and never
writes over yours. All are optional:

- `about`: what the space is, up to 1500 characters.
- `how_to_play`: the goal and how to reach it, up to 1000. Leave it out for a
  world, tool or wonder with no goal.
- `controls`: from the input handlers, up to 4 groups and 12 rows in all:
  `[{"label":"Keyboard","rows":[{"input":"Space","action":"Jump"}]}]`.
- `features`: up to 6 short lines on what sets the space apart.
- `orientation`: `landscape`, `portrait` or `any`, how the space is held on a
  phone. Take it from the layout (a fixed wide canvas is `landscape`); ask the
  creator when the code does not say. The Velven page's rotate notice trusts
  only this value.
- `players`: `single` or `multi`.
- `faq`: up to 8 `{"q","a"}` pairs, only if the project answers real questions.

There is no picture field. After listing, Velven opens the space in its own
browser, plays it, records a 5-second clip and takes its first frame as the
thumbnail. Until they land, a few minutes later, the listing's `status` is
`processing` and its page is the creator's alone; then it is `published` and
on the board.

## Step 4: put the proof on the site

Only a verified creator can list a space, so the proof goes on the site
first. It names the creator's `handle` from the token response.

Add `<meta name="velven" content="@HANDLE">` inside the `<head>` of the page
at the listed URL. Deploy (or publish the ChatGPT page again) and wait until
the live page serves it. A ChatGPT site must be published with "Who has
access" set to "Anyone on the Internet": one that only its owner can open
answers 401 and cannot be verified.

## Step 5: submit

```bash
curl -s -X POST "$VELVEN/api/spaces" \
  -H "authorization: Bearer $TOKEN" \
  -H "content-type: application/json" \
  -d @- <<'JSON'
{
  "url": "https://orbit-dodger.netlify.app",
  "title": "Orbit Dodger",
  "description": "Dodge debris in a decaying orbit. Arrow keys or swipe.",
  "space_type": "game",
  "engine": "three.js",
  "models": ["claude-opus-5"],
  "ai_tools": ["claude-code"],
  "devices": ["desktop", "mobile"],
  "how_made": "One session with Claude Code. Three.js scene, hand-rolled physics.",
  "source_url": "https://github.com/mara/orbit-dodger",
  "about": "Orbit Dodger drops you into a decaying orbit full of debris.",
  "how_to_play": "Stay alive as long as you can. Fuel cells lift your orbit.",
  "controls": [{"label":"Keyboard","rows":[{"input":"Left / Right","action":"Thrust"}]}],
  "orientation": "landscape",
  "players": "single"
}
JSON
```

Responses:

- 201 `{"slug","url","status","title"}`: the page exists
  at `url`, `https://velven.ai/HANDLE/SLUG`, and is the creator's alone while
  `status` is `processing`; it goes on the board once the clip lands. Go to
  step 6. The answer can also carry `boards` (the board keys Velven took from
  the page's block), `boards_error` (why it refused the block: fix the block,
  deploy, then send the boards with `PUT` as in step 2 of the SDK section) and
  `page_not_written: true` (send the page fields again with
  `PATCH $VELVEN/api/spaces/SLUG`, as in "Changing a listing").
- 409 `unverified` with `instruction` and `snippet`: the proof is not on the
  live site yet. Make exactly that change, deploy, wait a minute for caches,
  submit again. Stop after 3 tries and tell the creator what is missing.
- 401 `unauthorized`: delete `~/.config/velven/token`, repeat steps 1 and 2.
- 409 `duplicate` with `url`: already listed; give the creator that link. If
  that page says "unclaimed", Velven listed it itself: claim it (see "Claiming
  a space Velven already listed").
- 422 `invalid` with `issues:[{path,message}]`: fix those fields and retry once.
  A `url` issue can mean the host is not supported: Velven lists Vercel,
  Netlify, GitHub Pages, Cloudflare (workers.dev and pages.dev addresses),
  Replit (replit.app addresses) and ChatGPT sites for now.
- 422 `unreachable` with `message`: the URL did not answer (a timeout, DNS, a
  refused connection), or it is shut to visitors. When it is shut, the answer
  also carries `instruction`, saying how to open it (for example, turn off
  Vercel Authentication, or share a ChatGPT site with anyone on the internet):
  tell the creator, since that setting is theirs. Check the deploy is public and
  the address is right, then submit again.
- 422 `unframeable` with `instruction` and `snippet`: every space plays inside
  the Velven page, and this one's headers refuse it. Do what `instruction`
  says, with `snippet` in the file it names, deploy, submit again. Stop after
  3 tries and tell the creator what the page still sends.
  - `snippet` is null on a ChatGPT site: it cannot change its headers. Stop and
    tell the creator it cannot be listed until ChatGPT allows framing.
  - On Replit the snippet is the `.replit` entry only a static deployment
    reads. An app with a server (Autoscale or Reserved VM) sets
    `Content-Security-Policy: frame-ancestors 'self' https://velven.ai` on its
    own responses, and drops any `X-Frame-Options` or `frame-ancestors 'self'`
    its middleware adds (helmet's `frameguard` and its default
    `contentSecurityPolicy`).
- 403 `blocked`: the URL or account cannot be listed. Tell the creator, stop.
- 500 `server_error`: a problem on Velven's side. Retry once after a moment; if
  it persists, tell the creator.

`GET $VELVEN/api/spaces?mine=1` with the bearer token lists what this creator
already has: `[{slug,title,url,plays,upvotes,velven_url}]`.

## Step 6: reply to the creator

Reply with the Velven `url`. Offer the badge for the README or the page:

```html
<a href="https://velven.ai/mara/orbit-dodger"><img src="https://velven.ai/badge/orbit-dodger" alt="On Velven"></a>
```

Hosting note: Velven lists Vercel, Netlify, GitHub Pages, Cloudflare, Replit
and ChatGPT sites for now (a Cloudflare site at its workers.dev or pages.dev
address, a Replit app at its replit.app address; a custom domain on either is
not recognised yet). Every space plays
inside the Velven page, so the site has to allow `https://velven.ai` to frame
it: no `X-Frame-Options`, and any `Content-Security-Policy` must name it in
`frame-ancestors`. A page that blocks framing is not listed until it does; the
422 `unframeable` answer says where the header goes on that host.

## Keeping the clip current

Velven's background check re-records the clip when the deployed page changes.
After a deploy that changes what the space looks like, or when the creator
wants the clip to show something else, ask for a new take now:

```bash
curl -s -X POST "$VELVEN/api/spaces/recapture" \
  -H "authorization: Bearer $TOKEN" \
  -H "content-type: application/json" \
  -d '{"slug":"orbit-dodger","note":"start after the title screen, show the boss"}'
```

`note` is optional, one line, at most 100 characters: what the clip should
show. A creator gets 2 such retakes per space. Responses: 202
`{"slug","status":"queued"}` (or already queued or running); 409
`no_retakes_left`; 404 `not_found`; 503 `capture_off` (try later). The creator
can also upload a clip of their own on the space's edit page; its first frame
becomes the thumbnail.

## Changing a listing

- The page text or the details: `PATCH $VELVEN/api/spaces/SLUG` with only the
  fields to change, as in steps 3 and 5 (`null` clears one). `GET` the same
  address reads them. A 422 `invalid` names the fields to fix.
- A new address, when the space moved host: never submit it as a new space.
  Put the proof tag on the new page and allow framing there (step 4 and the
  hosting note), deploy, then:

```bash
curl -s -X POST "$VELVEN/api/spaces/SLUG/move" \
  -H "authorization: Bearer $TOKEN" \
  -H "content-type: application/json" \
  -d '{"url":"https://orbit-dodger.vercel.app"}'
```

  The slug, plays, boards, scores, saves and page text stay. A refusal carries
  `error` and `message`; `unverified` and `unframeable` also carry
  `instruction` and `snippet`, and `unreachable` carries `instruction` when the
  new page is shut to visitors, handled as on submit. If a server posts scores,
  tokens now carry the new origin as their audience: change the audience the
  server checks to the new origin and redeploy it. The board secret stays the
  same; issue a new one (SDK section, step 3) only if the server itself moved
  to a host that does not have it.

## Leaderboards, sign-in and saves

When the space keeps a score or progress, or the creator asks for a
leaderboard, high scores, sign-in or saves, add the Velven SDK. Velven hosts
the board and hands the rows back; the page draws its own board, Velven draws
none. Sign-in and boards work only once the space is published and claimed.

1. Load the script, or bundle it: `<script src="https://velven.ai/sdk/v1.js"></script>`
   in the page's `<head>`, or `npm i @velven/sdk` and `import { Velven } from "@velven/sdk"`.
   The SDK must run in the document at the listed URL, the one Velven frames.
   A game inside an iframe of the page's own (some ChatGPT sites keep theirs
   under a shell) cannot reach Velven, and its sign-in and score calls answer
   `unavailable`. Before promising a board there, check on the Velven page
   that `await Velven.ready()` answers `"velven"`.
2. Declare the boards beside the proof tag. No board is implicit.

```html
<script type="application/velven+json">
{"boards":[{"key":"main","trust":"client","metric":"points","sort":"desc","min":0,"max":1000000,"cooldown":3}]}
</script>
```

   Velven reads the block when the space is listed (the 201's `boards` or
   `boards_error`) and on its background check, which can take a day or more to
   come round to a space; a claim does not read it. So on a space already listed or just claimed, after deploying
   the block, send the same boards at once:

```bash
curl -s -X PUT "$VELVEN/api/spaces/SLUG/boards" \
  -H "authorization: Bearer $TOKEN" \
  -H "content-type: application/json" \
  -d '{"boards":[{"key":"main","trust":"client","metric":"points","sort":"desc","min":0,"max":1000000,"cooldown":3}]}'
```

   It answers `{"boards":[…]}`; 422 `invalid` with `issues` to fix; 409
   `too_many` when the space would hold more than 10 boards; 503 `failed`,
   try again.
3. Choose the tier with `trust`. A page with no server of the creator's
   checking each run uses `"client"`: the page posts with
   `Velven.scores.submit(value)`, taken under the board's `min`, `max` and
   `cooldown`, and every read is labelled as a client board. `"server"`, the
   default when `trust` is left out, takes scores only from the creator's own
   server; a page's `submit` to it answers `server_only`. For a server board:
   - Issue the secret with `POST $VELVEN/api/spaces/SLUG/secret`, which answers
     it once as `secret` and replaces any earlier one. Never print it, and never
     write it in the page, a client bundle or any file in the repository. Pipe
     it into the host's secret store. Check the host's CLI works first (for
     Vercel, `vercel env ls` in the project), since the old secret stops
     working the moment a new one is issued. For example:

```bash
SECRET=$(curl -sf -X POST "$VELVEN/api/spaces/SLUG/secret" -H "authorization: Bearer $TOKEN" | jq -r '.secret // empty') \
  && [ -n "$SECRET" ] && printf '%s' "$SECRET" | vercel env add VELVEN_BOARD_SECRET production --force
```

     `--force` replaces the variable an earlier run set. Then redeploy, since a
     running deployment keeps the environment it started with. If the add
     fails after the secret was issued, fix the CLI and issue it again.
     On a host with no such command, ask the creator to issue it under For
     developers on the Leaderboards tab of the space's edit page and paste it
     into the host's secrets settings.
   - The page gets the player's token with `await Velven.signIn()` and sends it
     to the server. The server posts with
     `postScore({ secret, token, value, requestId, board })` from
     `@velven/sdk/server` (Node 20+), or POSTs `$VELVEN/api/v1/scores` with the
     header `x-velven-secret` and `token`, `board`, `value`, `request_id` in the
     body.
   - The server can be on any host: Velven checks the secret and the token, not
     where the post came from, so a static site (ChatGPT, GitHub Pages) uses a
     function hosted elsewhere, with CORS allowing the page's origin.
4. Draw the board from `await Velven.scores.top({ limit: 10 })` and
   `await Velven.scores.around()`, each answering `{ ok, trust, rows }`. A row
   has `rank`, `value`, `meta`, `setAt` and `user` (`{ id, handle, avatar }`),
   so a name is `row.user.handle`. Rows are other players' data: render them
   as text, never as HTML.
5. Sign in from a button, never on load: `await Velven.signIn()` shows Velven's
   card to a guest and answers `{ ok, user, token }`; `Velven.user` is already
   set for a signed-in visitor. Every call resolves with `ok`; only a
   programming mistake throws, such as a score that is not a number or
   `Velven.data` before `ready()` settles.
   Identity and boards work inside Velven's page only: on the space's own
   site `await Velven.ready()` answers `"site"` and every identity and score
   call answers `unavailable`, so check the environment before drawing the
   board there. On the Velven page they also answer `unavailable` while the
   listing is `processing` (before its clip lands) or unclaimed: test there
   once it is published, and do not change working code over it.
6. Test on localhost: `?velven_user=alice` is a signed-in player and the
   board lives in memory, seeded, ranked by the page's block. Add
   `?velven_strict=1` to refuse a page submit to a server board, as Velven does.
7. Saves: if the game keeps progress in `localStorage`, move it to
   `Velven.data` (same `getItem`/`setItem`/`removeItem`/`clear`, used after
   `await Velven.ready()`), so the save follows a signed-in player.
   `Velven.data` never reads the game's old keys, so copy them in once, or
   every returning player starts over. Copy each old key `Velven.data` lacks,
   and remove the old keys only on a later visit, once `Velven.data` holds
   every one: if Velven could not hand over the save just then, the copy is
   dropped when it arrives, and the next visit copies again. Wherever the
   game resets or deletes progress (`Velven.data.clear()` or `removeItem`),
   remove the same keys from `localStorage` too, or the next visit copies the
   old progress back.

```js
await Velven.ready();
const OLD_KEYS = ["level", "coins"]; // the game's own localStorage keys
const old = OLD_KEYS.filter((k) => localStorage.getItem(k) !== null);
if (old.every((k) => Velven.data.getItem(k) !== null)) {
  for (const k of old) localStorage.removeItem(k); // Velven holds every one now
} else {
  for (const k of old) if (Velven.data.getItem(k) === null) Velven.data.setItem(k, localStorage.getItem(k));
}
```

   Keep a personal best on the board with `Velven.scores.mine()`, not in the save.
8. Playtime and sound: call `Velven.game.start()` when play begins or resumes
   and `Velven.game.stop()` at a menu, a level's end or game over, so Velven
   counts time played. If the game can mute itself, register
   `Velven.onMute((muted) => …)` and the Velven page shows a sound button.
9. Moderation on the bearer: `DELETE $VELVEN/api/spaces/SLUG/scores/ID` removes
   an entry by its ID (no endpoint shows entry IDs yet: the creator removes
   one with Remove on the Leaderboards tab of the space's edit page, and to
   act on a player, ban them); `POST` and
   `DELETE $VELVEN/api/spaces/SLUG/bans` with `{"handle":"…"}` ban and unban a
   player from every board of the space.

The whole surface, codes and limits: `curl -s $VELVEN/docs/sdk.md`.

## Claiming a space Velven already listed

Velven seeds the board with curated spaces it found itself. Those show as
"unclaimed" at `https://velven.ai/s/SLUG` with no creator attached, and a
`duplicate` error on submit points at one. Claiming moves it to the creator's
handle and keeps its plays. Use this when the creator says a space of theirs is
on Velven but not under their name, or asks to verify or claim it.

1. Get a token (steps 1 and 2). The proof names the `handle` from the token.
2. Put the proof on the site (step 4): the `velven` meta tag in the page's
   `<head>`. Wait until the live site serves it.
3. Take the slug from the Velven page URL (`/s/orbit-dodger` -> `orbit-dodger`)
   and call verify:

```bash
curl -s -X POST "$VELVEN/api/spaces/verify" \
  -H "authorization: Bearer $TOKEN" \
  -H "content-type: application/json" \
  -d '{"slug":"orbit-dodger"}'
```

Responses:

- 200 `{"verified":true,"url"}`: done. `url` is now `https://velven.ai/HANDLE/SLUG`;
  reply with it as in step 6. If the page declares boards, send them with
  `PUT` now (SDK section, step 2).
- 409 `unverified` with `instruction` and `snippet`: the proof is not on the live
  site yet. Make exactly that change, deploy, wait a minute, call again. Stop
  after 3 tries and tell the creator what is missing.
- 409 `owned`: someone else already verified it. Tell the creator; they can
  report it from the space page.
- 404 `not_found`: no published space has that slug. If the slug came from a
  `duplicate` answer, the listing exists but is hidden, because the site
  stopped answering or refuses frames (see the hosting note). Fix that,
  deploy, and call again after Velven's next check, which can take a day or more.
- 401 `unauthorized`: delete `~/.config/velven/token`, repeat steps 1 and 2.
- 403 `blocked`: this account cannot claim spaces. Tell the creator, stop.

The creator can also do it by hand at `https://velven.ai/s/SLUG/claim` once
signed in; the page checks the same proof.

## Values

The allowed values for `space_type`, `engine`, `models`, `ai_tools` and
`devices` are at `https://velven.ai/docs/agent.md` under "Allowed values".
Fetch that rather than guess; a value not on it is a 422.
