# Handsfree Project Notes

## Shipping changes — ALWAYS use ./deploy.sh

To ship any change, run exactly one command:

```bash
./deploy.sh "commit message"
```

It commits everything, rebuilds the APK, and restarts the server, in that
order. NEVER run git commit, build-apk.sh, or a server restart as separate
steps — partial deploys caused version drift and an update loop where the
app reinstalled a stale APK forever.

The restart at the end kills the server your session runs inside, ending
your turn. Speak your summary FIRST, then run ./deploy.sh as the final
action. If your turn dies there, the deploy still completed.

## Batch queued requests into ONE deploy

Every deploy makes the user manually update the app on their phone, which is
annoying. If user messages are queued behind the current turn (agentd logs
show the queue; the queued bubbles do too), do NOT deploy after each one.
Make the code changes for as many queued requests as possible, deploying
only when a queued task actually requires the new build to proceed (e.g.
testing behavior that needs the new APK), or once the queue is drained.
Commits can happen along the way; the rebuild+restart is the expensive part.

## APK Build Process

The Android APK bundles a copy of `public/index.html` inside the app at build time. Changes to the HTML require rebuilding the APK.

**Use the build script:**

```bash
./build-apk.sh
```

This script:
1. Gets the current git SHA and embeds it in public/index.html as APP_VERSION
2. Copies index.html to the Android assets
3. Builds the APK
4. Copies it to public/handsfree.apk
5. Restarts the server so it reports the same git SHA

This ensures the APK and server are always in sync. When the app connects, if its embedded SHA differs from the server's current SHA, it prompts the user to update.

**Important:** Commit your changes before running the build script. The APK embeds the current git SHA, so if you build before committing, the APK will have the old SHA and won't trigger updates.

Simply restarting the server is NOT enough for HTML changes to take effect in the app.

### If the APK is huge, it's one of these two (2026-08-21: 79MB -> 3.9MB)

1. Capacitor's `webDir` is `public/`, which is also where the built APK lands,
   so a `cap sync` copies handsfree.apk into `android/app/src/main/assets/`
   and every later build ships a stale copy of itself. build-apk.sh now `rm`s
   those, and `android/.gitignore` hides the whole assets dir, so nothing in
   git tells you it's there — check `unzip -l` on the APK.
2. Gradle's packager updates the output APK in place, so deleting a large
   asset leaves a hole: entries totalled 3.8MB while the file stayed 75MB.
   `./gradlew clean` before building reclaims it. Only needed after removing
   something big; a normal build doesn't require it.

Sanity check a suspicious build with `unzip -l public/handsfree.apk | tail -1`
and compare that total against the file size on disk.

## TTS: two engines, and how to change the voice

Two speech servers run side by side as systemd user services, and the web
server talks to whichever one `TTS_URL` points at:

- `tts-server.service` — Piper on :9877, one voice fixed at load
  (`/research/minima/tts-server/server.py`, currently en_GB-jenny_dioco).
- `kokoro-tts.service` — Kokoro on :9879, takes a voice name per request
  (`/research/minima/kokoro-tts-server/server.py`), 22 voices cached.

`TTS_URL` is NOT in `.env` — it's a systemd drop-in at
`~/.config/systemd/user/handsfree.service.d/tts.conf`, currently pointing at
Kokoro. Switching engines means editing that file, and the reload needs a
`handsfree` service restart, which kills agentd and you — so make it the very
last action of a turn, detached, the way deploy.sh does it.

To change the VOICE, do NOT restart anything: write the name into `.voice` in
the repo root. server.js re-reads it on every synthesis, so the next reply
speaks in the new voice. Kokoro names are `af_`/`am_` for American female/male
and `bf_`/`bm_` for British; `af_heart` is the default and highest quality.
A bad name returns a 500 and that one reply falls back to phone TTS.

Voices can be BLENDED by listing several. `af_heart,bf_emma` averages the
embeddings equally — that's kokoro's own `load_voice`. Weights are an addition
to our server (`resolve_voice` in the kokoro server.py):
`af_heart:0.6,bf_emma:0.4` normalizes the weights and sums, so they need not
add to 1. Weights summing to zero raise, which surfaces as a 500. Editing that
file DOES need `systemctl --user restart kokoro-tts.service` (~25s warmup);
that service is separate from handsfree, so restarting it will not kill you.

Don't trust the config files to say what's actually speaking — on 2026-08-16 I
told the user the voice was Piper's Jenny by reading the old server.py, while
Kokoro's Emma was in fact doing the talking. Check the listening port and the
service env first.

## Server process management — READ BEFORE RESTARTING ANYTHING

Two processes, both supervised by `run.sh` (single instance via `.run.lock`).
The supervisor normally runs under the systemd user service `handsfree`
(installed by `install-service.sh`, auto-started at boot via lingering):

- `server.js` — web layer: HTTPS, phone WebSocket, transcription, TTS relay.
- `agentd.js` — agent daemon: runs YOUR turns via the Agent SDK on
  ws://127.0.0.1:9878. You are a child of agentd, NOT of the web server.

What this means for you:

- You may restart the web server anytime with `fuser -k 8443/tcp` — it does
  NOT end your turn. The phone reconnects in ~2s and keeps streaming.
- `./deploy.sh` restarts agentd via SIGHUP, which waits for your current
  turn to finish before restarting. Your deploy turn survives; agentd picks
  up new code before the NEXT turn.
- NEVER kill agentd directly (`.agentd.pid`, port 9878) — that kills you
  mid-turn. Let deploy.sh's SIGHUP drain handle it.
- NEVER run `node server.js`, `node agentd.js`, or `./run.sh` yourself —
  duplicate copies fight over ports and crash-loop (caused thousands of
  EADDRINUSE restarts on 2026-07-13).
- NEVER use `pkill -f` with a pattern containing "server.js", "agentd.js",
  or "run.sh" — it kills your own shell wrapper. Kill by port or PID file.
- NEVER run `systemctl --user stop/restart handsfree` mid-turn — it kills
  the whole cgroup, including agentd and you. If a task truly requires it,
  run it detached with a delay as the very last action, like deploy.sh.
- Keys live in the git-ignored `.env`; don't move them to shell env.
