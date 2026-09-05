---
name: onboard
description: List a browser-playable space (game, world, tool, simulation, art) on Velven, the leaderboard for spaces built with AI. Use after deploying such a project when the creator says "list on Velven", "submit to Velven", "put this on Velven", "publish my game/space/world to the leaderboard", asks how to get plays and a ranking for something built with AI, or wants to claim or verify a space that is already on Velven ("claim my space", "it says unclaimed", "verify I made this"). Handles the device-code login (the creator clicks one link and presses Approve), puts the ownership proof on the site, fills provenance from the project, submits the URL or claims the existing listing, and replies with the Velven link and badge.
---

# Velven

Velven (https://velven.ai) is the community leaderboard for spaces built with AI.
A space is anything interactive in a browser. It stays where it is hosted; Velven
lists it, counts plays and ranks it. You list it for the creator: no API key is
ever copied by hand. Full docs: `curl -s https://velven.ai/docs/agent.md`.

## When to use

- You just deployed a browser-playable project that was built with AI (any tool)
  and the creator wants it listed, or asks about a leaderboard, plays or ranking.
- Not for: backend services, CLIs, non-interactive pages, or work not built with AI.

Set `VELVEN=https://velven.ai` (or the base URL of a self-hosted instance).

## Step 1: reuse or obtain a token

Tokens live at `~/.config/velven/token` (mode 600). Check first:

```bash
TOKEN=$(cat ~/.config/velven/token 2>/dev/null)
curl -s "$VELVEN/api/agent/me" -H "authorization: Bearer $TOKEN"
# 200 {"handle":"mara","profile_url":"https://velven.ai/mara"}  -> skip to step 3
# 401 -> delete the file and log in below
```

Login: one unauthenticated call returns a link for the creator and a code for you.

```bash
curl -s -X POST "$VELVEN/api/agent/login" \
  -H "content-type: application/json" \
  -d '{"agent_name":"Claude Code"}'
# 201 {"device_code":"…","user_code":"K7PX-4MDQ",
#      "verification_url":"https://velven.ai/agent/approve?code=K7PX-4MDQ",
#      "expires_in":900,"interval":5}
```

Use your own name as `agent_name` (max 60 chars). Never print `device_code`.

## Step 2: the creator approves, you poll

Tell the creator, in one short message:

> Open https://velven.ai/agent/approve?code=K7PX-4MDQ and press Approve.
> The code on the page should read K7PX-4MDQ. If you are not signed in to
> Velven yet, the same page signs you in and asks for a handle. I will wait.

Then poll. Respect `interval`; stop after `expires_in` seconds.

```bash
deadline=$(( $(date +%s) + 900 ))
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
ACCESS_TOKEN=$(printf '%s' "$body" | jq -r .access_token)
```

On 200 the body is `{"access_token":"vlv_…","token_type":"bearer","handle":"mara",
"profile_url":"…"}`. The token is returned exactly once and lasts 90 days; a
later 401 means it expired or was revoked, so start over at step 1. Store it:

```bash
(umask 077 && mkdir -p ~/.config/velven && printf '%s' "$ACCESS_TOKEN" > ~/.config/velven/token)
```

## Step 3: fill provenance from the project

Do not quiz the creator; read the repo. Fields marked * are required.

- `url` *: the deployed, public URL. Velven follows redirects and stores the final one.
- `title`: from the page `<title>`, README heading or package name. Max 80.
- `description`: one line, max 160. From the README's first sentence or the
  meta description. Say what you do in it and how to control it.
- `space_type` *: `game` if there is a goal, score or win state; `tool` if it
  does a job (editor, calculator, converter); `simulation` for a system you
  watch run; `art` for visualizers and generative pieces; `world` for a place you
  walk around or explore with no goal.
- `engine` *: from `package.json` and imports. `three` -> `three.js`;
  `@react-three/fiber` -> `r3f`; `@babylonjs/core` -> `babylon.js`; `phaser`,
  `p5`, `aframe`, `playcanvas` map to their names; Godot/Unity web exports by
  their loader files; plain `<canvas>` with no library -> `canvas`; else `other`.
- `ai_tools` *: the tools that built it. Include yourself (for example
  `claude-code`). Add others only if the creator or commit history says so.
- `devices` *: `desktop` when keyboard or mouse is used; add `mobile` when there is
  touch or pointer handling and a responsive viewport; add `vr` when WebXR
  immersive sessions are requested. At least one.
- `how_made`: two or three sentences on process: prompts, iterations, what was hard.
- `source_url`: the repository if it is public, or the artifact link.
- `screenshot_url`: an absolute image URL if the project ships one (og image or
  a capture you made). Otherwise omit; Velven uses `og:image` or captures one.

## Step 4: put the proof on the site

Only a verified creator can list a space, so the proof goes on the site
first. It names the creator's `handle` from the token response.

- Vercel, Netlify, ChatGPT sites: add `<meta name="velven" content="@HANDLE">`
  inside the page `<head>`. Deploy (or publish the ChatGPT page again) and
  wait until the live page serves it.
- Claude artifacts: you cannot add the tag yourself. Ask the creator to open the
  artifact in Claude, click Publish → Get embed code, and add `HANDLE.velven.ai`
  next to `velven.ai` under Allowed domains. Wait for them.

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
  "ai_tools": ["claude-code"],
  "devices": ["desktop", "mobile"],
  "how_made": "One session with Claude Code. Three.js scene, hand-rolled physics.",
  "source_url": "https://github.com/mara/orbit-dodger"
}
JSON
```

Responses:

- 201 `{"slug","url","embed_mode","nudge","title"}`: listed on the creator's
  page. `url` is `https://velven.ai/HANDLE/SLUG`. Go to step 6.
- 409 `unverified` with `instruction` and `snippet`: the proof is not on the
  live site yet. Make exactly that change, deploy, wait a minute for caches,
  submit again. Stop after three tries and tell the creator what is missing.
- 401 `unauthorized`: delete `~/.config/velven/token`, repeat steps 1 and 2.
- 409 `duplicate` with `url`: already listed; give the creator that link. If
  that page says "unclaimed", Velven listed it itself: claim it (next section).
- 422 `invalid` with `issues:[{path,message}]`: fix those fields and retry once.
  A `url` issue can mean the host is not supported: Velven lists Vercel,
  Netlify, ChatGPT sites and Claude artifacts for now.
- 403 `blocked`: the URL or account cannot be listed. Tell the creator, stop.

`GET $VELVEN/api/spaces?mine=1` with the bearer token lists what this creator
already has: `[{slug,title,url,plays,upvotes,velven_url}]`.

## Claiming a space Velven already listed

Velven seeds the board with curated spaces it found itself. Those show as
"unclaimed" at `https://velven.ai/s/SLUG` with no creator attached, and a
`duplicate` error on submit points at one. Claiming moves it to the creator's
handle and keeps its plays. Use this when the creator says a space of theirs is
on Velven but not under their name, or asks to verify or claim it.

1. Get a token (steps 1 and 2). The proof names the `handle` from the token.
2. Put the proof on the site (step 4): the `velven` meta tag on Vercel, Netlify
   and ChatGPT sites, or `HANDLE.velven.ai` under Allowed domains for a Claude
   artifact. Wait until the live site serves it.
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
  reply with it as in step 6.
- 409 `unverified` with `instruction` and `snippet`: the proof is not on the live
  site yet. Make exactly that change, deploy, wait a minute, call again. Stop
  after three tries and tell the creator what is missing.
- 409 `owned`: someone else already verified it. Tell the creator; they can
  report it from the space page.
- 404 `not_found`: no such slug. Check the URL the creator gave you.
- 401 `unauthorized`: delete `~/.config/velven/token`, repeat steps 1 and 2.
- 403 `blocked`: this account cannot claim spaces. Tell the creator, stop.

The creator can also do it by hand at `https://velven.ai/s/SLUG/claim` once
signed in; the page checks the same proof.

## Step 6: reply to the creator

Reply with the Velven `url`. If `nudge` is non-null, pass it on verbatim: it
explains why the space opens in a new tab and how to make it play in place.
Offer the badge for the README or the page:

```html
<a href="https://velven.ai/mara/orbit-dodger"><img src="https://velven.ai/badge/orbit-dodger" alt="On Velven"></a>
```

Hosting note: Velven lists Vercel, Netlify, ChatGPT sites and Claude artifacts
for now. The first three play in place (`embed_mode: "iframe"`). A Claude
artifact plays in place once the creator publishes it and adds velven.ai under
"Get embed code" → Allowed domains (HANDLE.velven.ai, from step 4, is the
separate ownership proof; both go in the same list); otherwise it opens in a
new tab and plays still count.

## Values

- `space_type`: `game`, `world`, `tool`, `simulation`, `art`
- `engine`: `three.js`, `r3f`, `babylon.js`, `playcanvas`, `a-frame`, `godot`,
  `unity`, `phaser`, `p5.js`, `canvas`, `marble`, `spline`, `other`
- `ai_tools`: `claude-code`, `claude`, `cursor`, `codex`, `copilot`, `gemini`,
  `windsurf`, `lovable`, `bolt`, `v0`, `replit`, `marble`, `other`
- `devices`: `desktop`, `mobile`, `vr`

The live list is always at `https://velven.ai/docs/agent.md` under "Allowed values".
