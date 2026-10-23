# Clientexec Extension Development Kit

A reference kit for **building**, **converting**, and **modernizing** Clientexec plugins, written so it works equally well for human developers and for AI coding tools.

## Two ways to use it

1. **Portable context (any AI tool).** Everything under `reference/` is plain Markdown. Give the file for the plugin type you're working on to Cursor, Copilot, ChatGPT, Claude, or any other tool, along with `reference/common/conventions.md`, then ask it to build or convert a plugin. If you're porting from WHMCS, also include the matching `convert-from-whmcs.md`.
2. **Claude Code skill.** This folder is also a skill. `SKILL.md` routes to the right reference on demand. Copy or symlink the folder into your Claude Code skills directory (for example `~/.claude/skills/clientexec-extension-dev-kit`) and invoke it from Claude Code.

## Layout

```
clientexec-extension-dev-kit/
├── SKILL.md                         # skill router (also a human index)
├── README.md                        # this file
└── reference/
    ├── common/
    │   └── conventions.md           # shared rules: naming, layout, settings, errors, logging, HTTP, views, style
    ├── server/
    │   ├── create.md                # author a new server (provisioning) plugin, with a complete example
    │   └── convert-from-whmcs.md    # port a WHMCS provisioning module to a Clientexec server plugin
    └── gateways/
        ├── create.md                # author a new payment gateway, with a complete offsite example
        └── convert-from-whmcs.md    # port a WHMCS payment gateway module to a Clientexec gateway
```

## Scope today

- **Server plugins:** full method surface of `ServerPlugin`, the `$args` structure, settings and per-package fields, actions and client actions, single sign-on, client area views and panels, resellers, a complete example plugin, and a WHMCS porting guide.
- **Payment gateways:** offline, offsite, onsite form, and stored payment method gateways; the full `$params` set, return contract, the `Plugin` result recorder, callbacks, settings core reads, payment verification rules, a complete offsite example with a webhook, and a WHMCS porting guide.
- **Shared conventions:** the rules every plugin type follows.

Planned: registrar plugins, snapins, service (cron) plugins, fraud, captcha, import/export, and an "updating older plugins" guide.

## Sourcing and accuracy

These references are distilled from the Clientexec plugin base classes and the plugins that ship with Clientexec:

- `modules/admin/models/ServerPlugin.php`, `GatewayPlugin.php`, `PluginCallback.php`, `library/CE/NE_Plugin.php`, `modules/billing/models/class.gateway.plugin.php`, and the gateway dispatcher in `modules/billing/models/BillingGateway.php`
- Server examples: `cpanel` (full featured, reseller support, SSO, panels), `hestiacp` (clean minimal hosting panel), `linode` (VPS with custom power actions and per-package settings), `enhance` (client area panel), `opensrsemail` (legacy panel pattern)
- Gateway examples: `mollie` (offsite plus webhook), `stripe` (onsite form plus stored payment methods), `stripecheckout` (hosted checkout), `offlinebanktransfer` (offline), `razorpay` (onsite order flow)
