# Clientexec Plugin Development Kit

A reference kit for **building**, **converting**, and **modernizing** Clientexec plugins, written so it works equally well for human developers and for AI coding tools.

## Two ways to use it

1. **Portable context (any AI tool).** Everything under `reference/` is plain Markdown. Give the file for the plugin type you're working on to Cursor, Copilot, ChatGPT, Claude, or any other tool, along with `reference/common/conventions.md`, then ask it to build or convert a plugin. If you're porting from WHMCS, also include the matching `convert-from-whmcs.md`.
2. **Claude Code skill.** This folder is also a skill. `SKILL.md` routes to the right reference on demand. Copy or symlink the folder into your Claude Code skills directory (for example `~/.claude/skills/clientexec-plugin-dev-kit`) and invoke it from Claude Code.

## Layout

```
clientexec-plugin-dev-kit/
├── SKILL.md                         # skill router (also a human index)
├── README.md                        # this file
└── reference/
    ├── common/
    │   └── conventions.md           # shared rules: naming, layout, settings, errors, logging, HTTP, views, style
    ├── server/
    │   ├── create.md                # author a new server (provisioning) plugin, with a complete example
    │   └── convert-from-whmcs.md    # port a WHMCS provisioning module to a Clientexec server plugin
    ├── gateways/
    │   ├── create.md                # author a new payment gateway, with a complete offsite example
    │   └── convert-from-whmcs.md    # port a WHMCS payment gateway module to a Clientexec gateway
    ├── registrars/
    │   ├── create.md                # author a new registrar plugin, with a complete example
    │   └── convert-from-whmcs.md    # port a WHMCS registrar module to a Clientexec registrar
    └── snapins/
        ├── create.md                # author a new snapin: pages, hook locations, AJAX, events, with a complete example
        ├── events.md                # every event Clientexec fires and the data it passes
        └── convert-from-whmcs.md    # port a WHMCS addon module or hooks file to a snapin
```

## Scope today

- **Server plugins:** full method surface of `ServerPlugin`, the `$args` structure, settings and per-package fields, actions and client actions, single sign-on, client area views and panels, resellers, a complete example plugin, and a WHMCS porting guide.
- **Payment gateways:** offline, offsite, onsite form, and stored payment method gateways; the full `$params` set, return contract, the `Plugin` result recorder, callbacks, settings core reads, payment verification rules, a complete offsite example with a webhook, and a WHMCS porting guide.
- **Registrars:** every method core calls, with the exact `$params` and return shapes core expects; domain search and name suggest, register/renew/transfer, nameservers, DNS, contacts, lock, domain and price import, automatic renewal, and a complete example.
- **Snapins:** admin and client area pages, hook locations, admin and client area AJAX, access control, event listeners, the full event catalog, a complete example, and a WHMCS addon module and hooks porting guide.
- **Shared conventions:** the rules every plugin type follows.

Planned: service (cron) plugins, fraud, captcha, two-factor authentication (2FA), and import/export.
