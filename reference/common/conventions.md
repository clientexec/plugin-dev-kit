# Clientexec Plugin Conventions

Rules shared by every plugin type. Load this alongside the reference for the type you're building.

## 1. Where plugins live and how they're named

Every plugin is a directory under `plugins/<type>/`:

| Type | Directory | Base class | Base class file |
| --- | --- | --- | --- |
| Server (provisioning) | `plugins/server/` | `ServerPlugin` | `modules/admin/models/ServerPlugin.php` |
| Registrar | `plugins/registrars/` | `RegistrarPlugin` | `modules/admin/models/RegistrarPlugin.php` |
| Payment gateway | `plugins/gateways/` | `GatewayPlugin` | `modules/admin/models/GatewayPlugin.php` |
| Snapin | `plugins/snapin/` | `SnapinPlugin` | `modules/admin/models/SnapinPlugin.php` |
| Service (scheduled task) | `plugins/services/` | `ServicePlugin` | `modules/admin/models/ServicePlugin.php` |
| Fraud | `plugins/fraud/` | `FraudPlugin` | `modules/admin/models/FraudPlugin.php` |
| Captcha | `plugins/captcha/` | `CaptchaPlugin` | `modules/admin/models/CaptchaPlugin.php` |
| SSL | `plugins/ssl/` | `SSLPlugin` | `modules/admin/models/SSLPlugin.php` |
| Import / Export | `plugins/import/`, `plugins/export/` | `ImportPlugin`, `ExportPlugin` | `modules/admin/models/` |

Naming is strict and derived from the directory name:

- Directory: all lowercase, letters and digits only. Example: `acmepanel`.
- Main file: `Plugin` + directory name with the first letter uppercased. Example: `PluginAcmepanel.php`.
- Class: same as the file name. Example: `class PluginAcmepanel extends ServerPlugin`.

PHP class names are case insensitive, so a few shipped plugins use other casing (`PluginDirectAdmin`), but new plugins should follow the rule exactly.

All of these base classes extend `NE_Plugin` (`library/CE/NE_Plugin.php`), which gives every plugin `$this->user` (the acting `User`), `$this->settings` (application settings), and `trackUsage()`.

## 2. Standard directory contents

```
plugins/server/acmepanel/
├── PluginAcmepanel.php       # the plugin class (required)
├── AcmepanelApi.php          # your API client, kept separate from the plugin class (recommended)
├── index.html                # empty file, prevents directory listing (required)
├── logo.png                  # shown in the admin plugin list
├── views/                    # .phtml templates for client area output (optional)
└── assets/                   # JS/CSS loaded by your views (optional)
```

Include an empty `index.html` in every directory you add.

Some official plugins also have a `resource/` directory with a `plugin.ini` and screenshots. That metadata is only used to list official plugins on the Clientexec website. Third-party plugins don't need it, and Clientexec doesn't read it at runtime.

Load your own files with paths relative to the plugin file, and core files with paths relative to the application root:

```php
require_once 'modules/admin/models/ServerPlugin.php';
require_once __DIR__ . '/AcmepanelApi.php';
```

If you bundle Composer dependencies, keep them inside the plugin directory and load the autoloader relative to it (`enhance` does this with `api/vendor/autoload.php`).

## 3. Settings: getVariables()

Every plugin declares its settings in `getVariables()`, which returns an array keyed by setting name. Clientexec renders the form, stores the values, and passes them back to you.

```php
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
    ];

    return $variables;
}
```

### How a setting's stored key is built

Clientexec stores each setting as `plugin_<directory>_<setting name with spaces replaced by underscores>`. Other characters are kept as they are.

| Plugin directory | Setting name | Key you read |
| --- | --- | --- |
| `hestiacp` | `Access Key` | `plugin_hestiacp_Access_Key` |
| `cpanel` | `Failure E-mail` | `plugin_cpanel_Failure_E-mail` |
| `linode` | `Prepend Package ID to Hostname` | `plugin_linode_Prepend_Package_ID_to_Hostname` |

Because the key is derived from the setting name, renaming a setting orphans every value already saved under the old name. Treat setting names as permanent once released.

Use plain string keys for setting names. Shipped plugins historically wrapped keys in `lang()`, but the key is part of the storage identifier, so it shouldn't depend on translation. Translate descriptions and visible values instead.

> **Note on `lang()` in getVariables():** everywhere else in plugin code, translate with `$this->user->lang()`. `getVariables()` is the one place every shipped plugin uses the global `lang()`. Follow the shipped pattern there until this kit says otherwise.

### Field types

These are the `type` values in use across shipped plugins:

| Type | Renders as | Notes |
| --- | --- | --- |
| `text` | single-line input | most common |
| `password` | masked input | pair with `'encryptable' => true` |
| `textarea` | multi-line input | used for long tokens (cPanel API token, Hestia secret key) |
| `yesno` | yes/no toggle | stored as `'1'` / `'0'` |
| `check` | checkbox | common in nested per-package fields |
| `options` | select | requires an `'options' => ['value' => 'Label', ...]` array |
| `dropdown` | select populated by a method | used inside `package_vars_values`; see the server reference |
| `hidden` | not shown | used for metadata Clientexec reads (`Name`, `Description`, `Actions`, and others) |
| `label` | static text | |
| `fieldset` | groups the fields that follow | `cpanel` uses it for reseller fields |

Other keys a setting can carry: `description` (help text, may contain simple HTML), `value` (default), `encryptable` (store encrypted), `label` (used on nested fields), and `helpid` (used by service plugins).

### Secrets

Mark every credential `'encryptable' => true`. Clientexec encrypts it at rest and hands it to your plugin already decrypted. If you need to store a secret yourself somewhere else (a custom field, your own table), encrypt it with `Clientexec::encryptString()` and decrypt with `Clientexec::decryptString()`. Never write a secret to a log or an error email: unset it from the array first (see section 5).

## 4. Errors

Throw `CE_Exception` with a message an admin can act on. This is the current pattern across shipped plugins, and Clientexec catches it and shows the message.

```php
try {
    $this->api->suspendAccount($username);
} catch (Exception $e) {
    CE_Lib::log(1, 'plugin_acmepanel::suspend::error: ' . $e->getMessage());
    throw new CE_Exception('AcmePanel could not suspend ' . $username . ': ' . $e->getMessage());
}
```

Older plugins return `CE_Error` objects or error strings, and some base class docblocks still describe that contract. Don't write new code that way. When a core helper returns a `CE_Error` (some network helpers do), convert it:

```php
if ($response instanceof CE_Error) {
    throw new CE_Exception('AcmePanel request failed: ' . $response);
}
```

`CE_Error` converts to its message when used as a string, which is how shipped plugins report it.

For partial failures (step one worked, step two didn't), say exactly what state things were left in. `hestiacp` is a good model: if the account is created but adding the domain fails, the exception says the account exists and the domain must be added by hand.

## 5. Logging

Use `CE_Lib::log($level, $message)`. Shipped plugins use level `1` for errors and level `4` for debug tracing, almost exclusively.

```php
CE_Lib::log(4, 'plugin_acmepanel::create: creating account ' . $username);
CE_Lib::log(1, 'plugin_acmepanel::create::error: ' . $e->getMessage());
```

Prefix messages with `plugin_<directory>::<method>` so they're easy to grep.

Before logging an `$args` array, remove secrets:

```php
unset($args['server']['variables']['plugin_acmepanel_API_Token']);
CE_Lib::log(1, 'AcmePanel error: ' . print_r($args, true));
```

Don't use `CE_Lib::debug()` in new code; it survives only in base class stubs.

## 6. Outbound HTTP

Keep TLS verification on and verify against the Composer CA bundle that ships with Clientexec. Don't tell admins to configure `php.ini` CA paths, and never set `CURLOPT_SSL_VERIFYPEER` to `false`.

```php
$ch = curl_init();
curl_setopt($ch, CURLOPT_URL, $url);
curl_setopt($ch, CURLOPT_RETURNTRANSFER, true);
curl_setopt($ch, CURLOPT_SSL_VERIFYPEER, true);
curl_setopt($ch, CURLOPT_SSL_VERIFYHOST, 2);
curl_setopt($ch, CURLOPT_CAINFO, \Composer\CaBundle\CaBundle::getSystemCaRootBundlePath());
curl_setopt($ch, CURLOPT_TIMEOUT, 60);
```

Put HTTP code in a separate API client class, not in the plugin class. `hestiacp/HestiacpApi.php` is a clean example, including how it refuses to treat an unexpected response body as success.

## 7. Translation

| Where | Use |
| --- | --- |
| Plugin PHP code (messages, labels, exceptions shown to people) | `$this->user->lang('...')` |
| `getVariables()` descriptions and values | `lang('...')` (see the note in section 3) |
| `.phtml` views | `$this->user->lang('...')` |
| Client side JavaScript | `clientexec.lang('...')` |

`lang()` uses `vsprintf`, so placeholders are `%s` and `%d`. Put dynamic values inside the call rather than concatenating outside it, so translators see the whole sentence:

```php
throw new CE_Exception($this->user->lang('Account %s could not be suspended', $username));
```

You don't need to register new strings anywhere. Clientexec's build process extracts them.

## 8. Views and output escaping

Plugins render `.phtml` templates through the `$view` object Clientexec passes in:

```php
$view->addScriptPath(__DIR__ . '/views/');
$view->rows = $rows;
return $view->render('usage.phtml');
```

Escaping rule:

- **Top-level view variables** (`$view->title = ...`, read in the template as `$this->title`) are escaped automatically. Don't wrap them in `htmlspecialchars()`, or you'll double-escape.
- **Values inside arrays or objects** you assign to the view are **not** escaped. Escape each one when you output it, with `$this->escape()` or `htmlspecialchars($value, ENT_QUOTES | ENT_SUBSTITUTE, 'UTF-8')`.

```php
<?php foreach ($this->rows as $row) { ?>
    <tr>
        <td><?php echo $this->escape($row['label']); ?></td>
        <td><?php echo (int) $row['percent']; ?>%</td>
    </tr>
<?php } ?>
```

Cast numbers you print into JavaScript (`<?php echo (int) $this->userPackageId; ?>`).

Front end notes:

- The client area uses Bootstrap 4 (`data-toggle`, `data-dismiss`, `mr-2`). Don't use Bootstrap 5 attributes in client area views.
- jQuery is available. The `clientexec` global provides `mask()`, `unMask()`, `parseResponse(response)`, and `lang()`.
- Client side confirmations use the jQuery `.confirm()` plugin, never the browser's native `confirm()`.
- Version your script URLs so browsers pick up changes: `assets/panels.js?v=<?php echo (int) filemtime(...); ?>`.

## 9. URLs

- `controller=` values never include "public". A public controller class such as `Admin_SnapinspublicController` is still addressed as `controller=snapins`.
- Don't hardcode `/admin/` in URLs. Use the `NE_CONTROLLER_ADMIN_DIR` constant.

## 10. Database access

Prefer Clientexec's models (`UserPackage`, `User`, `Package`, `Server`) over raw queries. When you do need a query, use the Laravel query builder through Capsule, imported as `Db`:

```php
use Illuminate\Database\Capsule\Manager as Db;

$count = Db::table('users')->where('status', 1)->count();
```

Clientexec tables have no prefix (`users`, not `tblusers`).

## 11. Code style

- PHP 8.2 compatible at minimum; avoid anything deprecated in 8.4 or 8.5.
- Short array syntax `[]`, never `array()`.
- Braces for every control structure, never the alternative `:` / `endif` syntax, including in `.phtml` files.
- `public`, `protected`, or `private` on every method.
- Declare return types on new methods. On methods you override, you may add a return type, but **do not add parameter types**: the parent methods are untyped, and narrowing a parameter type in a child class is a fatal error.
- No closing `?>` at the end of PHP-only files.
- Use `mb_*` string functions only for values that can be multibyte (names, descriptions). Plain functions are fine for ASCII-only values like usernames you've already validated.
- Keep a fetch and its immediate check together, with no blank line between them:

```php
$instance = $this->api->getInstance($instanceId);
if ($instance === null) {
    throw new CE_Exception('Instance ' . $instanceId . ' was not found on Linode');
}
```
