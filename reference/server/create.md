# Creating a Clientexec Server Plugin

A server plugin connects Clientexec to something that provisions services: a hosting control panel (cPanel, HestiaCP, Enhance), a VPS platform (Linode, Vultr, Proxmox), a game server panel, or a hosted service (OpenSRS Email). Clientexec calls the plugin when a package is activated, suspended, unsuspended, deleted, or changed, and when an admin or client clicks an action button.

Load `reference/common/conventions.md` first. It covers naming, directory layout, settings, errors, logging, HTTP, views, and code style, and this file assumes it.

**Examples to copy from:** `hestiacp` (small and clean), `cpanel` (resellers, SSO, client area), `linode` (VPS with custom power actions and per-package settings), `enhance` (client area panel).

---

## 1. Minimum viable plugin

A server plugin must:

1. Extend `ServerPlugin`.
2. Declare `$features`.
3. Implement `getVariables()` with at least `Name`, `Description`, and `Actions`.
4. For each action listed in `Actions`, implement a `do<Action>($args)` method.
5. Implement `getAvailableActions($userPackage)` so Clientexec shows the right buttons for the account's current state.
6. Implement `testConnection($args)` if `testConnection` is enabled in `$features`.

---

## 2. How Clientexec calls your plugin

### The two layers: `doX()` and `x()`

Clientexec invokes actions through `ServerPlugin::doAction()`, which prepends `do` to the action name and calls that method:

| Action name in `Actions` | Method called |
| --- | --- |
| `Create` | `doCreate($args)` |
| `Suspend` | `doSuspend($args)` |
| `UnSuspend` | `doUnSuspend($args)` |
| `Delete` | `doDelete($args)` |
| `Reboot` (custom) | `doReboot($args)` |

`doX()` receives only `['userPackageId' => <id>]` (plus anything the caller adds). Its job is to load the package, build the full argument set, and return a success message string:

```php
public function doSuspend($args): string
{
    $userPackage = new UserPackage($args['userPackageId']);
    $this->suspend($this->buildParams($userPackage));
    return $userPackage->getCustomField('Domain Name') . ' has been suspended.';
}
```

The lower-case methods `create()`, `update()`, `delete()`, `suspend()`, `unsuspend()` receive the full `$args` from `buildParams()` and do the actual API work. They return nothing on success and throw `CE_Exception` on failure. Keeping the two layers separate is the convention in every current plugin, and it means the real work can be called with a hand-built `$args` (for testing, or from another method).

`doAction()` also records usage with `trackUsage()` for every action except `CheckUserName`, so you don't need to.

### Lifecycle

| Event | Called |
| --- | --- |
| Admin saves a new or edited package's username/password | `validateCredentials($args)` |
| Package activated / admin clicks Create | `doCreate($args)` |
| Admin changes username, password, domain, IP, or package | `doUpdate($args)` with `$args['changes']` |
| Suspend / unsuspend (admin, or automated by the autosuspend service) | `doSuspend($args)` / `doUnSuspend($args)` |
| Delete / terminate | `doDelete($args)` |
| Admin or client opens the package, to decide which buttons to show | `getAvailableActions($userPackage)` |
| Admin clicks Test Connection on the server | `testConnection($args)` |
| Login link shown to client or admin | `getDirectLink(...)`, then `dopanellogin($args)` |
| Admin logs into the server itself from the server list | `getAdminDirectLink($args)` |
| Client views the package in the client area | `getPublicView($userPackage, $view)` and panel methods |

---

## 3. `$features`

Override the whole array. Anything you leave out is treated as `false`.

```php
public $features = [
    'packageName' => true,
    'testConnection' => true,
    'showNameservers' => true,
    'directlink' => true,
    'admindirectlink' => false,
    'publicView' => false,
    'upgrades' => true,
    'publicPanels' => [],
    'publicPanelActions' => []
];
```

| Feature | What it enables |
| --- | --- |
| `packageName` | Packages using this server get a "package name on server" field. Its value arrives as `$args['package']['name_on_server']`. Enable it when the remote side has its own named plans (cPanel packages, Hestia packages). Disable it when the plugin configures resources directly through per-package fields (Linode). |
| `testConnection` | Shows a Test Connection button on the server. Requires `testConnection($args)`. |
| `showNameservers` | The server's nameservers are shown in relation to packages on this server. Enable for web hosting panels. |
| `directlink` | Shows a login link for the account. Requires `getDirectLink()` and `dopanellogin()`. |
| `admindirectlink` | Lets an admin log into the server itself (for example WHM) from the server list. Requires `getAdminDirectLink($args)`. |
| `publicView` | Calls `getPublicView()` to add HTML to the client's main package page. |
| `upgrades` | The plugin supports package upgrades and downgrades. When the package changes, `update()` receives `$args['changes']['package']`. |
| `publicPanels` | Client area panels: `['key' => 'Tab label']`. Each key is also a method name. See section 9. |
| `publicPanelActions` | Allow-list of AJAX methods callable from each panel: `['panelKey' => ['methodName', ...]]`. Use the key `publicView` for actions called from `getPublicView()` output. See section 9. |

`linode` also sets `'filterClientActionsByStatus' => true`, which is not declared in the base class. See section 7.

### `$icons`

Font Awesome classes for your client area panels, keyed by panel key:

```php
public $icons = [
    'usage' => 'fa fa-chart-bar'
];
```

---

## 4. Settings

### Per-server settings: `getVariables()`

These are the fields an admin fills in when adding a server that uses your plugin. The server's hostname is a core field, not a plugin setting: read it from `$args['server']['variables']['ServerHostName']`.

Settings Clientexec reads from your `getVariables()` by name:

| Setting | Type | Purpose |
| --- | --- | --- |
| `Name` | hidden | Display name of the plugin. |
| `Description` | hidden | Description shown in server settings. |
| `Actions` | hidden | Comma-separated list of actions admins can run: `'Create,Delete,Suspend,UnSuspend'`. Add custom actions here (`linode` adds `Reboot,Boot,Shutdown,Rebuild,Changepass,RescueOn`). |
| `Registered Actions For Customer` | hidden | Comma-separated subset of actions clients may run from the client area. Leave the value empty if clients shouldn't run any. |
| `reseller` | hidden | `'1'` if the plugin can create reseller accounts. See section 10. |
| `package_addons` | hidden | Comma-separated list of addon variable names the plugin understands. See section 5. |
| `package_vars` | hidden | `'0'`. Declared alongside `package_vars_values` in every plugin that uses per-package fields. |
| `package_vars_values` | hidden | Per-package fields. See below. |

Everything else is your own setting: credentials, ports, options, a failure email address.

### Per-package fields: `package_vars_values`

Some settings belong to the package rather than the server: which plan, how much swap, which node. Declare them inside `package_vars_values`, and Clientexec shows them on each package that uses your plugin.

```php
'package_vars' => [
    'type' => 'hidden',
    'description' => lang('Whether package settings are set'),
    'value' => '0'
],
'package_vars_values' => [
    'type' => 'hidden',
    'description' => lang('Package settings'),
    'value' => [
        'plan' => [
            'type' => 'dropdown',
            'multiple' => false,
            'getValues' => 'getPlans',
            'label' => lang('Plan'),
            'description' => lang('Plan to create the instance on'),
            'value' => ''
        ],
        'swap' => [
            'type' => 'text',
            'label' => lang('Swap'),
            'description' => lang('Swap size in MB'),
            'value' => '512'
        ]
    ]
]
```

Values arrive as `$args['package']['variables']['plan']`, `$args['package']['variables']['swap']`.

A `dropdown` field names a method in `getValues`. Clientexec calls it with the **server ID** and expects an array of `value => label`:

```php
public function getPlans($serverId): array
{
    $server = new Server($serverId);
    $variables = $server->getAllServerPluginVariables($this->user, 'acmepanel');
    $this->setup(['server' => ['variables' => $variables]]);

    $plans = [0 => lang('-- Select Plan --')];
    foreach ($this->api->listPlans() as $plan) {
        $plans[$plan['id']] = $plan['name'];
    }

    return $plans;
}
```

### Package custom fields

When a plugin needs to store per-account data that isn't one of the built-in fields (a remote org ID, a root password, a hostname chosen at signup), the convention is a server setting that names a package custom field, which the admin creates:

```php
'Instance ID Custom Field' => [
    'type' => 'text',
    'description' => lang('Name of the package custom field that will hold the AcmePanel instance ID'),
    'value' => ''
],
```

Read and write it with the `CUSTOM_FIELDS_FOR_PACKAGE` flag:

```php
$fieldName = $args['server']['variables']['plugin_acmepanel_Instance_ID_Custom_Field'];
$instanceId = $userPackage->getCustomField($fieldName, CUSTOM_FIELDS_FOR_PACKAGE);
$userPackage->setCustomField($fieldName, $newId, CUSTOM_FIELDS_FOR_PACKAGE);
```

### Built-in package fields

These are read and written without the flag:

| Field | Holds |
| --- | --- |
| `User Name` | account username |
| `Password` | account password (encrypted at rest when the "Domain Passwords are Encrypted" setting is on; `buildParams()` decrypts it for you) |
| `Domain Name` | primary domain |
| `IP Address` | assigned IP |
| `Shared` | `1` for a shared IP, `0` for dedicated |
| `Server Id` | ID of the server the package is on |
| `Server Acct Properties` | free-form field plugins use to store the remote ID (Linode stores the instance ID here) |

If the remote side normalizes or changes the username, write it back: `$userPackage->setCustomField('User Name', $username);`

---

## 5. The `$args` array

`buildParams($userPackage)` assembles everything a plugin method needs. This is the exact structure:

```php
$args = [
    'customer' => [
        'id' => 12,
        'email' => 'jane@example.com',
        'first_name' => 'Jane',
        'last_name' => 'Doe',
        'organization' => 'Example Co'
    ],
    'package' => [
        'id' => 345,                         // user package ID
        'name' => 'Starter Hosting',         // Clientexec package (plan) name
        'username' => 'janedoe',
        'password' => 'plain text password', // already decrypted
        'domain_name' => 'example.com',
        'ip' => '203.0.113.10',
        'ServerAcctProperties' => '',
        'name_on_server' => 'starter',       // only meaningful when 'packageName' is enabled
        'variables' => [],                   // per-package fields from package_vars_values
        'customfields' => [],                // package custom fields
        'addons' => [],                      // only set when the package has addons
        'is_reseller' => 1,                  // only set for reseller packages
        'acl' => []                          // only set for reseller packages
    ],
    'server' => [
        'id' => 3,
        'variables' => [
            'ServerHostName' => 'server1.example.com',
            'plugin_acmepanel_API_Token' => '...'
            // every setting from getVariables(), keyed as plugin_<dir>_<Name>
        ],
        'nameservers' => []
    ]
];
```

Points that catch people out:

- Customer name keys are `first_name` and `last_name`, with underscores. `firstname` does not exist.
- `name_on_server` is the remote plan name. When the server has "prepend username" enabled, Clientexec prefixes it with the server's username and an underscore.
- `addons` is keyed by the addon variable names you declared in `package_addons`, for example `$args['package']['addons']['BANDWIDTH']`. The key is absent when the package has no addons, so check with `isset()`.
- `buildParams($userPackage, $extra)` merges `$extra` into the result. `doUpdate()` uses this to pass `changes` through.

### Test connection arguments

`testConnection($args)` and anything else called through `buildTestParams($serverId)` get only server settings:

```php
$args = [
    'server' => [
        'variables' => [ /* ServerHostName and plugin settings */ ]
    ]
];
```

So `setup($args)` should only depend on `$args['server']['variables']`.

### Update arguments

`update($args)` gets the normal `$args` plus `$args['changes']`, which holds only what changed:

| Key | Value |
| --- | --- |
| `username` | the new username (`$args['package']['username']` is still the old one) |
| `password` | the new password |
| `domain` | the new domain |
| `ip` | the new IP |
| `package` | present when the package changed; the new plan is in `$args['package']['name_on_server']` |
| `leave_reseller` | reseller package changed to another reseller package; keep reseller status |
| `remove_reseller` | `1` when moving from a reseller package to a non-reseller one |

The `ServerPlugin::update()` docblock still describes an older `CHANGE_USERNAME` / `CHANGE_PACKAGE` format. Current plugins use `$args['changes']`.

Process changes in order and collect errors so one failed change doesn't hide another. When a username changes, update `$args['package']['username']` before handling the next change (cPanel does this so a domain change in the same update targets the renamed account).

---

## 6. Method reference

Signatures are shown as they should appear in your class. Don't add parameter types to these methods (see conventions, section 11).

### `getVariables()`

Returns the settings array. Required. See section 4.

### `setup($args)` (convention, not a base method)

Every current plugin has one: read credentials from `$args['server']['variables']`, throw if any are missing, construct the API client into `$this->api`.

```php
public function setup($args): void
{
    $variables = $args['server']['variables'];
    if (empty($variables['ServerHostName']) || empty($variables['plugin_acmepanel_API_Token'])) {
        throw new CE_Exception('Missing server credentials: please fill out all information when editing the server.');
    }

    $this->api = new AcmepanelApi($variables['ServerHostName'], $variables['plugin_acmepanel_API_Token']);
}
```

### `validateCredentials($args)`

Called when a username/password is set, before anything is sent to the server. Normalize the username to what the remote side accepts and return it, or throw `CE_Exception` explaining what's wrong. The base implementation returns `$args['package']['username']` unchanged.

When `$args['noError']` is set, return the cleaned username without throwing. `hestiacp` and `cpanel` both honor this.

### `create($args)` / `doCreate($args)`

Create the account. If the plugin has `packageName` enabled and `name_on_server` is empty, throw: the package isn't configured. Store whatever the remote side gives back that you'll need later (remote ID, final username, assigned IP).

### `update($args)` / `doUpdate($args)`

Apply `$args['changes']`. `doUpdate()` must pass its own `$args` into `buildParams()` so the changes come through:

```php
public function doUpdate($args): string
{
    $userPackage = new UserPackage($args['userPackageId']);
    $this->update($this->buildParams($userPackage, $args));
    return $userPackage->getCustomField('Domain Name') . ' has been updated.';
}
```

If the remote side can't make a change (Hestia can't rename users), throw a clear message saying the admin must do it by hand.

### `suspend($args)`, `unsuspend($args)`, `delete($args)` and their `do` methods

Straightforward API calls. On delete, clear any remote ID you stored so `getAvailableActions()` offers Create again.

### `testConnection($args)`

Make a cheap authenticated call. Return nothing on success; throw `CE_Exception` with the reason on failure.

### `getAvailableActions($userPackage): array`

Return the actions valid for the account right now. Ask the remote side for the real state rather than trusting Clientexec's status, because the two can drift.

The base implementation returns actions based on Clientexec's package status alone: active gives `Suspend, Delete`, suspended gives `UnSuspend, Delete`, anything else gives `Create`. Override it.

Common shape:

```php
public function getAvailableActions($userPackage): array
{
    $args = $this->buildParams($userPackage);
    $this->setup($args);

    if ($args['package']['username'] == '') {
        return ['Create'];
    }

    try {
        $account = $this->api->getAccount($args['package']['username']);
    } catch (Exception $e) {
        return ['Create'];
    }

    $actions = [];
    $actions[] = $account['suspended'] ? 'UnSuspend' : 'Suspend';
    $actions[] = 'Delete';

    return $actions;
}
```

Action names are case sensitive and must match what you listed in `Actions`: `UnSuspend`, not `Unsuspend`.

For platforms with transitional states (booting, rebuilding), return only `Delete` while the resource is in transition, as `linode` does.

### `doCheckUserName($args)`

Returns `true` if the account already exists on the server. The base implementation infers this from `getAvailableActions()` (no `Create` means it exists), which is fine for most plugins.

### `getDirectLink($userPackage, $getRealLink = true, $fromAdmin = false, $isReseller = false)`

Returns login link information. It's called three ways:

| Call | Return |
| --- | --- |
| `$fromAdmin = true` | `['cmd' => 'panellogin', 'label' => 'Login to AcmePanel']`. The admin UI shows the label and runs the action `panellogin` when clicked, which calls `dopanellogin()`. |
| `$getRealLink = true` | `['fa' => 'fa fa-user fa-fw', 'link' => <SSO URL>, 'text' => <label>, 'form' => '']`. Here you actually create the SSO session. |
| `$getRealLink = false` | Same shape, but `link` points back into Clientexec so the SSO session is only created when the client clicks. |

The deferred client link, as used by `cpanel` and `enhance`:

```php
$link = 'index.php?fuse=clients&controller=products&action=openpackagedirectlink&packageId=' . $userPackage->getId() . '&sessionHash=' . CE_Lib::getSessionHash();
```

Use the deferred form so page loads don't generate single-use login tokens nobody clicks.

`$isReseller` asks for the reseller panel (WHM rather than cPanel). Plugins without reseller support can ignore it.

### `dopanellogin($args)`

Returns the real SSO URL for the admin "login as client" action:

```php
public function dopanellogin($args): string
{
    $userPackage = new UserPackage($args['userPackageId']);
    $response = $this->getDirectLink($userPackage);
    return $response['link'];
}
```

### `getAdminDirectLink($args)`

Only with `admindirectlink`. Unlike other methods, `$args` here is the **flat** server variables array (`$args['ServerHostName']`, `$args['plugin_acmepanel_API_Token']`), not nested under `server`. Return a login URL for the server's admin interface, or `false`.

### `getPackages($args)` / `getAccounts($args)`

Optional. These are currently consumed only by the cPanel-specific WHM importers, which expect cPanel's response shapes. Don't implement them for other platforms expecting a generic importer to use them.

---

## 7. Custom actions and client actions

To add an action such as Reboot:

1. Add it to `Actions` (admin) and, if clients should have it, to `Registered Actions For Customer`.
2. Implement `doReboot($args)`, returning a success message.
3. Return `'Reboot'` from `getAvailableActions()` only when it makes sense.

```php
'Actions' => [
    'type' => 'hidden',
    'description' => lang('Current actions that are active for this plugin per server'),
    'value' => 'Create,Delete,Suspend,UnSuspend,Reboot'
],
'Registered Actions For Customer' => [
    'type' => 'hidden',
    'description' => lang('Current actions that are active for this plugin per server for customers'),
    'value' => 'Reboot'
],
```

`linode` enables `'filterClientActionsByStatus' => true` in `$features`, which, going by its usage, limits client buttons to the registered client actions that `getAvailableActions()` also returns. Use it whenever client actions depend on state.

Client actions run with client privileges. Anything a client action does must only affect that client's own package. `doAction()` receives the package ID from Clientexec, so don't accept a package or resource ID from request input inside a `do` method.

---

## 8. Error handling in server plugins

Follow conventions section 4. Patterns specific to server plugins:

**Failure email.** Many panels offer a "Failure E-mail" setting and send the details of a failed provisioning call there. If you do this, strip secrets first, log it, then send:

```php
private function emailError(string $operation, string $message, array $args): string
{
    $to = $args['server']['variables']['plugin_acmepanel_Failure_E-mail'] ?? '';
    $error = 'AcmePanel account ' . $operation . ' failed. ';
    if ($to !== '') {
        $error .= 'An email with the details was sent to ' . $to . '. ';
    }

    unset($args['server']['variables']['plugin_acmepanel_API_Token']);
    CE_Lib::log(1, 'AcmePanel error: ' . print_r(['operation' => $operation, 'message' => $message, 'params' => $args], true));

    if ($to !== '') {
        $mailGateway = new NE_MailGateway();
        $mailGateway->mailMessageEmail(
            $message,
            $to,
            'AcmePanel Plugin',
            $to,
            '',
            'AcmePanel Account ' . $operation . ' Failure'
        );
    }

    return $error . $message;
}
```

This needs `require_once 'library/CE/NE_MailGateway.php';`.

**Unknown responses are failures.** If the API returns something you don't recognize (HTML from a proxy, an empty body), treat it as an error. Never default to success.

**Long-running operations.** `ServerPlugin`'s constructor removes PHP's time limit, so polling is possible, but bound every loop with a maximum number of attempts. An unbounded `while` waiting for an IP to appear will hang the request forever if the remote side never delivers.

---

## 9. Client area output

### `getPublicView($userPackage, $view): string`

With `publicView` enabled, Clientexec calls this on the client's main package page and inserts the returned HTML. Return `''` when there's nothing to show (for example, the package isn't active).

```php
public function getPublicView(UserPackage $userPackage, $view): string
{
    if (!$userPackage->isActive()) {
        return '';
    }

    $view->addScriptPath(__DIR__ . '/views/');
    $view->userPackageId = $userPackage->getId();

    return $view->render('publicview.phtml');
}
```

This method's signature is typed in the base class, so keep it exactly as shown.

### Panels

Each entry in `$features['publicPanels']` adds a tab to the client's package page. The key is a method on your plugin, called with the package and the view, returning HTML:

```php
public $features = [
    // ...
    'publicPanels' => [
        'usage' => 'Resource usage'
    ],
    'publicPanelActions' => [
        'usage' => ['refreshUsage']
    ]
];

public $icons = [
    'usage' => 'fa fa-chart-bar'
];

public function usage($userPackage, $view): string
{
    if ($userPackage->getCustomerId() != $this->user->getId()) {
        throw new CE_Exception($this->user->lang('You are trying to access a product that does not exist.'));
    }

    $view->addScriptPath(__DIR__ . '/views/');
    $view->userPackageId = $userPackage->getId();
    $view->rows = $this->getUsageRows($userPackage);

    return $view->render('usage.phtml');
}
```

Before calling a panel method, the controller checks that the package belongs to the logged-in client, is active, uses your plugin, and that the key is declared in `publicPanels` and exists as a method. `enhance` checks ownership again in the plugin anyway, which is worth copying: it costs one comparison and protects you if the method is ever reached another way.

If a panel method throws, the client is redirected to the package page with the exception's message, so keep internal details (hostnames, raw API errors) out of messages thrown from panel methods.

### Panel actions (AJAX)

Panel JavaScript posts to the `panelaction` endpoint with the package ID, the panel key, and the method name:

```javascript
$.ajax({
    type: 'POST',
    url: 'index.php?fuse=clients&controller=products&action=panelaction',
    data: $.extend({}, formData, { id: packageId, panel: 'usage', do: 'refreshUsage' }),
    dataType: 'json',
    complete: function (xhr) {
        var response = xhr.responseJSON || { success: false };
        clientexec.parseResponse(response);
    }
});
```

Put the routing keys (`id`, `panel`, `do`) last in `$.extend()` so form data can never override them. The session hash is added by Clientexec's global AJAX setup.

Before calling the method, the controller requires:

- a `POST` request;
- `panel` and `do` matching `^[a-z][a-z0-9_]{0,64}$` (case insensitive);
- a package that belongs to the logged-in client, is active, and uses your plugin;
- `panel` declared in `publicPanels`, or `panel` equal to `'publicView'` with `publicView` enabled;
- `do` listed under that panel in `publicPanelActions`, and existing as a method.

Anything else is rejected with "Invalid action" and logged.

The action method receives the package and `$_POST`, and returns an array:

```php
public function refreshUsage(UserPackage $userPackage, array $request): array
{
    $this->clearUsageCache($userPackage);
    return ['message' => $this->user->lang('Usage refreshed')];
}
```

Return `['message' => ...]` for a notice. Anything under `data` is merged into the top level of the JSON response, so `['data' => ['url' => $url]]` arrives in JavaScript as `response.url`.

Throw `CE_Exception` for errors the client should see: its message is returned as the error. Any other exception is logged and the client gets a generic "An error occurred while processing your request." Since API clients usually throw `CE_Exception` with the raw error, catch those in panel actions and rethrow a message written for the client, logging the original with `CE_Lib::log()`.

Because the request comes from the client, treat every value in `$request` as hostile. Validate formats, and confirm that anything named in the request (a mailbox, a database, an FTP login) belongs to this package's account before acting on it. `cpanel`'s `assertOwnsMailbox()` and `assertDatabaseExists()` show the pattern. This matters most when the remote API call runs with root or reseller credentials and isn't scoped to the account on its own.

### Legacy panel route

`opensrsemail` posts to `index.php?fuse=admin&action=doaction&controller=snapins&plugin=opensrsemail&type=server`, and its action methods take a single `$request` array and load the package from `$request['id']`. That's the older pattern. Use `panelaction` with `publicPanelActions` for new plugins.

### Caching remote data

Client area panels are loaded often. Cache remote calls in the session for a few minutes, and handle running without a session (cron, API, CLI) by fetching directly. `ServerPlugin::getSession()` returns the session or `null`:

```php
$session = $this->getSession();
$cacheKey = 'acmepanel_usage_' . $userPackage->getId();
$cached = $session !== null && isset($session->$cacheKey) ? $session->$cacheKey : null;
if ($cached !== null && $cached['time'] >= time() - 300) {
    return $cached['rows'];
}
```

---

## 10. Reseller accounts

Plugins that can create reseller accounts set the hidden `reseller` setting to `'1'` and declare reseller-specific fields (cPanel uses a `fieldset` and `reseller_acl_fields`). For a reseller package, `$args['package']['is_reseller']` is `1` and `$args['package']['acl']` holds the ACL values. Handle `leave_reseller` and `remove_reseller` in `update()`.

`cpanel` is the only complete reference for this. Read its `setResellerACLs()`, `addReseller()`, and `removeReseller()` before implementing reseller support.

---

## 11. Complete example

A fictional panel called AcmePanel with a REST API. It covers settings, a per-package field, the standard actions, one custom action available to clients, SSO, test connection, and a client area usage panel.

### `plugins/server/acmepanel/PluginAcmepanel.php`

```php
<?php

require_once 'modules/admin/models/ServerPlugin.php';
require_once __DIR__ . '/AcmepanelApi.php';

class PluginAcmepanel extends ServerPlugin
{
    public $features = [
        'packageName' => true,
        'testConnection' => true,
        'showNameservers' => true,
        'directlink' => true,
        'admindirectlink' => false,
        'publicView' => false,
        'upgrades' => true,
        'filterClientActionsByStatus' => true,
        'publicPanels' => [
            'usage' => 'Resource usage'
        ],
        'publicPanelActions' => [
            'usage' => ['refreshUsage']
        ]
    ];

    public $icons = [
        'usage' => 'fa fa-chart-bar'
    ];

    /** @var AcmepanelApi */
    public $api;

    public function getVariables()
    {
        $variables = [
            'Name' => [
                'type' => 'hidden',
                'description' => 'Used by CE to show plugin',
                'value' => 'AcmePanel'
            ],
            'Description' => [
                'type' => 'hidden',
                'description' => lang('Description viewable by admin in server settings'),
                'value' => lang('AcmePanel control panel integration')
            ],
            'API Token' => [
                'type' => 'password',
                'description' => lang('API token generated in AcmePanel under Settings > API'),
                'value' => '',
                'encryptable' => true
            ],
            'Port' => [
                'type' => 'text',
                'description' => lang('AcmePanel API port. Default is 8443.'),
                'value' => '8443'
            ],
            'Failure E-mail' => [
                'type' => 'text',
                'description' => lang('E-mail address AcmePanel error messages will be sent to'),
                'value' => ''
            ],
            'Actions' => [
                'type' => 'hidden',
                'description' => lang('Current actions that are active for this plugin per server'),
                'value' => 'Create,Delete,Suspend,UnSuspend,RestartServices'
            ],
            'Registered Actions For Customer' => [
                'type' => 'hidden',
                'description' => lang('Current actions that are active for this plugin per server for customers'),
                'value' => 'RestartServices'
            ],
            'package_vars' => [
                'type' => 'hidden',
                'description' => lang('Whether package settings are set'),
                'value' => '0'
            ],
            'package_vars_values' => [
                'type' => 'hidden',
                'description' => lang('Package settings'),
                'value' => [
                    'php_version' => [
                        'type' => 'dropdown',
                        'multiple' => false,
                        'getValues' => 'getPhpVersions',
                        'label' => lang('PHP version'),
                        'description' => lang('Default PHP version for new accounts on this package'),
                        'value' => ''
                    ]
                ]
            ]
        ];

        return $variables;
    }

    public function setup($args): void
    {
        $variables = $args['server']['variables'];
        if (empty($variables['ServerHostName']) || empty($variables['plugin_acmepanel_API_Token'])) {
            throw new CE_Exception('Missing server credentials: please fill out all information when editing the server.');
        }

        $port = $variables['plugin_acmepanel_Port'] ?? '';
        $this->api = new AcmepanelApi(
            $variables['ServerHostName'],
            $port !== '' ? $port : '8443',
            $variables['plugin_acmepanel_API_Token']
        );
    }

    public function getPhpVersions($serverId): array
    {
        $server = new Server($serverId);
        $this->setup(['server' => ['variables' => $server->getAllServerPluginVariables($this->user, 'acmepanel')]]);

        $versions = [0 => lang('-- Server default --')];
        foreach ($this->api->get('/php/versions') as $version) {
            $versions[$version] = 'PHP ' . $version;
        }

        return $versions;
    }

    public function validateCredentials($args)
    {
        $username = strtolower(trim($args['package']['username']));
        $username = preg_replace('/[^a-z0-9]/', '', $username);
        $username = substr($username, 0, 16);

        if (isset($args['noError'])) {
            return $username;
        }

        if ($username === '') {
            throw new CE_Exception('The AcmePanel username must contain at least one letter or number.');
        }

        if (preg_match('/^[0-9]/', $username) === 1) {
            throw new CE_Exception('The AcmePanel username cannot start with a number.');
        }

        if (trim($args['package']['password']) === '') {
            throw new CE_Exception('The AcmePanel password is blank.');
        }

        return $username;
    }

    public function doCreate($args): string
    {
        $userPackage = new UserPackage($args['userPackageId']);
        $this->create($this->buildParams($userPackage));
        return $userPackage->getCustomField('Domain Name') . ' has been created.';
    }

    public function create($args): void
    {
        $this->setup($args);

        if (empty($args['package']['name_on_server'])) {
            throw new CE_Exception("This package is not configured properly. Missing 'Package Name on Server'.");
        }

        $username = strtolower(trim($args['package']['username']));
        $payload = [
            'username' => $username,
            'password' => $args['package']['password'],
            'email' => $args['customer']['email'],
            'name' => trim($args['customer']['first_name'] . ' ' . $args['customer']['last_name']),
            'plan' => $args['package']['name_on_server'],
            'domain' => $args['package']['domain_name']
        ];

        $phpVersion = $args['package']['variables']['php_version'] ?? '';
        if ($phpVersion !== '' && $phpVersion !== '0') {
            $payload['php_version'] = $phpVersion;
        }

        try {
            $account = $this->api->post('/accounts', $payload);
        } catch (Exception $e) {
            throw new CE_Exception($this->emailError('Creation', $e->getMessage(), $args));
        }

        $userPackage = new UserPackage($args['package']['id']);
        $userPackage->setCustomField('User Name', $account['username']);
        if (!empty($account['ip'])) {
            $userPackage->setCustomField('IP Address', $account['ip']);
        }
    }

    public function doUpdate($args): string
    {
        $userPackage = new UserPackage($args['userPackageId']);
        $this->update($this->buildParams($userPackage, $args));
        return $userPackage->getCustomField('Domain Name') . ' has been updated.';
    }

    public function update($args): void
    {
        $this->setup($args);
        $username = strtolower(trim($args['package']['username']));
        $errors = [];

        foreach ($args['changes'] as $key => $value) {
            switch ($key) {
                case 'username':
                    try {
                        $this->api->patch('/accounts/' . rawurlencode($username), ['username' => strtolower(trim($value))]);
                        $username = strtolower(trim($value));
                    } catch (Exception $e) {
                        $errors[] = $this->emailError('Username Change', $e->getMessage(), $args);
                    }
                    break;

                case 'password':
                    try {
                        $this->api->patch('/accounts/' . rawurlencode($username), ['password' => $value]);
                    } catch (Exception $e) {
                        $errors[] = $this->emailError('Password Change', $e->getMessage(), $args);
                    }
                    break;

                case 'domain':
                    try {
                        $this->api->patch('/accounts/' . rawurlencode($username), ['domain' => $value]);
                    } catch (Exception $e) {
                        $errors[] = $this->emailError('Domain Change', $e->getMessage(), $args);
                    }
                    break;

                case 'package':
                    try {
                        $this->api->patch('/accounts/' . rawurlencode($username), ['plan' => $args['package']['name_on_server']]);
                    } catch (Exception $e) {
                        $errors[] = $this->emailError('Package Change', $e->getMessage(), $args);
                    }
                    break;

                case 'ip':
                    $errors[] = 'AcmePanel does not support changing an account IP through the API. Please change it on the server.';
                    break;
            }
        }

        if (count($errors) > 0) {
            CE_Lib::log(4, 'plugin_acmepanel::update::error: ' . print_r($errors, true));
            throw new CE_Exception($errors[0]);
        }
    }

    public function doSuspend($args): string
    {
        $userPackage = new UserPackage($args['userPackageId']);
        $this->suspend($this->buildParams($userPackage));
        return $userPackage->getCustomField('Domain Name') . ' has been suspended.';
    }

    public function suspend($args): void
    {
        $this->setup($args);
        $username = strtolower(trim($args['package']['username']));

        try {
            $this->api->post('/accounts/' . rawurlencode($username) . '/suspend');
        } catch (Exception $e) {
            throw new CE_Exception($this->emailError('Suspension', $e->getMessage(), $args));
        }
    }

    public function doUnSuspend($args): string
    {
        $userPackage = new UserPackage($args['userPackageId']);
        $this->unsuspend($this->buildParams($userPackage));
        return $userPackage->getCustomField('Domain Name') . ' has been unsuspended.';
    }

    public function unsuspend($args): void
    {
        $this->setup($args);
        $username = strtolower(trim($args['package']['username']));

        try {
            $this->api->post('/accounts/' . rawurlencode($username) . '/unsuspend');
        } catch (Exception $e) {
            throw new CE_Exception($this->emailError('Unsuspension', $e->getMessage(), $args));
        }
    }

    public function doDelete($args): string
    {
        $userPackage = new UserPackage($args['userPackageId']);
        $this->delete($this->buildParams($userPackage));
        return $userPackage->getCustomField('Domain Name') . ' has been deleted.';
    }

    public function delete($args): void
    {
        $this->setup($args);
        $username = strtolower(trim($args['package']['username']));

        try {
            $this->api->delete('/accounts/' . rawurlencode($username));
        } catch (Exception $e) {
            throw new CE_Exception($this->emailError('Deletion', $e->getMessage(), $args));
        }
    }

    public function doRestartServices($args): string
    {
        $userPackage = new UserPackage($args['userPackageId']);
        $params = $this->buildParams($userPackage);
        $this->setup($params);
        $username = strtolower(trim($params['package']['username']));

        try {
            $this->api->post('/accounts/' . rawurlencode($username) . '/restart');
        } catch (Exception $e) {
            CE_Lib::log(1, 'plugin_acmepanel::doRestartServices::error: ' . $e->getMessage());
            throw new CE_Exception($this->user->lang('Services for %s could not be restarted. Please try again later.', $username));
        }

        return $this->user->lang('Services for %s have been restarted.', $username);
    }

    public function testConnection($args): void
    {
        CE_Lib::log(4, 'plugin_acmepanel::testConnection');
        $this->setup($args);

        try {
            $this->api->get('/status');
        } catch (Exception $e) {
            throw new CE_Exception('Could not connect to AcmePanel: ' . $e->getMessage());
        }
    }

    public function getAvailableActions($userPackage): array
    {
        $args = $this->buildParams($userPackage);
        $this->setup($args);
        $username = strtolower(trim($args['package']['username']));

        if ($username === '') {
            return ['Create'];
        }

        try {
            $account = $this->api->get('/accounts/' . rawurlencode($username));
        } catch (Exception $e) {
            return ['Create'];
        }

        $actions = [];
        if (!empty($account['suspended'])) {
            $actions[] = 'UnSuspend';
        } else {
            $actions[] = 'Suspend';
            $actions[] = 'RestartServices';
        }
        $actions[] = 'Delete';

        return $actions;
    }

    public function getDirectLink($userPackage, $getRealLink = true, $fromAdmin = false, $isReseller = false): array
    {
        $linkText = $this->user->lang('Login to AcmePanel');

        if ($fromAdmin) {
            return [
                'cmd' => 'panellogin',
                'label' => $linkText
            ];
        }

        if (!$getRealLink) {
            return [
                'fa' => 'fa fa-user fa-fw',
                'link' => 'index.php?fuse=clients&controller=products&action=openpackagedirectlink&packageId=' . $userPackage->getId() . '&sessionHash=' . CE_Lib::getSessionHash(),
                'text' => $linkText,
                'form' => ''
            ];
        }

        $args = $this->buildParams($userPackage);
        $this->setup($args);
        $session = $this->api->post('/accounts/' . rawurlencode(strtolower(trim($args['package']['username']))) . '/sso');

        return [
            'fa' => 'fa fa-user fa-fw',
            'link' => $session['url'],
            'text' => $linkText,
            'form' => ''
        ];
    }

    public function dopanellogin($args): string
    {
        $userPackage = new UserPackage($args['userPackageId']);
        $response = $this->getDirectLink($userPackage);
        return $response['link'];
    }

    public function usage($userPackage, $view): string
    {
        if ($userPackage->getCustomerId() != $this->user->getId()) {
            throw new CE_Exception($this->user->lang('You are trying to access a product that does not exist.'));
        }

        $view->addScriptPath(__DIR__ . '/views/');
        $view->userPackageId = $userPackage->getId();
        $view->rows = $this->getUsageRows($userPackage);

        return $view->render('usage.phtml');
    }

    public function refreshUsage(UserPackage $userPackage, array $request): array
    {
        $session = $this->getSession();
        if ($session !== null) {
            $cacheKey = 'acmepanel_usage_' . $userPackage->getId();
            unset($session->$cacheKey);
        }

        return ['message' => $this->user->lang('Usage refreshed')];
    }

    private function getUsageRows($userPackage): ?array
    {
        $session = $this->getSession();
        $cacheKey = 'acmepanel_usage_' . $userPackage->getId();
        $cached = $session !== null && isset($session->$cacheKey) ? $session->$cacheKey : null;
        if ($cached !== null && $cached['time'] >= time() - 300) {
            return $cached['rows'];
        }

        $args = $this->buildParams($userPackage);
        $this->setup($args);

        try {
            $usage = $this->api->get('/accounts/' . rawurlencode(strtolower(trim($args['package']['username']))) . '/usage');
        } catch (Exception $e) {
            CE_Lib::log(4, 'plugin_acmepanel::getUsageRows::error for package ' . $userPackage->getId() . ': ' . $e->getMessage());
            return null;
        }

        $rows = [];
        foreach ($usage['resources'] as $resource) {
            $rows[] = [
                'label' => $resource['label'],
                'used' => $resource['used'],
                'limit' => $resource['limit'] === null ? $this->user->lang('Unlimited') : $resource['limit']
            ];
        }

        if ($session !== null) {
            $session->$cacheKey = ['time' => time(), 'rows' => $rows];
        }

        return $rows;
    }

    private function emailError(string $operation, string $message, array $args): string
    {
        $to = $args['server']['variables']['plugin_acmepanel_Failure_E-mail'] ?? '';
        $error = 'AcmePanel account ' . $operation . ' failed. ';
        if ($to !== '') {
            $error .= 'An email with the details was sent to ' . $to . '. ';
        }

        unset($args['server']['variables']['plugin_acmepanel_API_Token']);
        unset($args['package']['password']);
        CE_Lib::log(1, 'AcmePanel error: ' . print_r(['operation' => $operation, 'message' => $message, 'params' => $args], true));

        if ($to !== '') {
            require_once 'library/CE/NE_MailGateway.php';
            $mailGateway = new NE_MailGateway();
            $mailGateway->mailMessageEmail(
                $message,
                $to,
                'AcmePanel Plugin',
                $to,
                '',
                'AcmePanel Account ' . $operation . ' Failure'
            );
        }

        return $error . $message;
    }
}
```

### `plugins/server/acmepanel/AcmepanelApi.php`

```php
<?php

class AcmepanelApi
{
    private string $baseUrl;
    private string $token;

    public function __construct(string $host, string $port, string $token)
    {
        $this->baseUrl = 'https://' . trim($host) . ':' . trim($port) . '/api/v1';
        $this->token = trim($token);
    }

    public function get(string $path): array
    {
        return $this->request('GET', $path);
    }

    public function post(string $path, array $body = []): array
    {
        return $this->request('POST', $path, $body);
    }

    public function patch(string $path, array $body): array
    {
        return $this->request('PATCH', $path, $body);
    }

    public function delete(string $path): array
    {
        return $this->request('DELETE', $path);
    }

    private function request(string $method, string $path, array $body = []): array
    {
        $ch = curl_init();
        curl_setopt($ch, CURLOPT_URL, $this->baseUrl . $path);
        curl_setopt($ch, CURLOPT_CUSTOMREQUEST, $method);
        curl_setopt($ch, CURLOPT_RETURNTRANSFER, true);
        curl_setopt($ch, CURLOPT_SSL_VERIFYPEER, true);
        curl_setopt($ch, CURLOPT_SSL_VERIFYHOST, 2);
        curl_setopt($ch, CURLOPT_CAINFO, \Composer\CaBundle\CaBundle::getSystemCaRootBundlePath());
        curl_setopt($ch, CURLOPT_TIMEOUT, 60);
        curl_setopt($ch, CURLOPT_HTTPHEADER, [
            'Authorization: Bearer ' . $this->token,
            'Accept: application/json',
            'Content-Type: application/json'
        ]);
        if ($method !== 'GET' && count($body) > 0) {
            curl_setopt($ch, CURLOPT_POSTFIELDS, json_encode($body));
        }

        $response = curl_exec($ch);
        if ($response === false) {
            $error = curl_error($ch);
            curl_close($ch);
            throw new CE_Exception('Connection to AcmePanel failed: ' . $error);
        }

        $status = curl_getinfo($ch, CURLINFO_HTTP_CODE);
        curl_close($ch);

        $decoded = json_decode($response, true);
        if (!is_array($decoded)) {
            throw new CE_Exception('AcmePanel returned an unexpected response (HTTP ' . $status . ').');
        }

        if ($status < 200 || $status >= 300) {
            $message = $decoded['error'] ?? ('HTTP ' . $status);
            throw new CE_Exception('AcmePanel API error: ' . $message);
        }

        return $decoded;
    }
}
```

### `plugins/server/acmepanel/views/usage.phtml`

```php
<?php if ($this->rows === null) { ?>
    <p><?php echo $this->user->lang('Resource usage is not available right now. Please try again later.'); ?></p>
<?php } else { ?>
    <table class="table">
        <thead>
            <tr>
                <th><?php echo $this->user->lang('Resource'); ?></th>
                <th><?php echo $this->user->lang('Used'); ?></th>
                <th><?php echo $this->user->lang('Limit'); ?></th>
            </tr>
        </thead>
        <tbody>
            <?php foreach ($this->rows as $row) { ?>
                <tr>
                    <td><?php echo $this->escape($row['label']); ?></td>
                    <td><?php echo $this->escape($row['used']); ?></td>
                    <td><?php echo $this->escape($row['limit']); ?></td>
                </tr>
            <?php } ?>
        </tbody>
    </table>
    <button type="button" class="btn btn-outline-primary" id="acmepanel-refresh-usage">
        <?php echo $this->user->lang('Refresh'); ?>
    </button>
<?php } ?>

<script>
    (function () {
        var packageId = <?php echo (int) $this->userPackageId; ?>;

        $('#acmepanel-refresh-usage').on('click', function () {
            clientexec.mask();
            $.ajax({
                type: 'POST',
                url: 'index.php?fuse=clients&controller=products&action=panelaction',
                data: { id: packageId, panel: 'usage', do: 'refreshUsage' },
                dataType: 'json',
                complete: function (xhr) {
                    var response = xhr.responseJSON || { success: false };
                    clientexec.parseResponse(response);
                    if (response.success == true) {
                        location.reload();
                    } else {
                        clientexec.unMask();
                    }
                }
            });
        });
    })();
</script>
```

Add a `logo.png` and an empty `index.html` in each directory.

---

## 12. Checklist before release

- [ ] Directory, file, and class names follow the naming rule.
- [ ] Every credential setting has `'encryptable' => true`.
- [ ] `setup()` throws a clear message when credentials are missing.
- [ ] Every action in `Actions` has a matching `do` method, and names match exactly (`UnSuspend`).
- [ ] `getAvailableActions()` reflects the real remote state.
- [ ] `create()` stores what later calls need (final username, remote ID, IP).
- [ ] `delete()` clears stored remote IDs.
- [ ] `update()` handles every change key it can, and gives a clear manual-action message for the ones it can't.
- [ ] No secrets in logs or emails.
- [ ] TLS verification on, using the Composer CA bundle.
- [ ] Every polling loop has a maximum number of attempts.
- [ ] Client actions and panel actions only touch the requesting client's own package, and validate every request value.
- [ ] View output inside arrays is escaped.
- [ ] Panel action methods are listed in `publicPanelActions`.
- [ ] No parameter types added to overridden `ServerPlugin` methods.

---

## 13. Modernizing an older plugin

Older server plugins tend to share the same issues. Fix them in this order:

1. **Errors:** replace `return new CE_Error(...)` and returned error strings with `throw new CE_Exception(...)`.
2. **`update()` input:** if the plugin reads `$args['CHANGE_USERNAME']`, `$args['CHANGE_PACKAGE']` and so on, switch to `$args['changes']` (section 5).
3. **TLS:** remove `CURLOPT_SSL_VERIFYPEER => false` and set `CURLOPT_CAINFO` from the Composer CA bundle.
4. **Client area actions:** move from the legacy `fuse=admin&controller=snapins&action=doaction` route to `panelaction` with a `publicPanelActions` allow-list, and change action methods to `(UserPackage $userPackage, array $request): array`.
5. **Escaping:** escape every array value printed in views.
6. **Translation:** replace global `lang()` with `$this->user->lang()` outside `getVariables()`.
7. **Style:** `array()` to `[]`, add visibility to every method, add return types, remove closing `?>`, replace alternative control syntax with braces.
8. **Unbounded loops:** add a maximum attempt count to any loop that polls the remote API.

---

## Canonical reference files

| What | File |
| --- | --- |
| Base class | `modules/admin/models/ServerPlugin.php` |
| Clean minimal plugin | `plugins/server/hestiacp/PluginHestiacp.php`, `HestiacpApi.php` |
| Full hosting panel, resellers, SSO, admin direct link | `plugins/server/cpanel/PluginCpanel.php` |
| VPS with power actions, per-package fields, client actions | `plugins/server/linode/PluginLinode.php` |
| Client area panel with session caching | `plugins/server/enhance/PluginEnhance.php`, `usage.phtml` |
| Panel JavaScript helper (panelaction, open-URL actions) | `plugins/server/cpanel/assets/panels.js` |
