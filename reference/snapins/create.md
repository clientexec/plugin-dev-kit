# Creating a Clientexec Snapin

A snapin adds features to Clientexec itself, rather than connecting it to an outside service. A snapin can:

- add pages to the admin menu or the client area menu, and a box to the client area home page;
- add content to existing pages at fixed **hook locations** (client profile tabs, package tabs, the dashboard, the top of every admin page, the client area footer);
- respond to **events** such as an invoice being paid, a client being created, or a ticket being opened;
- expose AJAX actions to its own admin pages and, when declared, to the client area.

For WHMCS developers, a snapin covers what both an **addon module** and a **hooks file** do.

Load `reference/common/conventions.md` first. The event catalog is in `reference/snapins/events.md`.

**Examples to copy from:** `csfunblock` (client area page that checks login), `cetransactions` (admin menu page, admin profile tab, admin AJAX), `ceinvoices` (single admin hook), `crisp` (client area footer script), `ownedsupport` and `clicksend` (event listeners).

---

## 1. Files

```
plugins/snapin/acmestatus/
├── PluginAcmestatus.php   # the snapin (required)
├── index.html
├── status.phtml           # one template per view or hook, named after it
├── status.js              # optional: loaded automatically with the "status" view
├── status.css             # optional: loaded automatically with the "status" view
└── widget.phtml
```

Each view and hook has a name (its "tpl"). For a tpl named `status`, Clientexec calls the snapin's `status()` method if it exists, then renders `status.phtml`. For views, `status.js` and `status.css` in the snapin directory are added to the page automatically.

There is no install, enable, disable, or upgrade hook. If a snapin needs its own table, create it with `CREATE TABLE IF NOT EXISTS` before first use, and don't drop data when the snapin is disabled.

---

## 2. The class

```php
<?php

require_once 'modules/admin/models/SnapinPlugin.php';

class PluginAcmestatus extends SnapinPlugin
{
    public function getVariables()
    {
        $variables = [
            'Plugin Name' => [
                'type' => 'hidden',
                'description' => '',
                'value' => 'Acme Status'
            ],
            'Status API URL' => [
                'type' => 'text',
                'description' => lang('URL of your Acme Status JSON feed'),
                'value' => ''
            ]
        ];

        return $variables;
    }

    public function init()
    {
        $this->setDescription('Shows service status to staff and clients');
        $this->addMappingHook('admin_top_dashboard', 'widget', 'Service Status', 'Shows current service status on the dashboard');
    }
}
```

### `init()`

Register every mapping in `init()`. Clientexec calls `init()` when it loads the snapin to render a page or run an action. It does **not** call it when loading the snapin to deliver an event (section 9), so don't rely on anything set up in `init()` inside a listener.

| Setter | Effect |
| --- | --- |
| `setDescription($text)` | Description in the snapin settings list. Plain text. |
| `setEnabledByDefault(true)` | Enabled on new installs. |
| `setSystemPlugin(true)` | Treated as part of Clientexec: settings hidden, always enabled, and access controlled only by group permissions. Use only for features that ship with Clientexec. |
| `setSettingLocation($location)` | Show settings under another settings section (`users`, `support`, `style`, `signup`, `security`, `packages`, `email`, `company`, `billing`) instead of Setup > Plugins > Snapins. |
| `setPermissionLocation($location)` | Group the snapin's permission under another section (`home`, `clients`, `domains`, `billing`, `support`, `knowledgebase`, `files`, `reports`, `admin`). |
| `$this->settingsNotes = '...'` | Note shown on the settings screen. |
| `public $hideOnDemo = true;` | Hide the snapin in demo mode. |

### Settings

Your settings are stored as `plugin_<directory>_<Setting Name>`, with spaces kept, the same as gateways. Read them with `$this->getVariable('Status API URL')` or `$this->settings->get('plugin_acmestatus_Status API URL')`.

Clientexec adds these settings itself, so don't declare them:

| Setting | Added when |
| --- | --- |
| `Enabled` | always |
| `Viewable by all staff` | the snapin has admin features (section 4) |
| `Viewable by all customers` | the snapin has client area features (section 4) |

Snapins with a client area home page box (`addMappingForPublicMain()`) can also declare `Use Image` (hidden, `'1'`) to show the box's icon as an image, and `Public Description` (hidden) for the text clients see.

As with every non-server plugin type, `encryptable` has no effect on snapin settings (conventions, section 3).

---

## 3. Where a snapin appears: mappings

### Admin and client area menu entries

```php
$this->addMappingForTopMenu($type, $loc, $tpl, $title, $description, $icon);
```

| Argument | Values |
| --- | --- |
| `$type` | `'admin'` or `'public'` |
| `$loc` | Admin: `clients`, `billing`, `support`, `tools` (default). Client area: `products`, `profile`, `billing`, `support`, `tools` (default). Add `\|Group Name` to group several entries under a submenu: `'support\|Screen Share'`. |
| `$tpl` | The view's name: method `<tpl>()` and template `<tpl>.phtml` |
| `$title` | Menu text and page title |
| `$description` | Shown in the snapin settings as one of its features |
| `$icon` | Font Awesome class, optional |

This also registers `$tpl` as a view.

### Client area home page box

```php
$this->addMappingForPublicMain($tpl, $title, $description, $icon, $iconStyle);
```

Adds a box to the client area home page linking to the view `$tpl`. `$icon` is a Font Awesome class or, with the `Use Image` setting, an image path.

### Hook locations

```php
$this->addMappingHook($location, $tpl, $title, $description, $packageType);
```

| Location | Where it renders |
| --- | --- |
| `admin_global_top` | Top of every admin page |
| `admin_top_dashboard` | Admin dashboard, top |
| `admin_middle_dashboard` | Admin dashboard, middle |
| `admin_bottom_dashboard` | Admin dashboard, bottom |
| `admin_top_active_panel` | Admin active panel, top |
| `admin_bottom_active_panel` | Admin active panel, bottom |
| `admin_profiletab` | A tab on the admin client profile |
| `admin_profileproducttab` | A tab on a client's package in the admin. Filter with `$packageType`. |
| `admin_top_invoice` | Top of the admin invoice view |
| `public_profileproducttab` | A tab on the client's package page in the client area. Filter with `$packageType`. |
| `client_top_submitticket` | Top of the client area submit ticket page |
| `clientarea_all` | Every client area page |
| `clientarea_footer` | Client area footer, on every page (good for chat widgets and tracking scripts) |

`$packageType` is `-1` (all, the default), `PACKAGE_TYPE_GENERAL`, `PACKAGE_TYPE_HOSTING`, `PACKAGE_TYPE_SSL`, or `PACKAGE_TYPE_DOMAIN`.

Hooks know the page they're on only through the request. Package and profile tabs read the ID with `$this->getParam('id', FILTER_SANITIZE_NUMBER_INT)`, as `ceinvoices` does.

### Extra views

```php
$this->addMappingView($tpl, $title);
```

Registers a view that isn't in a menu, such as a detail page your main page links to. Admins reach it at:

```
index.php?fuse=admin&controller=snapins&view=viewsnapin&plugin=acmestatus&v=<tpl>
```

In the client area, an extra view only renders if it's listed in `$publicViews` (section 5).

### Settings buttons

```php
$this->addMappingForSettings('clearcache', 'Clear Cache', 'Clears the cached status feed.');
```

Adds a button to the snapin's settings screen. Clicking it calls the snapin's `callAction()` with `$_REQUEST['pluginaction']` set to the method name, so handle it there (`gotomeeting` handles its `deletecustomfields` button this way).

---

## 4. Who can use a snapin

Clientexec decides access in `User::canUseSnapin()`. The snapin must be enabled, and then:

- super admins always pass;
- in the client area, anyone passes when **Viewable by all customers** is on, **including visitors who aren't logged in**;
- otherwise, the user's group needs the `snapin_<directory>` permission.

Which of those settings and permissions exist depends on the snapin's features, as listed on its settings screen:

| Admin features | Client area features |
| --- | --- |
| `topmenu` entries of type `admin` | `topmenu` entries of type `public` |
| hooks at admin locations | hooks at client area locations (`public_profileproducttab`, `client_top_submitticket`, `clientarea_all`, `clientarea_footer`) |
| | `publicmain` boxes |

A snapin with no client area features has no "Viewable by all customers" setting and never appears in customer group permissions.

Access to the snapin is coarse: it says the user may use *this snapin*, not that they may see *this record*. Everything finer is your job (section 8).

---

## 5. Views and how they're reached

| Where | URL | Renders |
| --- | --- | --- |
| Admin | `index.php?fuse=admin&controller=snapins&view=viewsnapin&plugin=<name>&v=<tpl>` | any view the snapin registered |
| Client area | `index.php?fuse=admin&controller=snapins&view=snapin&plugin=<name>&v=<tpl>` | only views registered with `addMappingForPublicMain()`, `addMappingForTopMenu('public', ...)`, or listed in `$publicViews` |

Menu entries and home page boxes link with an `h=` hash instead of `v=`. Both work.

Pages are requested with `view=`. Actions that change something are requested with `action=` and require the session hash.

### Rendering

```php
public function status(): void
{
    $this->view->incidents = $this->fetchIncidents();
    $this->view->updated = date('Y-m-d H:i');
}
```

The method sets view variables, then `status.phtml` is rendered. Inside the template, use `$this->incidents` and `$this->updated`.

Escaping follows the usual rule (conventions, section 8): top-level view variables are escaped automatically, values inside arrays are not.

| Property | Effect |
| --- | --- |
| `public $disableLayout = true;` (or set it in the method) | Return only the snapin's output, without the page layout. Admin views only. |
| `public $overrideTemplate = true;` | Don't render `<tpl>.phtml`; use the method's return value as the output. |

### Extra client area views: `$publicViews`

```php
public $publicViews = ['incident'];
```

Lists views that may be rendered in the client area besides the ones registered as public. Only add views that are meant for clients and do their own checks (section 8).

---

## 6. Admin AJAX: `callAction()`

Admin pages call back into the snapin through:

```
index.php?fuse=admin&controller=plugin&action=doplugin&type=snapin&plugin=<name>&pluginaction=<what>
```

which calls `callAction()`. Branch on `$_REQUEST['pluginaction']`:

```php
public function callAction($callback = true): void
{
    if (NE_PUBLIC) {
        return;
    }

    if (!$this->user->canUseSnapin($this->getInternalName())) {
        throw new CE_ExceptionPermissionDenied('You do not have access to perform that action');
    }

    switch ($_REQUEST['pluginaction'] ?? '') {
        case 'clearcache':
            $this->clearCache();
            $this->send([], false, $this->user->lang('Cache cleared'));
            break;
    }
}
```

`$this->send($data, $error, $message)` outputs the JSON response.

Start every `callAction()` with the `NE_PUBLIC` guard and a `canUseSnapin()` check. `callAction()` is meant for admin pages only, and the guard keeps it that way even if a future route reaches it from the client area. Use `canUseSnapin()` rather than `checkPermissions('snapin_<name>')`: it also honors "Viewable by all staff", which a plain permission check doesn't.

---

## 7. Client area AJAX: `$publicActions`

Client area pages call:

```
index.php?fuse=admin&controller=snapins&action=doaction&plugin=<name>&do=<method>
```

Only methods listed in `$publicActions` can be called. Each receives `$_REQUEST` and returns `['message' => ..., 'data' => [...]]`; anything under `data` is merged into the top level of the JSON response.

```php
public $publicActions = ['refreshStatus'];

public function refreshStatus(array $request): array
{
    if ($this->user->isAnonymous()) {
        throw new CE_Exception($this->user->lang('You must be logged in to do that.'));
    }

    $this->clearCache();

    return [
        'message' => $this->user->lang('Status refreshed'),
        'data' => ['incidents' => $this->fetchIncidents()]
    ];
}
```

| The method throws | The client sees |
| --- | --- |
| `CE_Exception` | its message, as the error |
| anything else | "An error occurred while processing your request." (the real error is logged) |

Call it from the page with jQuery. The session hash is added automatically:

```javascript
$.post('index.php?fuse=admin&controller=snapins&action=doaction', { plugin: 'acmestatus', do: 'refreshStatus' }, function (response) {
    clientexec.parseResponse(response);
}, 'json');
```

---

## 8. Security rules

1. **Anyone who can use the snapin can call every method in `$publicActions` and render every public view**, including visitors who aren't logged in when "Viewable by all customers" is on. If a method or view is for logged-in clients, check `$this->user->isAnonymous()` first, as `csfunblock` does.
2. **Check ownership of anything named in the request.** A package ID from `$this->getParam('id')` in a client area hook, or any ID passed to a public action, must belong to `$this->user` before you show or change it.
3. **Keep helpers private.** Only methods in `$publicActions` are callable from the client area, but private is still the right default for everything that isn't a view, a hook, a listener, or an action.
4. **Admin code checks permissions too.** Start `callAction()` with the `NE_PUBLIC` guard and `canUseSnapin()`. In admin views and actions that expose more than the snapin itself implies (billing data, client details), also call `checkPermissions('<permission>')` for the matching core permission.
5. **Escape view output** inside arrays (conventions, section 8).
6. **Treat event payloads as hints.** Re-read the record from the database and check its state before acting (section 9).

---

## 9. Events

To run code when something happens in Clientexec, list `[event name, method]` pairs in `$listeners`:

```php
public $listeners = [
    ['Ticket-CreateByClient', 'onTicketCreated'],
    ['Invoice-Paid', 'onInvoicePaid'],
];

public function onInvoicePaid($e): void
{
    $params = $e->getParams();
    $invoiceId = (int) ($params['invoiceId'] ?? 0);

    // ...
}
```

The method receives a `Zend_EventManager_Event`. `$e->getParams()` returns the event's data; `$e->getTarget()` returns the object that fired it. Every event and its data is listed in `events.md`.

**How listeners are registered:**
- On every request handled by a Clientexec controller, listeners of every **enabled** snapin are attached. That includes scheduled service runs, since `cron.php` routes through a controller. REST API requests don't attach them.
- The list of snapins with listeners is cached in the user's session. After enabling a snapin or adding a listener, save the snapin settings or start a new session before testing.
- The snapin instance delivering an event is created without calling `init()`. Settings (`getVariable()`), `$this->user`, and `$this->db` work; mappings and anything else set up in `init()` don't.

**Rules for listeners:**
- **They run inside the request that fired the event.** A slow or failing listener slows down or breaks whatever the user was doing: paying an invoice, opening a ticket. Keep them fast, use short timeouts on any outbound call, and catch every exception.
- **Re-read and check state.** An `Invoice-Paid` listener should load the invoice and confirm it's paid before doing anything with lasting effect. Don't act on an ID alone.
- **`Client-PasswordChange` carries the new password in plain text.** Never log it.
- **Return values are ignored**, except for the two events that use them (see `events.md`).

---

## 10. Complete example

A fictional "Acme Status" snapin that shows a service status feed:
- on the admin dashboard (hook);
- on an admin page with a cache button (admin menu, admin AJAX);
- on a client area page and home page box, with a refresh action (public view, public action);
- and posts new client tickets to a chat webhook (event listener).

### `plugins/snapin/acmestatus/PluginAcmestatus.php`

```php
<?php

require_once 'modules/admin/models/SnapinPlugin.php';

class PluginAcmestatus extends SnapinPlugin
{
    public $title = 'Acme Status';

    public $publicActions = ['refreshStatus'];

    public $listeners = [
        ['Ticket-CreateByClient', 'onTicketCreated']
    ];

    public function getVariables()
    {
        $variables = [
            'Plugin Name' => [
                'type' => 'hidden',
                'description' => '',
                'value' => 'Acme Status'
            ],
            'Public Description' => [
                'type' => 'hidden',
                'description' => 'Description to be seen by public',
                'value' => 'Service status'
            ],
            'Status API URL' => [
                'type' => 'text',
                'description' => lang('URL of your Acme Status JSON feed (https only)'),
                'value' => ''
            ],
            'Ticket Webhook URL' => [
                'type' => 'text',
                'description' => lang('Optional chat webhook that is told about new client tickets (https only)'),
                'value' => ''
            ]
        ];

        return $variables;
    }

    public function init()
    {
        $this->setDescription('Shows service status to staff and clients');
        $this->addMappingHook('admin_top_dashboard', 'widget', 'Service Status', 'Shows current service status on the admin dashboard');
        $this->addMappingForTopMenu('admin', 'tools', 'incidents', 'Service Status', 'Lists current incidents for staff');
        $this->addMappingForTopMenu('public', 'support', 'status', 'Service Status', 'Shows current service status to clients');
        $this->addMappingForPublicMain('status', 'Service Status', 'See current service status', 'fa fa-heartbeat');
        $this->addMappingForSettings('clearcache', 'Clear Cache', 'Clears the cached status feed.');
    }

    public function widget(): void
    {
        $this->view->incidents = $this->fetchIncidents();
    }

    public function incidents(): void
    {
        $this->view->incidents = $this->fetchIncidents();
    }

    public function status(): void
    {
        $this->view->incidents = $this->fetchIncidents();
        $this->view->loggedIn = !$this->user->isAnonymous();
    }

    public function callAction($callback = true): void
    {
        if (NE_PUBLIC) {
            return;
        }

        if (!$this->user->canUseSnapin($this->getInternalName())) {
            throw new CE_ExceptionPermissionDenied('You do not have access to perform that action');
        }

        switch ($_REQUEST['pluginaction'] ?? '') {
            case 'clearcache':
                $this->clearCache();
                $this->send([], false, $this->user->lang('Cache cleared'));
                break;
        }
    }

    public function refreshStatus(array $request): array
    {
        if ($this->user->isAnonymous()) {
            throw new CE_Exception($this->user->lang('You must be logged in to do that.'));
        }

        $this->clearCache();

        return [
            'message' => $this->user->lang('Status refreshed'),
            'data' => ['incidents' => $this->fetchIncidents()]
        ];
    }

    public function onTicketCreated($e): void
    {
        $webhookUrl = trim((string) $this->getVariable('Ticket Webhook URL'));
        if ($webhookUrl === '' || stripos($webhookUrl, 'https://') !== 0) {
            return;
        }

        $params = $e->getParams();
        $ticketId = (int) ($params['ticketId'] ?? 0);
        if ($ticketId <= 0) {
            return;
        }

        $ticketUrl = rtrim(CE_Lib::getSoftwareURL(), '/') . NE_CONTROLLER_ADMIN_DIR . '/index.php?fuse=support&controller=ticket&view=viewtickets&id=' . $ticketId;

        try {
            $this->postJson($webhookUrl, ['text' => 'New client ticket #' . $ticketId . ': ' . $ticketUrl], 5);
        } catch (Throwable $ex) {
            CE_Lib::log(1, 'plugin_acmestatus::onTicketCreated: webhook failed for ticket ' . $ticketId . ': ' . $ex->getMessage());
        }
    }

    private function fetchIncidents(): array
    {
        $cacheKey = 'plugin_acmestatus_cache';
        $cached = $this->settings->get($cacheKey);
        if ($cached != '') {
            $decoded = json_decode($cached, true);
            if (is_array($decoded) && isset($decoded['time'], $decoded['incidents']) && $decoded['time'] >= time() - 300) {
                return $decoded['incidents'];
            }
        }

        $url = trim((string) $this->getVariable('Status API URL'));
        if ($url === '' || stripos($url, 'https://') !== 0) {
            return [];
        }

        try {
            $feed = $this->getJson($url, 10);
        } catch (Throwable $ex) {
            CE_Lib::log(1, 'plugin_acmestatus::fetchIncidents: ' . $ex->getMessage());
            return [];
        }

        $incidents = [];
        foreach ($feed['incidents'] ?? [] as $incident) {
            $incidents[] = [
                'title' => (string) ($incident['title'] ?? ''),
                'status' => (string) ($incident['status'] ?? ''),
                'updated' => (string) ($incident['updated_at'] ?? '')
            ];
        }

        $this->settings->updateValue($cacheKey, json_encode(['time' => time(), 'incidents' => $incidents]));

        return $incidents;
    }

    private function clearCache(): void
    {
        $this->settings->updateValue('plugin_acmestatus_cache', '');
    }

    private function getJson(string $url, int $timeout): array
    {
        return $this->request($url, null, $timeout);
    }

    private function postJson(string $url, array $body, int $timeout): void
    {
        $this->request($url, $body, $timeout);
    }

    private function request(string $url, ?array $body, int $timeout): array
    {
        $ch = curl_init();
        curl_setopt($ch, CURLOPT_URL, $url);
        curl_setopt($ch, CURLOPT_RETURNTRANSFER, true);
        curl_setopt($ch, CURLOPT_SSL_VERIFYPEER, true);
        curl_setopt($ch, CURLOPT_SSL_VERIFYHOST, 2);
        curl_setopt($ch, CURLOPT_CAINFO, \Composer\CaBundle\CaBundle::getSystemCaRootBundlePath());
        curl_setopt($ch, CURLOPT_TIMEOUT, $timeout);
        curl_setopt($ch, CURLOPT_PROTOCOLS, CURLPROTO_HTTPS);
        if ($body !== null) {
            curl_setopt($ch, CURLOPT_POST, true);
            curl_setopt($ch, CURLOPT_POSTFIELDS, json_encode($body));
            curl_setopt($ch, CURLOPT_HTTPHEADER, ['Content-Type: application/json']);
        }

        $response = curl_exec($ch);
        if ($response === false) {
            $error = curl_error($ch);
            curl_close($ch);
            throw new CE_Exception('Request to ' . parse_url($url, PHP_URL_HOST) . ' failed: ' . $error);
        }

        $status = curl_getinfo($ch, CURLINFO_HTTP_CODE);
        curl_close($ch);

        if ($status < 200 || $status >= 300) {
            throw new CE_Exception('Request to ' . parse_url($url, PHP_URL_HOST) . ' returned HTTP ' . $status);
        }

        $decoded = json_decode($response, true);

        return is_array($decoded) ? $decoded : [];
    }
}
```

### `plugins/snapin/acmestatus/status.phtml`

```php
<div class="card">
    <div class="card-body">
        <?php if (count($this->incidents) === 0) { ?>
            <p><?php echo $this->user->lang('All systems are operating normally.'); ?></p>
        <?php } else { ?>
            <ul class="list-unstyled" id="acmestatus-incidents">
                <?php foreach ($this->incidents as $incident) { ?>
                    <li>
                        <strong><?php echo $this->escape($incident['title']); ?></strong>
                        (<?php echo $this->escape($incident['status']); ?>)
                        <small class="text-muted"><?php echo $this->escape($incident['updated']); ?></small>
                    </li>
                <?php } ?>
            </ul>
        <?php } ?>

        <?php if ($this->loggedIn) { ?>
            <button type="button" class="btn btn-outline-primary" id="acmestatus-refresh">
                <?php echo $this->user->lang('Refresh'); ?>
            </button>
        <?php } ?>
    </div>
</div>
```

### `plugins/snapin/acmestatus/status.js`

```javascript
$(function () {
    $('#acmestatus-refresh').on('click', function () {
        clientexec.mask();
        $.post('index.php?fuse=admin&controller=snapins&action=doaction', { plugin: 'acmestatus', do: 'refreshStatus' }, function (response) {
            clientexec.parseResponse(response);
            if (response.success == true) {
                location.reload();
            } else {
                clientexec.unMask();
            }
        }, 'json');
    });
});
```

`widget.phtml` and `incidents.phtml` render the same `incidents` list for staff. Add an empty `index.html`.

---

## 11. Checklist before release

- [ ] Mappings registered in `init()`; nothing a listener needs comes from `init()`.
- [ ] No `Enabled`, `Viewable by all staff`, or `Viewable by all customers` in `getVariables()`.
- [ ] Every method in `$publicActions` and every public view checks `isAnonymous()` if it's for logged-in clients, and checks ownership of every ID it's given.
- [ ] `$publicViews` lists only views meant for clients.
- [ ] `callAction()` starts with the `NE_PUBLIC` guard and a `canUseSnapin()` check.
- [ ] Helpers are private.
- [ ] Listeners are fast, use short timeouts, catch every exception, and re-read state instead of trusting the payload.
- [ ] No passwords or secrets in logs.
- [ ] Template output inside arrays escaped.
- [ ] TLS verification on, using the Composer CA bundle.

---

## Canonical reference files

| What | File |
| --- | --- |
| Base class: mappings, rendering, helpers | `modules/admin/models/SnapinPlugin.php` |
| Admin view route | `modules/admin/controllers/SnapinsController.php` (`viewsnapinAction()`) |
| Client area view and action routes | `modules/admin/controllers/SnapinspublicController.php` |
| Access check | `User::canUseSnapin()` |
| Enabled snapins, hook mappings | `modules/admin/models/PluginGateway.php` |
| Listener registration | `library/CE/Controller/Action.php` |
| Firing events | `CE_Lib::trigger()` in `library/CE/Lib.php` |
| Client area page with a login check | `plugins/snapin/csfunblock/` |
| Admin page, admin tab, admin AJAX | `plugins/snapin/cetransactions/` |
| Event listeners | `plugins/snapin/ownedsupport/`, `plugins/snapin/clicksend/` |
