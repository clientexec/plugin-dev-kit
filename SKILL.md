---
name: clientexec-plugin-dev-kit
description: Build, convert, or modernize Clientexec plugins (server/provisioning, payment gateways, registrars, and snapins today; services planned). Use when writing a new Clientexec plugin, porting a WHMCS module, addon, or hooks file to Clientexec, or updating an older Clientexec plugin to current conventions.
---

# Clientexec Plugin Development Kit

Always load `reference/common/conventions.md` first. It holds the rules every plugin type follows, and most mistakes come from skipping it.

Then load the file for the task:

| Task | Load |
| --- | --- |
| New server / provisioning plugin (hosting panel, VPS, game server, email service) | `reference/server/create.md` |
| Port a WHMCS provisioning module | `reference/server/create.md` + `reference/server/convert-from-whmcs.md` |
| Update an older server plugin | `reference/server/create.md` (see "Modernizing an older plugin") |
| New payment gateway | `reference/gateways/create.md` |
| Port a WHMCS payment gateway | `reference/gateways/create.md` + `reference/gateways/convert-from-whmcs.md` |
| Update an older gateway | `reference/gateways/create.md` (see "Modernizing an older gateway") |
| New registrar | `reference/registrars/create.md` |
| Port a WHMCS registrar module | `reference/registrars/create.md` + `reference/registrars/convert-from-whmcs.md` |
| Update an older registrar | `reference/registrars/create.md` (see "Modernizing an older registrar") |
| New snapin (admin or client area pages, hook output, event listeners) | `reference/snapins/create.md` + `reference/snapins/events.md` |
| Port a WHMCS addon module or hooks file | `reference/snapins/create.md` + `reference/snapins/events.md` + `reference/snapins/convert-from-whmcs.md` |

## Ground rules for the agent

1. Never invent a Clientexec method, constant, or `$args` key. If the reference doesn't list it and you can't find it in the base class or a shipped plugin, say so and ask.
2. Never add parameter types to methods you override from a base class (`ServerPlugin`, `GatewayPlugin`, `RegistrarPlugin`, `SnapinPlugin`). The parent methods are untyped, and narrowing a parameter type in a child is a fatal error in PHP. Return types are fine.
3. Copy patterns from the examples named in each reference (for server plugins: `hestiacp`, `cpanel`, `linode`, `enhance`; for gateways: `mollie`, `stripe`, `stripecheckout`, `offlinebanktransfer`; for registrars: `namesilo`; for snapins: `csfunblock`, `cetransactions`, `ownedsupport`).
4. For gateways, never record a payment from values the browser or a callback request supplied. Verify with the processor first (`reference/gateways/create.md`, section 8).
5. For snapins, only methods in `$publicActions` and views registered as public are reachable from the client area, possibly by visitors who aren't logged in. Check login and ownership in each one (`reference/snapins/create.md`, section 8).
6. Produce complete files, not fragments.
