---
name: clientexec-extension-dev-kit
description: Build, convert, or modernize Clientexec plugins (server/provisioning today; registrars, gateways, snapins and services planned). Use when writing a new Clientexec plugin, porting a WHMCS module to Clientexec, or updating an older Clientexec plugin to current conventions.
---

# Clientexec Extension Development Kit

Always load `reference/common/conventions.md` first. It holds the rules every plugin type follows, and most mistakes come from skipping it.

Then load the file for the task:

| Task | Load |
| --- | --- |
| New server / provisioning plugin (hosting panel, VPS, game server, email service) | `reference/server/create.md` |
| Port a WHMCS provisioning module | `reference/server/create.md` + `reference/server/convert-from-whmcs.md` |
| Update an older server plugin | `reference/server/create.md` (see "Modernizing an older plugin") |

## Ground rules for the agent

1. Never invent a Clientexec method, constant, or `$args` key. If the reference doesn't list it and you can't find it in the base class or a shipped plugin, say so and ask.
2. Never add parameter types to methods you override from `ServerPlugin`. The parent methods are untyped, and narrowing a parameter type in a child is a fatal error in PHP. Return types are fine.
3. Copy patterns from the examples named in each reference (for server plugins: `hestiacp`, `cpanel`, `linode`, `enhance`).
4. Produce complete files, not fragments.
