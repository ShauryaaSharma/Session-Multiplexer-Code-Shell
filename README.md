# Session-Multiplexer-Code-Shell
Browser-based code editor with a live, interactive terminal. Sessions get their own PTY (via node-pty) multiplexed over Socket.IO on a single backend host; project files sync to/from S3. Monaco-style editor + file tree included. Single-host architecture, no per-project isolation or dynamic provisioning.
