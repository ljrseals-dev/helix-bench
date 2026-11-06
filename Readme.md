```markdown
# Helix Bench — Realtime Live Preview & Vercel Deployment

This is the App Builder workspace (Helix Bench).  
It is designed so the sandbox can keep a **live, realtime preview** of your app while you edit, and so the same codebase deploys cleanly to **Vercel**.

---

## 1. Making the Bench Realtime (Live Preview)

The platform’s live preview works by discovering whatever is listening on a specific address and streaming it (with HMR) into the user’s chat UI.

### Required contract

| Requirement | Why it matters |
|-------------|----------------|
| Server must bind **`0.0.0.0:8080`** | The preview proxy only looks for a process on this host:port. |
| Use the provided start command | `npm run dev` already expands to the correct Vite flags via `scripts/with-app-env.mjs`. |
| Leave the dev server running | Killing it blanks the user’s live preview. |
| Prefer HMR over restarts | Source edits are pushed instantly. Only restart for `vite.config` / dependency changes. |

### How the realtime loop works

1. You (or `startup.sh`) start the app with:
   ```bash
   npm run dev
   # → node scripts/with-app-env.mjs vite dev --host 0.0.0.0 --port 8080
   ```
2. The platform’s preview proxy discovers the process on `0.0.0.0:8080`.
3. Vite’s HMR websocket keeps the browser in the chat UI in sync with every file save.
4. On session revive / hibernate the platform re-runs `/workspace/startup.sh`.  
   A correct `startup.sh` looks like this:

   ```sh
   #!/bin/sh
   set -eu
   cd /workspace
   node scripts/preview.mjs stop || true          # never inherit stale :8081 preview
   if curl -sf -o /dev/null --max-time 2 http://127.0.0.1:8080/; then
     exit 0                                       # already running → do nothing
   fi
   npm run dev >>/tmp/app-startup.log 2>&1 &      # start in background
   ```

### What breaks realtime

- Binding only to `127.0.0.1` or a different port → preview proxy never finds the app.
- Starting Vite directly (`vite dev`) instead of through `npm run dev` → missing env / plugins.
- Leaving `nitro({ preset: "vercel" })` active in **dev** mode → Nitro opens a second port and the single-port 8080 contract is broken.
- Killing the process mid-session → user sees a blank preview pane.

### Quick verification (inside the sandbox)

```bash
curl -sf -o /dev/null --max-time 2 http://127.0.0.1:8080/ && echo "live"
# or the full smoke:
node scripts/browser-smoke.mjs http://127.0.0.1:8080/ /workspace/screenshots/preview.png
```

---

## 2. Hooking the Bench up to Vercel

The same codebase is built and deployed by the platform to Vercel.  
Your job is to keep the production build clean; the platform handles the actual deploy.

### Key files already present

| File | Role |
|------|------|
| `vercel.json` | Forces `npm install --omit=dev` so only production dependencies are installed on Vercel. |
| `vite.config.ts` | Gates `nitro({ preset: "vercel", serverDir: "./server" })` so it only runs on `build` / `preview`, never in `dev`. |
| `package.json` → `"build"` | `vite build` + `db:migrate` so the schema is ready when the serverless functions start. |

### Rules that keep the Vercel deploy working

1. **Production dependencies must be in `dependencies`**  
   Vercel runs `npm install --omit=dev`. Anything imported by the production build (Vite plugins, Tailwind/PostCSS plugins, Nitro, etc.) that is only in `devDependencies` will be missing and the build fails.

2. **`npm run build` must succeed and produce a renderable output**  
   The most common blank-page failure is:
   ```
   Failed to load module script … MIME type "text/html"
   ```
   This means the built `index.html` is requesting assets that 404, so the server returns the HTML fallback. Fix the asset base path / output directory so `/assets/*` resolve correctly.

3. **Never write to the filesystem at runtime in production**  
   Serverless functions are ephemeral. Use the provided DB layer (PGLite locally, Neon/`DATABASE_URL` on Vercel) instead of local files.

4. **Preserve the nitro gate in `vite.config.ts`**  
   ```ts
   // already present – do not remove the command / isPreview guard
   nitro({
     preset: "vercel",
     serverDir: "./server",   // required for the Home Screen install page
   })
   // only applied when command === "build" || isPreview
   ```

5. **Local smoke of the production build** (optional but recommended)
   ```bash
   npm run build
   npm run preview:restart   # serves the built output on 127.0.0.1:8081
   # open http://127.0.0.1:8081 and confirm it renders
   ```

### Environment variables on Vercel

The platform injects:

- `DATABASE_URL` (when a database is enabled)
- Auth credentials (when auth is enabled)

You never hard-code secrets. The same code works in the live preview (dev) and on the deployed Vercel URL.

---

## Quick checklist

**Realtime (sandbox)**  
- [ ] App listens on `0.0.0.0:8080`  
- [ ] Started via `npm run dev` (or a correct `startup.sh`)  
- [ ] Dev server left running  
- [ ] HMR working after file saves  

**Vercel**  
- [ ] All production imports live in `dependencies`  
- [ ] `npm run build` succeeds  
- [ ] Built assets are reachable (no MIME-type 404)  
- [ ] Nitro is gated to build/preview only  
- [ ] No runtime filesystem writes  

Once both sides of the contract are satisfied, the bench stays realtime while you work and deploys cleanly when the platform pushes it to Vercel.
``````markdown
# System Prompt — Helix Bench Realtime & Vercel Contract

You are operating inside the Helix Bench (App Builder) workspace.  
Your output must keep the live preview realtime and remain deployable to Vercel.  
Follow every rule below exactly.

## 1. Realtime Live Preview Contract

The platform streams whatever listens on `0.0.0.0:8080` into the user’s chat UI as a live preview with HMR.

### Mandatory rules

- The development server **must** bind to `0.0.0.0:8080`.  
  Never bind only to `127.0.0.1` or any other port.
- Always start the server with:
  ```
  npm run dev
  ```
  (This expands to the correct Vite flags via `scripts/with-app-env.mjs`.)  
  Never invoke `vite` directly.
- Leave the development server running.  
  Killing it blanks the user’s live preview.
- Prefer Vite HMR for source changes.  
  Restart the server only when `vite.config` or dependencies change.
- On every session (including hibernate/revive) ensure a correct `startup.sh` exists:

  ```sh
  #!/bin/sh
  set -eu
  cd /workspace
  node scripts/preview.mjs stop || true
  if curl -sf -o /dev/null --max-time 2 http://127.0.0.1:8080/; then
    exit 0
  fi
  npm run dev >>/tmp/app-startup.log 2>&1 &
  ```

- Never leave Nitro active in development mode.  
  Nitro must be gated so it only runs on `build` or `preview`; otherwise it opens a second port and breaks the single-port 8080 contract.

### Verification

After starting or restarting the server, confirm it is reachable:

```
curl -sf -o /dev/null --max-time 2 http://127.0.0.1:8080/
```

## 2. Vercel Deployment Contract

The same codebase is built and deployed by the platform to Vercel.  
You never trigger the deploy yourself, but your output must produce a clean production build.

### Mandatory rules

- All packages required by the production build (Vite plugins, Tailwind/PostCSS plugins, Nitro, etc.) **must** live in `dependencies`.  
  Vercel installs with `npm install --omit=dev` (see `vercel.json`). Packages only in `devDependencies` will be missing and the build will fail.
- `npm run build` must succeed and emit valid, renderable output.  
  The most common blank-page failure is:
  ```
  Failed to load module script … MIME type "text/html"
  ```
  This means built assets 404 and the server returns the HTML fallback. Fix the asset base path so `/assets/*` resolve correctly.
- Never write to the local filesystem at runtime in production code.  
  Use the provided database layer (PGLite in the sandbox, `DATABASE_URL` / Neon on Vercel).
- Preserve the Nitro gate already present in `vite.config.ts`:
  - Nitro with `preset: "vercel"` and `serverDir: "./server"` must run only when `command === "build"` or when previewing the production build.
  - It must never run during `vite dev`.
- Do not hard-code secrets, hosts, or ports.  
  The platform injects `DATABASE_URL` and auth credentials on deploy.

### Local production smoke (when needed)

```
npm run build
npm run preview:restart   # serves the built output on 127.0.0.1:8081
```

Confirm the page at `http://127.0.0.1:8081` renders correctly before considering the build complete.

## 3. Priority Order

1. Keep the app listening on `0.0.0.0:8080` with HMR live.  
2. Ensure `npm run build` succeeds and the production output is renderable.  
3. Never break the Nitro gate or move production dependencies into `devDependencies`.  

Any change that violates the realtime contract or the Vercel contract is a regression and must be fixed before finishing the turn.
```
