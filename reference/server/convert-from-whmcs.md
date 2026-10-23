# Converting a WHMCS Provisioning Module to a Clientexec Server Plugin

Load `reference/common/conventions.md` and `reference/server/create.md` first. This file only maps WHMCS concepts onto them.

## The short version

A WHMCS provisioning module is a set of prefixed global functions that receive `$params` and return `'success'` or an error string. A Clientexec server plugin is a class with paired `doX()` / `x()` methods that receive `$args`, return nothing (or a message), and throw `CE_Exception` on failure.

The API client code (the part that talks to the remote panel) usually ports almost unchanged. Move it into its own class file (`AcmepanelApi.php`), then rewrite the module functions as plugin methods.

**Don't port by translating line by line.** Read the WHMCS module to learn what the remote API expects, then write the Clientexec plugin from `create.md`'s example. Line-by-line ports carry over WHMCS assumptions (database tables, `localAPI`, `$params` keys) that don't exist in Clientexec.

---

## 1. File and structure

| WHMCS | Clientexec |
| --- | --- |
| `modules/servers/acmepanel/acmepanel.php` | `plugins/server/acmepanel/PluginAcmepanel.php` |
| Global functions prefixed `acmepanel_` | Methods on `class PluginAcmepanel extends ServerPlugin` |
| `acmepanel_MetaData()` | the `$features` property |
| `lib/` helper classes | Same, loaded with `require_once __DIR__ . '/...'` |
| `templates/*.tpl` (Smarty) | `views/*.phtml` (PHP templates) |
| `lang/english.php` | `$this->user->lang('English text')`; strings are extracted automatically |
| `logo.png` | `logo.png` |

---

## 2. Function mapping

| WHMCS function | Clientexec equivalent |
| --- | --- |
| `_MetaData()` | `$features` (display name and description come from the `Name` and `Description` settings) |
| `_ConfigOptions()` | `package_vars_values` in `getVariables()` (per-package fields) |
| `_CreateAccount($params)` | `doCreate($args)` + `create($args)` |
| `_SuspendAccount($params)` | `doSuspend($args)` + `suspend($args)` |
| `_UnsuspendAccount($params)` | `doUnSuspend($args)` + `unsuspend($args)` |
| `_TerminateAccount($params)` | `doDelete($args)` + `delete($args)` |
| `_ChangePassword($params)` | `update($args)`, case `password` |
| `_ChangePackage($params)` | `update($args)`, case `package`; enable `'upgrades' => true` |
| `_TestConnection($params)` | `testConnection($args)`; enable `'testConnection' => true` |
| `_ServiceSingleSignOn($params)` / `_LoginLink($params)` | `getDirectLink()` + `dopanellogin()`; enable `'directlink' => true` |
| `_AdminSingleSignOn($params)` | `getAdminDirectLink($args)`; enable `'admindirectlink' => true` |
| `_AdminCustomButtonArray()` | names added to the `Actions` setting, each with a `do<Name>($args)` method |
| `_ClientAreaCustomButtonArray()` | names added to `Registered Actions For Customer`, each with a `do<Name>($args)` method |
| `_ClientArea($params)` | `getPublicView()` for the main package page, or `publicPanels` methods for tabs |
| `_AdminServicesTabFields` / `_AdminServicesTabFieldsSave` | No direct equivalent documented yet; store values in package custom fields |
| `_UsageUpdate($params)` | No direct equivalent documented yet |
| `_ListAccounts` (account sync / import) | No generic equivalent; `getAccounts()` is only used by the cPanel importers |

Anything in the "no direct equivalent" rows: stop and ask rather than inventing a Clientexec hook.

---

## 3. `$params` to `$args`

| WHMCS `$params` | Clientexec `$args` |
| --- | --- |
| `serviceid` | `$args['package']['id']` |
| `pid` | no direct key; per-package configuration arrives in `$args['package']['variables']` |
| `userid` | `$args['customer']['id']` |
| `clientsdetails['email']` | `$args['customer']['email']` |
| `clientsdetails['firstname']` | `$args['customer']['first_name']` |
| `clientsdetails['lastname']` | `$args['customer']['last_name']` |
| `clientsdetails['companyname']` | `$args['customer']['organization']` |
| `domain` | `$args['package']['domain_name']` |
| `username` | `$args['package']['username']` |
| `password` | `$args['package']['password']` (already decrypted) |
| `dedicatedip` | `$args['package']['ip']` |
| `configoption1` ... `configoption24` | `$args['package']['variables']['<your field key>']` |
| `configoptions` (configurable options) | `$args['package']['addons']['<NAME>']`, for names declared in `package_addons` |
| `customfields` | `$args['package']['customfields']`, or `$userPackage->getCustomField($name, CUSTOM_FIELDS_FOR_PACKAGE)` |
| `serverhostname` / `serverip` | `$args['server']['variables']['ServerHostName']` |
| `serverusername` | your own setting, e.g. `plugin_acmepanel_Username` |
| `serverpassword` | your own setting, marked `'encryptable' => true` |
| `serveraccesshash` | your own setting (`textarea` for long tokens), `'encryptable' => true` |
| `serversecure` | your own `yesno` setting |
| `serverport` | your own `text` setting with a default |

### Server credentials

WHMCS gives every server a fixed set of fields (hostname, username, password, access hash, secure, port) and modules pick from them. Clientexec has the hostname as a core field and everything else comes from your `getVariables()`. Declare exactly the credentials your API needs, with names that make sense for that panel ("API Token", not "Access Hash" unless the panel calls it that).

### `configoptionN` to named fields

WHMCS config options are positional (`configoption1`, `configoption2`, ...). Clientexec per-package fields are named. Give each one a meaningful key:

```php
// WHMCS
function acmepanel_ConfigOptions()
{
    return [
        'Plan' => ['Type' => 'dropdown', 'Loader' => 'acmepanel_LoadPlans'],
        'Swap (MB)' => ['Type' => 'text', 'Default' => '512'],
    ];
}
// used as $params['configoption1'], $params['configoption2']
```

```php
// Clientexec, inside getVariables()
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
            'description' => '',
            'value' => ''
        ],
        'swap' => [
            'type' => 'text',
            'label' => lang('Swap (MB)'),
            'description' => '',
            'value' => '512'
        ]
    ]
]
// used as $args['package']['variables']['plan'], $args['package']['variables']['swap']
```

A WHMCS `Loader` function becomes a plugin method named in `getValues`. It receives the server ID (not `$params`) and returns `value => label`. See `create.md`, section 4.

If a WHMCS config option really names a remote plan, consider enabling `'packageName' => true` instead and reading `$args['package']['name_on_server']`. That's how cPanel, HestiaCP, and Enhance do it.

---

## 4. Return values and errors

WHMCS:

```php
function acmepanel_SuspendAccount(array $params)
{
    try {
        $api = new AcmeApi($params['serverhostname'], $params['serveraccesshash']);
        $api->suspend($params['username']);
    } catch (Exception $e) {
        logModuleCall('acmepanel', __FUNCTION__, $params, $e->getMessage(), $e->getTraceAsString());
        return $e->getMessage();
    }
    return 'success';
}
```

Clientexec:

```php
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
        CE_Lib::log(1, 'plugin_acmepanel::suspend::error: ' . $e->getMessage());
        throw new CE_Exception('AcmePanel could not suspend ' . $username . ': ' . $e->getMessage());
    }
}
```

| WHMCS | Clientexec |
| --- | --- |
| `return 'success';` | return nothing from `x()`; return a message string from `doX()` |
| `return 'Some error';` | `throw new CE_Exception('Some error');` |
| `_TestConnection` returns `['success' => false, 'error' => '...']` | `testConnection()` throws `CE_Exception` |
| `logModuleCall(...)` | `CE_Lib::log(1 or 4, ...)`, with secrets unset first |
| `_ServiceSingleSignOn` returns `['success' => true, 'redirectTo' => $url]` | `getDirectLink()` returns `['fa' => ..., 'link' => $url, 'text' => ..., 'form' => '']` |

`logModuleCall()` accepts a list of values to mask. `CE_Lib::log()` doesn't, so remove secrets from arrays yourself before logging them.

---

## 5. Things WHMCS modules do that you shouldn't port

**Writing to WHMCS tables.** Modules often update `tblhosting` directly to save a username, IP, or remote ID. In Clientexec, use the package model:

| WHMCS | Clientexec |
| --- | --- |
| `Capsule::table('tblhosting')->where('id', $serviceid)->update(['username' => $u])` | `$userPackage->setCustomField('User Name', $u);` |
| `... ->update(['dedicatedip' => $ip])` | `$userPackage->setCustomField('IP Address', $ip);` |
| storing a remote ID in a custom field or `tblhosting.notes` | `$userPackage->setCustomField('Server Acct Properties', $remoteId);` or a package custom field named in your settings |

Clientexec does use Laravel's query builder, imported as `use Illuminate\Database\Capsule\Manager as Db;`, but its tables are different and have no prefix. Don't carry over any `tbl*` table names.

**`localAPI()`.** There's no Clientexec equivalent. Use the models (`UserPackage`, `User`, `Package`, `Server`).

**`$_SESSION`, `$_GET`, `$_POST` in client area code.** Client area actions receive their input as the `$request` array passed to the panel action method. Use `$this->getSession()` for session data.

**Disabling TLS verification.** Many WHMCS modules set `CURLOPT_SSL_VERIFYPEER` to `false`. Don't. Use the Composer CA bundle (conventions, section 6).

**Hooks inside the module file.** WHMCS lets a module register hooks in `hooks.php`. Server plugins don't have hooks; event-driven behavior belongs in a snapin, which is a separate plugin type (reference coming).

---

## 6. Client area

| WHMCS | Clientexec |
| --- | --- |
| `_ClientArea()` returning `['templatefile' => 'overview', 'vars' => [...]]` | `getPublicView($userPackage, $view)` rendering `views/overview.phtml` |
| A custom page via `_ClientAreaAllowedFunctions` / `modop=custom&a=...` | A panel in `publicPanels`, rendered by a method of the same name |
| AJAX to `clientarea.php?action=productdetails&id=...&modop=custom&a=foo` | POST to `index.php?fuse=clients&controller=products&action=panelaction` with `id`, `panel`, `do`; `do` must be listed in `publicPanelActions` |
| Smarty `{$var}` | `<?php echo $this->var; ?>` (top-level vars are escaped automatically) |
| Smarty `{$item.name}` in a loop | `<?php echo $this->escape($item['name']); ?>` (array values are not escaped) |
| Smarty `{$LANG.key}` | `<?php echo $this->user->lang('English text'); ?>` |
| Bootstrap 3/4 markup (WHMCS Six / Twenty-One themes) | Bootstrap 4 markup |

See `create.md`, section 9 for the full panel and panel action pattern, including validating that every value in `$request` belongs to the client's account.

---

## 7. Porting checklist

- [ ] API client moved into its own class file, TLS verification on.
- [ ] Every `_Function` mapped using section 2; anything without a mapping raised as a question, not improvised.
- [ ] Server credentials declared in `getVariables()`, secrets marked `encryptable`.
- [ ] `configoptionN` replaced by named `package_vars_values` fields, or by `packageName` if the option is a remote plan name.
- [ ] Configurable options mapped to `package_addons`.
- [ ] Every `return 'success'` / `return $error` replaced by the `doX()` / `x()` pattern with `CE_Exception`.
- [ ] No `tbl*` tables, `localAPI()`, or WHMCS globals left.
- [ ] Smarty templates rewritten as `.phtml` with correct escaping.
- [ ] `getAvailableActions()` written (WHMCS has no equivalent, so it's easy to forget).
- [ ] Everything in the `create.md` release checklist.
