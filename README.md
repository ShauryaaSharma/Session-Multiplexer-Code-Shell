# Session Multiplexer Code Shell

Browser-based code editor with a live, interactive terminal. Sessions get their own PTY (via `node-pty`) multiplexed over Socket.IO on a single backend host; project files sync to/from S3. Monaco-style editor + file tree included. Single-host architecture — no per-project isolation or dynamic provisioning.

## What this is

A minimal, self-hosted "online IDE" — think a stripped-down code-execution playground. Open a project in the browser, edit files with a Monaco-based editor, and get a real shell (not a simulated one) to run commands against your project, all from one backend process.

## How it works

1. **Project creation** — `POST /project` copies a base template (`node-js` or `python`) from an S3 bucket into a new `code/<replId>` prefix.
2. **Session connect** — the frontend opens a Socket.IO connection with `?roomId=<replId>`. On connect, the backend pulls that project's files from S3 into a local `tmp/<replId>` folder on the host and sends back the root file tree.
3. **Editing** — `fetchDir` / `fetchContent` / `updateContent` events read and write files on local disk, and mirror saves back to S3.
4. **Terminal** — `requestTerminal` forks a PTY (`node-pty`) scoped to that session's `tmp/<replId>` directory. Terminal I/O streams both ways over the same socket, keyed by socket ID. Multiple sessions multiplex their own PTYs through one backend process — hence the name.

Because the terminal is a real shell and not a sandboxed interpreter, it can run **any file of your choice** — not just the seeded boilerplate — as long as the runtime/toolchain it needs (Node, Python, etc.) is installed on the machine the backend runs on. The only requirement on the storage side is that the project's boilerplate files exist under the expected prefix in your S3-compatible bucket — this works with **AWS S3** or **Cloudflare R2** (or any other S3-compatible endpoint), configured via `S3_ENDPOINT`.

## Architecture

```
frontend (Vite + React)  <-- Socket.IO / REST -->  backend (Express + Socket.IO)
                                                          |
                                                     node-pty (per-session shell)
                                                          |
                                                     local disk (tmp/<replId>)
                                                          |
                                                          v
                                                          S3 (source of truth)
```

- **Single host, single process.** All sessions, terminals, and file storage run inside one backend instance. There is no per-project isolation, containerization, or dynamic compute provisioning — every user shares the same machine's CPU, memory, and filesystem.
- **No database.** Project existence isn't tracked anywhere; `replId` collisions silently overwrite existing S3 data.
- **No auth.** Any client that knows (or guesses) a `replId` can join that session's socket room and read/write its files.

## Tech stack

**Backend** — Node.js, Express, Socket.IO, `node-pty`, AWS SDK (S3), TypeScript
**Frontend** — React, Vite, Monaco Editor, xterm.js, Socket.IO client, TypeScript

## Project structure

```
backend/
  src/
    index.ts     # entrypoint — wires up HTTP + WS servers
    http.ts      # REST: POST /project (template -> S3)
    ws.ts        # Socket.IO handlers: file sync, terminal I/O
    pty.ts       # TerminalManager — forks/writes/kills PTYs per session
    fs.ts        # local filesystem read/write helpers
    aws.ts       # S3 fetch/copy/save helpers
frontend/
  src/
    components/
      Landing.tsx     # create/open a project
      CodingPage.tsx  # editor + terminal + output panel
      Editor.tsx      # file tree + Monaco editor
      Terminal.tsx    # xterm.js terminal bound to the socket
      Output.tsx      # iframe preview (expects something on :3000)
```

## Getting started

### Prerequisites

- Node.js 18+
- Any language runtimes/tools you want to run code with (Node, Python, etc.) installed on the host running the backend — the terminal is a real shell, so it can execute whatever is available on that machine
- An S3-compatible bucket — **AWS S3** or **Cloudflare R2** both work — with `base/node-js` and `base/python` template folders pre-seeded

### Backend

```bash
cd backend
yarn install
cp .env.example .env   # fill in S3_BUCKET, AWS_ACCESS_KEY_ID, AWS_SECRET_ACCESS_KEY, S3_ENDPOINT
yarn dev                # starts on :3001
```

### Frontend

```bash
cd frontend
yarn install
yarn dev                # starts on :5173 (Vite default)
```

Open the frontend, enter a repl ID, pick a language, and click **Start Coding**.

## Environment variables

| Variable | Description |
|---|---|
| `S3_BUCKET` | Bucket used for project templates and storage |
| `AWS_ACCESS_KEY_ID` | AWS credential |
| `AWS_SECRET_ACCESS_KEY` | AWS credential |
| `S3_ENDPOINT` | Override for S3-compatible storage (e.g. Cloudflare R2, MinIO) |
| `PORT` | Backend port (default `3001`) |

## Known limitations

- **No automatic run step.** Creating a project only copies template files — there's no `npm install` or process launch. The "Output" panel expects something listening on `localhost:3000`, but nothing starts it; you have to run your app manually in the terminal, and it only actually renders if you're on the same machine as the backend.
- **No isolation between projects.** Everything executes on one shared host with no sandboxing, resource limits, or per-project compute.
- **No auth or access control.** Sessions are joinable by anyone with the `replId`.
- **No persistence layer.** No database tracks which projects exist; re-creating an existing `replId` silently overwrites it.
- **Terminal session cleanup bug.** `TerminalManager` keys sessions by socket ID but deletes by PID on exit, so exited terminal entries are never actually cleared from memory.

This repo represents an early, single-host iteration of the idea. A companion repo builds on this with per-project isolation via Kubernetes-scheduled sandboxes.
