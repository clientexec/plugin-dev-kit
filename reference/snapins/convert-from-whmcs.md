# Converting WHMCS Addon Modules and Hooks to a Clientexec Snapin

Load `reference/common/conventions.md`, `reference/snapins/create.md`, and `reference/snapins/events.md` first. This file only maps WHMCS concepts onto them.

## The short version

| WHMCS | Clientexec |
| --- | --- |
| Addon module (`modules/addons/acme/acme.php`) | A snapin (`plugins/snapin/acme/PluginAcme.php`) |
| Hooks file (`includes/hooks/*.php` or a module's `hooks.php`) | `$listeners` on a snapin, for events; hook location mappings, for page output |
| Both in one addon | One snapin: mappings for pages, `$listeners` for events |
| Hooks for scheduled tasks (`DailyCronJob`, `AfterCronJob`) | Not a snapin: these belong in a service plugin |

---

## 1. Addon module functions

| WHMCS | Clientexec |
| --- | --- |
| `acme_config()` | `getVariables()` for settings, plus `init()` for everything else (mappings, description) |
| `acme_config()` `name`, `description` | `Plugin Name` setting, `setDescription()` |
| `acme_activate()` / `acme_deactivate()` | No equivalent. Create tables with `CREATE TABLE IF NOT EXISTS` before first use; never drop data. |
| `acme_upgrade($vars)` | No equivalent. Check the schema before use and alter it if needed. |
| `acme_output($vars)` (admin page) | `addMappingForTopMenu('admin', ...)` plus a view method and template |
| Admin page sub-pages (`&action=...` on `modulelink`) | Extra views with `addMappingView()`, linked with `view=viewsnapin&v=<tpl>` |
| AJAX from the admin page | `callAction()`, branching on `$_REQUEST['pluginaction']` |
| `acme_sidebar($vars)` | No equivalent |
| `acme_clientarea($vars)` | `addMappingForTopMenu('public', ...)` and/or `addMappingForPublicMain()`, plus a view |
| AJAX from the client area page | Methods listed in `$publicActions`, called through `action=doaction` |
| Admin role access control (`Access Control` in addon settings) | "Viewable by all staff", or the `snapin_<name>` permission per staff group |
| Client group restrictions | "Viewable by all customers", or the `snapin_<name>` permission per customer group |
| `$vars['modulelink']` | `index.php?fuse=admin&controller=snapins&view=viewsnapin&plugin=<name>&v=<tpl>` |
| `$vars['<setting>']` | `$this->getVariable('<Setting Name>')` |
| Smarty `.tpl` templates | `.phtml` templates named after the view |
| `lang/*.php` files | `$this->user->lang('English text')`; strings are extracted automatically |

### Client area pages

A WHMCS `_clientarea()` returns `['pagetitle' => ..., 'templatefile' => ..., 'vars' => [...]]`. In a snapin, the view method sets `$this->view->...` and the template named after the view is rendered.

WHMCS `requirelogin => true` has no flag equivalent. Check `$this->user->isAnonymous()` at the top of the view and redirect to login, as `csfunblock` does.

---

## 2. Hook points

### Page output hooks

| WHMCS hook | Clientexec |
| --- | --- |
| `ClientAreaHeadOutput`, `ClientAreaHeaderOutput` | `clientarea_all` hook location |
| `ClientAreaFooterOutput` | `clientarea_footer` hook location |
| `AdminAreaHeaderOutput`, `AdminAreaHeadOutput` | `admin_global_top` hook location |
| `AdminHomeWidgets` | `admin_top_dashboard`, `admin_middle_dashboard`, or `admin_bottom_dashboard` |
| `AdminAreaClientSummaryPage` | `admin_profiletab` (a tab on the client profile) |
| `AdminAreaViewTicketPage` | No equivalent |
| `ClientAreaProductDetailsOutput` | `public_profileproducttab` with a `$packageType` filter |
| `AdminServicesTabFields` (from a hook) | `admin_profileproducttab` with a `$packageType` filter |
| `ClientAreaPageSubmitTicket` | `client_top_submitticket` |
| `ClientAreaPage*` returning template variables | No equivalent |
| `ClientAreaPrimarySidebar`, `ClientAreaSecondarySidebar`, navbar hooks | No equivalent |

A WHMCS output hook returns HTML. A Clientexec hook location renders the template named in the mapping, after calling the method of the same name.

### Event hooks

| WHMCS hook | Clientexec event |
| --- | --- |
| `ClientAdd` | `Client-Create` |
| `ClientEdit` | `Client-Update` |
| `ClientDelete` / `PreDeleteClient` | `Client-Delete` (fires after) |
| `ClientChangePassword` | `Client-PasswordChange` |
| `ClientLogin`, `AdminLogin` | `Account-Login` |
| `ClientLogout`, `AdminLogout` | `Account-Logoff` |
| `InvoiceCreated` / `InvoiceCreation` | `Invoice-Create` |
| `UpdateInvoiceTotal`, `InvoiceChangeGateway` | `Invoice-Update` |
| `InvoicePaid` | `Invoice-Paid` |
| `InvoicePaymentReminder` | No equivalent |
| `InvoiceUnpaid` | `Invoice-Unpaid` |
| `InvoiceRefunded` | `Invoice-Refunded` |
| `InvoiceCancelled` | `Invoice-Voided` |
| `InvoiceSplit` | No equivalent; `Invoice-Merge` is the reverse |
| `AcceptOrder`, `AfterShoppingCartCheckout` | `Order-NewPackage` (once per package) |
| `AfterModuleCreate` | `Plugin-Action-Create`, or `UserPackage-Activate` |
| `AfterModuleSuspend` | `Plugin-Action-Suspend`, or `UserPackage-Suspend` |
| `AfterModuleUnsuspend` | `Plugin-Action-UnSuspend`, or `UserPackage-UnSuspend` |
| `AfterModuleTerminate` | `Plugin-Action-Delete`, or `UserPackage-Cancel` |
| `AfterModuleChangePackage` | No confirmed equivalent |
| `AfterModuleChangePassword` | No confirmed equivalent |
| `AfterRegistrarRegistration` | `Plugin-Action-Register` |
| `AfterRegistrarRenewal` | `Plugin-Action-Renew` |
| `AfterRegistrarTransfer` | `Plugin-Action-DomainTransferWithPopup` |
| `ServiceDelete` | `UserPackage-Delete` |
| `CancellationRequest` | `UserPackage-RequestCancel` |
| `TicketOpen` | `Ticket-CreateByClient` |
| `TicketOpenAdmin` | `Ticket-CreateByAdmin` |
| `TicketUserReply` | `Ticket-ReplyByCustomer` |
| `TicketAdminReply` | `Ticket-ReplyByAdmin` |
| `TicketClose` | `Ticket-Closed` |
| `TicketDelete` | `Ticket-Deleted` |
| `ServerAdd`, `ServerEdit`, `ServerDelete` | `Server-Add`, `Server-Edit`, `Server-Delete` |
| `DailyCronJob`, `AfterCronJob`, `PreCronJob` | A service plugin, not a snapin. Events fired during service runs do still reach snapin listeners. |
| `EmailPreSend`, `EmailTplMergeFields` | No equivalent |
| Any `Pre...` hook that can abort the action | No equivalent: Clientexec events fire after the change and can't cancel it |

Prefer the `UserPackage-*` events over `Plugin-Action-*` when you care that a package changed state, however it happened. `Plugin-Action-*` only fires for actions run through the plugin action path.

### Hook parameters and return values

- **WHMCS passes `$vars`; Clientexec passes an event object.** Read the data with `$e->getParams()`, using the keys in `events.md`. They differ from WHMCS's: `invoiceId` rather than `invoiceid`, and a `User` object rather than individual client fields.
- **WHMCS hooks often return data that changes the outcome.** Clientexec ignores listener return values except for `Package-Update` and `Service-CurrencyRateUpdate`.
- **WHMCS hook priority** (the third argument to `add_hook()`) has no equivalent. Don't depend on running before or after another snapin.

---

## 3. Things not to port

- **`localAPI()` calls.** Use Clientexec's models (`User`, `UserPackage`, `Invoice`) and gateways.
- **WHMCS tables** (`tblclients`, `tblinvoices`, `tblhosting`). Clientexec tables have no prefix and a different structure.
- **`Capsule::schema()` in activate/upgrade functions.** Create tables before first use instead (section 1).
- **`$_SESSION` lookups for the current user.** Use `$this->user`.
- **Trusting `$vars` in output hooks.** In a client area hook, verify that any package or record ID from the request belongs to `$this->user` (`create.md`, section 8).

---

## 4. Porting checklist

- [ ] One snapin per addon module, with its hooks as `$listeners` and hook location mappings.
- [ ] Settings in `getVariables()`; mappings in `init()`.
- [ ] Activate/upgrade logic replaced with create-before-use; nothing dropped on disable.
- [ ] Admin page as an admin top menu view; client area page as a public top menu view or home page box.
- [ ] Admin AJAX through `callAction()` with the `NE_PUBLIC` guard and `canUseSnapin()`; client area AJAX through `$publicActions` only.
- [ ] Every WHMCS hook mapped using section 2; anything without a mapping raised as a question.
- [ ] Listeners read keys from `events.md`, re-read state, and never depend on a return value.
- [ ] Cron hooks moved to a service plugin.
- [ ] Everything in the `create.md` release checklist.
