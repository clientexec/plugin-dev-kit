# Creating a Clientexec Registrar Plugin

A registrar plugin connects Clientexec to a domain registrar. Clientexec calls it to check availability during domain search, to register, renew, and transfer domains, to show and edit a domain's nameservers, DNS records, contacts, and lock in the client area, and to import existing domains and TLD prices from the registrar.

Load `reference/common/conventions.md` first. This file assumes it.

**Examples to copy from:** `namesilo` (complete, compact, current conventions), `enom` (paged domain import), `openprovider` and `realtimeregister` (REST APIs). The `resellerclub` / `resellbiz` / `netearthone` family share one API client and show how to factor a shared client out of the plugin class.

---

## 1. How core reaches a registrar

There are three paths, and they pass settings differently from servers and gateways.

| Path | Used for | How the plugin is called |
| --- | --- | --- |
| **Actions** | Register, Renew, Transfer, Cancel, custom actions | `RegistrarPlugin::doAction()` calls `do<Action>($params)` |
| **Direct calls through `DomainNameGateway::callPlugin()`** | Nameservers, DNS, contacts, lock, general info, EPP code, transfer status, auto renew, privacy | `$plugin->method($params)` |
| **Calls from search and import** | `checkDomain()`, `fetchDomains()`, `getTLDsAndPrices()` | `$plugin->method($params)` |

On every path, each of your settings is added to `$params` **under its bare setting name**:

```php
$params['API Key']             // not plugin_acmereg_API Key, not plugin_acmereg_API_Key
$params['Use testing server']
```

`$this->getVariable('API Key')` also works anywhere in the plugin.

Through `callPlugin()`, `$params` also always contains `tld`, `sld`, and `userPackageId` for the domain.

### What `callPlugin()` does with exceptions

| Plugin throws | Result |
| --- | --- |
| `CE_Exception` | Rethrown. On client-facing pages, a leading `"<Name> API Error:"` / `"<Name> Plugin Error:"` / `"<Name> Error:"` prefix is replaced with `"Error:"`, then the message is shown. |
| `MethodNotImplemented` | Rethrown. Callers treat it as "this registrar doesn't support that feature" and hide or skip the feature. |
| any other `Exception` | Logged, and `callPlugin()` returns `false`. The client sees the feature as unavailable without a message. |

So: throw `CE_Exception` with a message a client can read for real failures, throw `MethodNotImplemented` for features the registrar doesn't offer, and don't let other exception types escape.

`MethodNotImplemented` lives in `modules/domains/models/MethodNotImplemented.php`, which `RegistrarPlugin.php` already loads.

---

## 2. Files

```
plugins/registrars/acmereg/
├── PluginAcmereg.php     # the plugin (required)
├── AcmeregApi.php        # API client (recommended)
├── index.html
└── logo.png
```

---

## 3. Settings

Settings Clientexec reads by name:

| Setting | Type | Meaning |
| --- | --- | --- |
| `Plugin Name` | hidden | Display name. |
| `Actions` | hidden | Actions available while the domain **isn't registered yet**. Usually `'Register'`. |
| `Registered Actions` | hidden | Actions available to admins once the domain **is registered**. |
| `Registered Actions For Customer` | hidden | Subset of registered actions clients may run. |
| `Supported Features` | label | Optional HTML list of what the plugin supports, shown in the plugin settings. |

Action lists are comma separated. Each entry is either a bare command (`Register`) or a command followed by its button label in parentheses (`Renew (Renew Domain)`). The command maps to a `do<Command>()` method.

```php
'Registered Actions' => [
    'type' => 'hidden',
    'description' => lang('Current actions that are active for this plugin (when a domain is registered)'),
    'value' => 'Renew (Renew Domain),DomainTransferWithPopup (Initiate Transfer),SendTransferKey (Send Auth Info),Cancel'
],
```

`Cancel` is handled by the base class (`doCancel()`), so you can list it without implementing anything.

Everything else is your own: credentials, test mode, defaults for privacy and auto renew.

As with every non-server plugin type, `encryptable` has no effect on registrar settings (conventions, section 3).

---

## 4. `$features`

```php
public $features = [
    'nameSuggest' => true,
    'importDomains' => true,
    'importPrices' => true
];
```

| Feature | Meaning |
| --- | --- |
| `nameSuggest` | `checkDomain()` can check a list of TLDs in one call (section 6). |
| `importDomains` | The plugin implements `fetchDomains()`, so admins can import the domains held at the registrar. |
| `importPrices` | The plugin implements `getTLDsAndPrices()`, so admins can import TLDs and the registrar's prices as domain products. |

---

## 5. Package custom fields a registrar reads and writes

A domain is a user package. These custom fields hold its registrar state:

| Field | Meaning |
| --- | --- |
| `Domain Name` | Full domain. `callPlugin()` splits it into `sld` and `tld` for you. |
| `Registrar` | Directory name of the registrar plugin handling the domain. |
| `Registrar Order Id` | Your reference for the registration. Current plugins set it to `<Registrar>-<userPackageId>` after registering, renewing, or transferring. When `getGeneralInfo()` returns `id = -1`, core shows this instead. |
| `Registration Option` | `1` for a transfer. |
| `Transfer Status` | Free text shown while a transfer is in progress. Set it to exactly `Completed` when the transfer finishes; core only shows expiration, lock, and similar details for a transfer once it reads `Completed`. |
| `Auto Renew` | Clientexec's copy of the auto renew flag. `namesilo` refreshes it from the registrar in `getGeneralInfo()`. |
| `Domain Extra Attr` | Serialized array of TLD-specific extra attributes from the order form, plus `eppCode`, `domainUsername`, and `domainPassword` where used. `buildRegisterParams()` and `buildTransferParams()` hand it to you as `ExtendedAttributes`. |

---

## 6. Method reference

### `checkDomain($params)`

Called during domain search. `$params` has your settings, plus:

| Key | Value |
| --- | --- |
| `sld` | the name searched for (without the TLD) |
| `tld` | the TLD searched for |
| `namesuggest` | list of TLDs to check as well, when name suggest is enabled for the product |
| `allAvailableTLDs` | every TLD configured in the product group |
| `enableNamespinner` | `true` if the product enables name spinning (suggesting alternate names) |

Return the results under a `result` key:

```php
return [
    'result' => [
        ['domain' => 'example', 'tld' => 'com', 'status' => 0],
        ['domain' => 'example', 'tld' => 'net', 'status' => 1],
    ]
];
```

| `status` | Meaning |
| --- | --- |
| `0` | available to register |
| `1` | already registered (this is what a transfer search looks for) |
| `2` | TLD not supported by the registrar |
| `3` | invalid domain |
| `5` | couldn't reach the registry |

Report premium domains you won't sell at the normal price as `1`, as `namesilo` does.

If the response has no `result` key, the search shows "There was an error contacting the registrar, please try again!" Exceptions are not caught by the search, so catch API failures and either return without `result` or throw `CE_Exception` with a clear message.

With `nameSuggest` enabled and `$params['namesuggest']` non-empty, check `sld` against every TLD in the list (batch the requests if your API limits how many names it accepts per call). Without it, check only `sld.tld`.

`checkDomain()` is also used to decide whether a domain is registered before an automatic renewal, so it must work outside a search.

### `doRegister($params)` and `registerDomain($params)`

`doRegister()` is the `Register` action. Build the full parameter set with `buildRegisterParams()`, register, store your order reference, and return a message:

```php
public function doRegister($params): string
{
    $userPackage = new UserPackage($params['userPackageId']);
    $this->registerDomain($this->buildRegisterParams($userPackage, $params));
    $userPackage->setCustomField('Registrar Order Id', $userPackage->getCustomField('Registrar') . '-' . $params['userPackageId']);
    return $userPackage->getCustomField('Domain Name') . ' has been registered.';
}
```

`buildRegisterParams()` adds:

| Key | Value |
| --- | --- |
| `tld`, `sld` | the domain |
| `NumYears` | registration period in years, from the package's billing cycle. **`0` when the billing cycle isn't measured in years**, so guard against it. |
| `RegistrantFirstName`, `RegistrantLastName`, `RegistrantOrganizationName`, `RegistrantEmailAddress`, `RegistrantPhone` | from the client's profile |
| `RegistrantAddress1`, `RegistrantCity`, `RegistrantPostalCode`, `RegistrantCountry` | from the client's profile. There is no `RegistrantAddress2`. |
| `RegistrantStateProvince` | state or province **name** |
| `RegistrantStateProvinceCode` | state or province **code** |
| `NS1` ... `NS11` | `['hostname' => ...]` for each nameserver of the server hosting the domain's site, when there is one |
| `usedns` | `'default'` when there are no hosting nameservers to use; use the registrar's defaults |
| `ExtendedAttributes` | the unserialized `Domain Extra Attr` (TLD extra attributes), or `false` |
| `DomainUsername`, `DomainPassword` | a generated per-domain credential, for registrars that need one |
| `renewname` | the domain's `Auto Renew` custom field |
| `package_addons` | addon values, keyed by addon variable name, when the domain has addons. `IDPROTECT` is ID protection (WHOIS privacy). |

`RegistrantPhone` is whatever the client typed. Most registrars want E.164-style `+1.5555555555`; normalize it, as most shipped plugins do in a `validatePhone()` helper.

Throw `CE_Exception` if registration fails. Don't return error codes: the base class docblock's `array(code)` return describes an older contract.

### `doRenew($params)` and `renewDomain($params)`

The `Renew` action, and also what core runs automatically after a renewal invoice is paid (when "Auto-Renew Domain" is on, the domain is inside the renewal window, and the registrar says the domain can be renewed). Use `buildRenewParams()`, which adds `tld`, `sld`, `NumYears`, and `package_addons`.

### `canAutoRenewDomain($params)` (optional)

If implemented, core calls it before an automatic renewal and only renews when it returns exactly `true`. Return `false` for statuses that need a manual renewal (redemption, pending delete, transferred away).

It's called through `callPlugin()`, so `$params` has your settings plus `tld`, `sld`, and `userPackageId`, like the other direct calls. It doesn't get `NumYears` or registrant data.

When it doesn't return `true`, the renewal is skipped and core opens a "manual renewal required" ticket. That includes throwing anything: a `CE_Exception`'s message is included in the ticket, and other exceptions are logged and treated as "not renewable". Without the method, core only blocks renewal of domains that have already expired.

### `doDomainTransferWithPopup($params)`, `initiateTransfer($params)`, `getTransferStatus($params)`

`DomainTransferWithPopup` is the transfer action. Use `buildTransferParams()`, which adds the registrant fields, `ExtendedAttributes`, `DomainUsername`, `DomainPassword`, and `eppCode` (from the action input, or the `eppCode` stored in `Domain Extra Attr`). After starting the transfer, set `Registrar Order Id` and set `Transfer Status` to something meaningful.

`getTransferStatus()` returns a string shown to the admin and client while the transfer is pending. When the registrar reports completion, set the `Transfer Status` field to `Completed`. Throw `MethodNotImplemented` if the registrar can't report transfer status.

### `getGeneralInfo($params)`

Returns the domain's state at the registrar:

| Key | Value |
| --- | --- |
| `id` | registrar's ID for the domain, or `-1` to show `Registrar Order Id` instead |
| `domain` | full domain name |
| `expiration` | expiration date in a format `strtotime()` understands. Core computes days to expiry from it, so this must be accurate. |
| `registrationstatus` | registrar's status text |
| `purchasestatus` | registrar's purchase status text, or `'N/A'` |
| `autorenew` | `1` or `0` |
| `is_registered` | `true` if the domain is registered (used by expiration reminders) |
| `is_expired` | `true` if the domain has expired (used by expiration reminders) |

Return every key. If the call fails, throw `CE_Exception`; core falls back to showing "Unknown" values.

### `getNameServers($params)` and `setNameServers($params)`

`getNameServers()` returns the nameservers as **integer keys**, plus two flags:

```php
return [
    'usesDefault' => false,   // true if the domain uses the registrar's default nameservers
    'hasDefault' => true,     // true if the registrar has default nameservers the client can switch to
    0 => 'ns1.example.net',
    1 => 'ns2.example.net'
];
```

When `hasDefault` is true, the client sees a "Use Defaults" toggle, and the nameserver fields are hidden while `usesDefault` is true.

`setNameServers()` receives:

| Key | Value |
| --- | --- |
| `ns` | the nameservers, keyed from `1` (`[1 => 'ns1...', 2 => 'ns2...']`), blanks removed |
| `default` | `true` if the client chose the registrar's defaults |

### `getDNS($params)` and `setDNS($params)`

`getDNS()` returns:

```php
return [
    'records' => [
        ['id' => '123', 'hostname' => 'www', 'address' => '203.0.113.10', 'type' => 'A'],
    ],
    'types' => ['A', 'AAAA', 'CNAME', 'MX', 'TXT'],
    'default' => true
];
```

`default` must be `true` for records to be shown at all. It means the domain uses the registrar's nameservers, so the registrar's zone is the live one. When it's anything else, the client is told host records need the default nameservers.

`setDNS()` receives `['records' => [...]]`, each record with:

| Key | Value |
| --- | --- |
| `id` | your record ID for existing records; a client-side ID for new ones |
| `hostname`, `address`, `type` | as edited |
| `new` | `true` for records the client added, `false` for existing ones |

The list is the complete desired zone: records the client deleted are simply absent. Core rejects blank hostnames and addresses before calling you, but validate `type` against what the registrar accepts, and validate `address` for the record type.

A `CE_Exception` thrown from `getDNS()` is shown to the client in place of the records.

### `getContactInformation($params)` and `setContactInformation($params)`

`getContactInformation()` returns contacts by type. Each field is `[label, value]`:

```php
return [
    'Registrant' => [
        'FirstName' => [$this->user->lang('First Name'), 'Jane'],
        'LastName' => [$this->user->lang('Last Name'), 'Doe'],
        'EmailAddress' => [$this->user->lang('E-mail'), 'jane@example.com'],
        // ...
    ]
];
```

The client area shows and edits the `Registrant` type only. Domain import reads `Registrant`, falling back to `Billing` when the registrant has no email address.

Use these field keys, which most shipped plugins use: `OrganizationName`, `FirstName`, `LastName`, `Address1`, `Address2`, `City`, `StateProvince`, `PostalCode`, `Country`, `EmailAddress`, `Phone`, `Fax`.

`setContactInformation()` receives each edited field as `Registrant_<Key>` (for example `$params['Registrant_FirstName']`), plus `type => 'Registrant'`, `tld`, `sld`, and `userPackageId`. The keys are whatever your `getContactInformation()` returned, so the two methods must use the same keys.

### `getRegistrarLock($params)`, `doSetRegistrarLock($params)`, `setRegistrarLock($params)`

`getRegistrarLock()` returns `1` if locked, `0` if not.

The client's lock toggle runs the `SetRegistrarLock` action. Use `buildLockParams()`, which puts the requested state in `$params['lock']`:

```php
public function doSetRegistrarLock($params): string
{
    $userPackage = new UserPackage($params['userPackageId']);
    $this->setRegistrarLock($this->buildLockParams($userPackage, $params));
    return 'Updated Registrar Lock.';
}
```

### `setAutorenew($params)`

`$params['autorenew']` is the requested state. Throw `MethodNotImplemented` if the registrar has no auto renew setting.

### `doSendTransferKey($params)` and `sendTransferKey($params)`

The `SendTransferKey` action: ask the registrar to email the auth code to the registrant.

### `getEPPCode($params)`

Returns the auth code to show in the client area and admin. The base class returns `'N/A'`, which hides the field. Override it only if the registrar exposes the code through its API.

### `disablePrivateRegistration($params)`

Called when a domain with the `IDPROTECT` addon is cancelled. Throw `MethodNotImplemented` if the registrar doesn't support privacy.

### `fetchDomains($params)` (with `importDomains`)

Returns `[$domains, $meta]`:

```php
return [
    [
        ['id' => '98765', 'sld' => 'example', 'tld' => 'com', 'exp' => '2027-03-01'],
    ],
    [
        'total' => 230,
        'start' => 1,
        'end' => 100,
        'next' => 101,
        'numPerPage' => 100
    ]
];
```

If `total` is greater than `end`, core calls again with `$params['next']` set to your `next` value until it has every domain. If the registrar returns everything in one call, set `total` and `end` to the count so core doesn't page. During import, core calls `getGeneralInfo()` and `getContactInformation()` for each selected domain.

### `getTLDsAndPrices($params)` (with `importPrices`)

Returns prices keyed by TLD, without the leading dot:

```php
return [
    'com' => ['pricing' => ['register' => '9.99', 'transfer' => '9.99', 'renew' => '11.99']],
    'net' => ['pricing' => ['register' => '11.49', 'transfer' => '11.49', 'renew' => '13.99']],
];
```

These are the registrar's costs, imported as a starting point for the admin's own prices.

### Methods you don't need

`checkNSStatus`, `registerNS`, `editNS`, and `deleteNS` appear as empty stubs in many shipped plugins. They aren't in the base class and nothing in core calls them. Leave them out.

---

## 7. Complete example

A fictional registrar, Acmereg, with a JSON REST API. Every core-called method is included.

### `plugins/registrars/acmereg/PluginAcmereg.php`

```php
<?php

require_once 'modules/admin/models/RegistrarPlugin.php';
require_once __DIR__ . '/AcmeregApi.php';

class PluginAcmereg extends RegistrarPlugin
{
    public $features = [
        'nameSuggest' => true,
        'importDomains' => true,
        'importPrices' => true
    ];

    public function getVariables()
    {
        $variables = [
            'Plugin Name' => [
                'type' => 'hidden',
                'description' => lang('How CE sees this plugin (not to be confused with the Signup Name)'),
                'value' => 'Acmereg'
            ],
            'API Key' => [
                'type' => 'password',
                'description' => lang('API key from your Acmereg reseller dashboard'),
                'value' => ''
            ],
            'Use testing server' => [
                'type' => 'yesno',
                'description' => lang('Select Yes to send requests to the Acmereg sandbox.'),
                'value' => '0'
            ],
            'Default Privacy?' => [
                'type' => 'yesno',
                'description' => lang('Select Yes to enable WHOIS privacy by default for new registrations and transfers.'),
                'value' => '0'
            ],
            'Actions' => [
                'type' => 'hidden',
                'description' => lang('Current actions that are active for this plugin (when a domain isn\'t registered)'),
                'value' => 'Register'
            ],
            'Registered Actions' => [
                'type' => 'hidden',
                'description' => lang('Current actions that are active for this plugin (when a domain is registered)'),
                'value' => 'Renew (Renew Domain),DomainTransferWithPopup (Initiate Transfer),SendTransferKey (Send Auth Info),Cancel'
            ],
            'Registered Actions For Customer' => [
                'type' => 'hidden',
                'description' => lang('Current actions that are active for this plugin (when a domain is registered)'),
                'value' => 'SendTransferKey (Send Auth Info)'
            ]
        ];

        return $variables;
    }

    public function checkDomain($params)
    {
        $sld = strtolower(trim($params['sld']));
        $tlds = [strtolower($params['tld'])];
        if (!empty($params['namesuggest']) && is_array($params['namesuggest'])) {
            $tlds = array_values(array_unique(array_merge($tlds, array_map('strtolower', $params['namesuggest']))));
        }

        $results = [];
        foreach (array_chunk($tlds, 50) as $chunk) {
            $names = [];
            foreach ($chunk as $tld) {
                $names[] = $sld . '.' . $tld;
            }

            try {
                $response = $this->api($params)->post('/domains/check', ['domains' => $names]);
            } catch (Exception $e) {
                CE_Lib::log(1, 'plugin_acmereg::checkDomain::error: ' . $e->getMessage());
                return [];
            }

            foreach ($response['domains'] as $domain) {
                $split = DomainNameGateway::splitDomain($domain['name']);
                $status = 1;
                if ($domain['available'] === true && empty($domain['premium'])) {
                    $status = 0;
                } elseif (($domain['reason'] ?? '') === 'unsupported_tld') {
                    $status = 2;
                }

                $results[] = [
                    'domain' => $split[0],
                    'tld' => $split[1],
                    'status' => $status
                ];
            }
        }

        return ['result' => $results];
    }

    public function doRegister($params): string
    {
        $userPackage = new UserPackage($params['userPackageId']);
        $this->registerDomain($this->buildRegisterParams($userPackage, $params));
        $userPackage->setCustomField('Registrar Order Id', $userPackage->getCustomField('Registrar') . '-' . $params['userPackageId']);
        return $userPackage->getCustomField('Domain Name') . ' has been registered.';
    }

    public function registerDomain($params): void
    {
        if ((int) $params['NumYears'] < 1) {
            throw new CE_Exception('Acmereg error: the domain billing cycle must be one or more years.');
        }

        $payload = [
            'domain' => strtolower($params['sld'] . '.' . $params['tld']),
            'years' => (int) $params['NumYears'],
            'registrant' => $this->registrantFromParams($params),
            'privacy' => $this->wantsPrivacy($params),
            'auto_renew' => !empty($params['renewname']),
            'extra' => is_array($params['ExtendedAttributes']) ? $params['ExtendedAttributes'] : []
        ];

        $nameservers = $this->nameserversFromParams($params);
        if (count($nameservers) > 0) {
            $payload['nameservers'] = $nameservers;
        }

        try {
            $this->api($params)->post('/domains', $payload);
        } catch (Exception $e) {
            throw new CE_Exception('Acmereg error: ' . $e->getMessage());
        }
    }

    public function doRenew($params): string
    {
        $userPackage = new UserPackage($params['userPackageId']);
        $this->renewDomain($this->buildRenewParams($userPackage, $params));
        $userPackage->setCustomField('Registrar Order Id', $userPackage->getCustomField('Registrar') . '-' . $params['userPackageId']);
        return $userPackage->getCustomField('Domain Name') . ' has been renewed.';
    }

    public function renewDomain($params): void
    {
        if ((int) $params['NumYears'] < 1) {
            throw new CE_Exception('Acmereg error: the domain billing cycle must be one or more years.');
        }

        try {
            $this->api($params)->post('/domains/' . rawurlencode($this->domainName($params)) . '/renew', [
                'years' => (int) $params['NumYears']
            ]);
        } catch (Exception $e) {
            throw new CE_Exception('Acmereg error: ' . $e->getMessage());
        }
    }

    public function canAutoRenewDomain($params): bool
    {
        $domain = $this->api($params)->get('/domains/' . rawurlencode($this->domainName($params)));

        return in_array($domain['status'] ?? '', ['active', 'grace'], true);
    }

    public function doDomainTransferWithPopup($params): string
    {
        $userPackage = new UserPackage($params['userPackageId']);
        $this->initiateTransfer($this->buildTransferParams($userPackage, $params));
        $userPackage->setCustomField('Registrar Order Id', $userPackage->getCustomField('Registrar') . '-' . $params['userPackageId']);
        $userPackage->setCustomField('Transfer Status', 'Transfer requested');
        return 'Transfer of ' . $userPackage->getCustomField('Domain Name') . ' has been initiated.';
    }

    public function initiateTransfer($params): void
    {
        if (trim((string) $params['eppCode']) === '') {
            throw new CE_Exception('Acmereg error: an EPP code is required to transfer this domain.');
        }

        try {
            $this->api($params)->post('/transfers', [
                'domain' => $this->domainName($params),
                'auth_code' => $params['eppCode'],
                'registrant' => $this->registrantFromParams($params),
                'privacy' => $this->wantsPrivacy($params)
            ]);
        } catch (Exception $e) {
            throw new CE_Exception('Acmereg error: ' . $e->getMessage());
        }
    }

    public function getTransferStatus($params)
    {
        try {
            $transfer = $this->api($params)->get('/transfers/' . rawurlencode($this->domainName($params)));
        } catch (Exception $e) {
            throw new CE_Exception('Acmereg error: ' . $e->getMessage());
        }

        if (($transfer['status'] ?? '') === 'completed') {
            $userPackage = new UserPackage($params['userPackageId']);
            $userPackage->setCustomField('Transfer Status', 'Completed');
            return 'Completed';
        }

        return (string) ($transfer['status_message'] ?? $transfer['status'] ?? 'Pending');
    }

    public function getGeneralInfo($params)
    {
        try {
            $domain = $this->api($params)->get('/domains/' . rawurlencode($this->domainName($params)));
        } catch (Exception $e) {
            throw new CE_Exception('Acmereg error: ' . $e->getMessage());
        }

        $autoRenew = !empty($domain['auto_renew']) ? 1 : 0;
        $userPackage = new UserPackage($params['userPackageId']);
        $userPackage->setCustomField('Auto Renew', $autoRenew);

        return [
            'id' => (string) $domain['id'],
            'domain' => $this->domainName($params),
            'expiration' => (string) $domain['expires_at'],
            'registrationstatus' => (string) $domain['status'],
            'purchasestatus' => 'N/A',
            'autorenew' => $autoRenew,
            'is_registered' => true,
            'is_expired' => strtotime((string) $domain['expires_at']) < time()
        ];
    }

    public function getNameServers($params)
    {
        try {
            $domain = $this->api($params)->get('/domains/' . rawurlencode($this->domainName($params)));
        } catch (Exception $e) {
            throw new CE_Exception('Acmereg error: ' . $e->getMessage());
        }

        $info = [
            'usesDefault' => !empty($domain['uses_default_nameservers']),
            'hasDefault' => true
        ];
        foreach ($domain['nameservers'] as $nameserver) {
            $info[] = (string) $nameserver;
        }

        return $info;
    }

    public function setNameServers($params): void
    {
        $payload = ['use_default' => (bool) $params['default']];
        if (!$params['default']) {
            $payload['nameservers'] = array_values($params['ns']);
        }

        try {
            $this->api($params)->put('/domains/' . rawurlencode($this->domainName($params)) . '/nameservers', $payload);
        } catch (Exception $e) {
            throw new CE_Exception('Acmereg error: ' . $e->getMessage());
        }
    }

    public function getDNS($params)
    {
        try {
            $zone = $this->api($params)->get('/domains/' . rawurlencode($this->domainName($params)) . '/records');
        } catch (Exception $e) {
            throw new CE_Exception('Acmereg error: ' . $e->getMessage());
        }

        $records = [];
        foreach ($zone['records'] as $record) {
            $records[] = [
                'id' => (string) $record['id'],
                'hostname' => (string) $record['name'],
                'address' => (string) $record['content'],
                'type' => (string) $record['type']
            ];
        }

        return [
            'records' => $records,
            'types' => ['A', 'AAAA', 'CNAME', 'MX', 'TXT'],
            'default' => !empty($zone['uses_default_nameservers'])
        ];
    }

    public function setDNS($params): void
    {
        $allowedTypes = ['A', 'AAAA', 'CNAME', 'MX', 'TXT'];
        $records = [];
        foreach ($params['records'] as $record) {
            $type = strtoupper(trim($record['type']));
            if (!in_array($type, $allowedTypes, true)) {
                throw new CE_Exception($this->user->lang('Record type %s is not supported.', $type));
            }

            $records[] = [
                'name' => trim($record['hostname']),
                'type' => $type,
                'content' => trim($record['address'])
            ];
        }

        try {
            $this->api($params)->put('/domains/' . rawurlencode($this->domainName($params)) . '/records', ['records' => $records]);
        } catch (Exception $e) {
            throw new CE_Exception('Acmereg error: ' . $e->getMessage());
        }
    }

    public function getContactInformation($params)
    {
        try {
            $contact = $this->api($params)->get('/domains/' . rawurlencode($this->domainName($params)) . '/contacts/registrant');
        } catch (Exception $e) {
            throw new CE_Exception('Acmereg error: ' . $e->getMessage());
        }

        return [
            'Registrant' => [
                'OrganizationName' => [$this->user->lang('Organization'), (string) $contact['organization']],
                'FirstName' => [$this->user->lang('First Name'), (string) $contact['first_name']],
                'LastName' => [$this->user->lang('Last Name'), (string) $contact['last_name']],
                'Address1' => [$this->user->lang('Address') . ' 1', (string) $contact['address1']],
                'Address2' => [$this->user->lang('Address') . ' 2', (string) $contact['address2']],
                'City' => [$this->user->lang('City'), (string) $contact['city']],
                'StateProvince' => [$this->user->lang('Province') . '/' . $this->user->lang('State'), (string) $contact['state']],
                'PostalCode' => [$this->user->lang('Postal Code'), (string) $contact['postal_code']],
                'Country' => [$this->user->lang('Country'), (string) $contact['country']],
                'EmailAddress' => [$this->user->lang('E-mail'), (string) $contact['email']],
                'Phone' => [$this->user->lang('Phone'), (string) $contact['phone']],
                'Fax' => [$this->user->lang('Fax'), (string) $contact['fax']]
            ]
        ];
    }

    public function setContactInformation($params): void
    {
        $contact = [
            'organization' => $params['Registrant_OrganizationName'] ?? '',
            'first_name' => $params['Registrant_FirstName'] ?? '',
            'last_name' => $params['Registrant_LastName'] ?? '',
            'address1' => $params['Registrant_Address1'] ?? '',
            'address2' => $params['Registrant_Address2'] ?? '',
            'city' => $params['Registrant_City'] ?? '',
            'state' => $params['Registrant_StateProvince'] ?? '',
            'postal_code' => $params['Registrant_PostalCode'] ?? '',
            'country' => $params['Registrant_Country'] ?? '',
            'email' => $params['Registrant_EmailAddress'] ?? '',
            'phone' => $this->formatPhone($params['Registrant_Phone'] ?? '', $params['Registrant_Country'] ?? ''),
            'fax' => $params['Registrant_Fax'] ?? ''
        ];

        if (filter_var($contact['email'], FILTER_VALIDATE_EMAIL) === false) {
            throw new CE_Exception($this->user->lang('Please enter a valid e-mail address.'));
        }

        try {
            $this->api($params)->put('/domains/' . rawurlencode($this->domainName($params)) . '/contacts/registrant', $contact);
        } catch (Exception $e) {
            throw new CE_Exception('Acmereg error: ' . $e->getMessage());
        }
    }

    public function getRegistrarLock($params)
    {
        try {
            $domain = $this->api($params)->get('/domains/' . rawurlencode($this->domainName($params)));
        } catch (Exception $e) {
            throw new CE_Exception('Acmereg error: ' . $e->getMessage());
        }

        return !empty($domain['locked']) ? 1 : 0;
    }

    public function doSetRegistrarLock($params): string
    {
        $userPackage = new UserPackage($params['userPackageId']);
        $this->setRegistrarLock($this->buildLockParams($userPackage, $params));
        return 'Updated Registrar Lock.';
    }

    public function setRegistrarLock($params): void
    {
        try {
            $this->api($params)->put('/domains/' . rawurlencode($this->domainName($params)) . '/lock', [
                'locked' => $params['lock'] == 1
            ]);
        } catch (Exception $e) {
            throw new CE_Exception('Acmereg error: ' . $e->getMessage());
        }
    }

    public function setAutorenew($params): void
    {
        try {
            $this->api($params)->put('/domains/' . rawurlencode($this->domainName($params)) . '/auto-renew', [
                'enabled' => (bool) $params['autorenew']
            ]);
        } catch (Exception $e) {
            throw new CE_Exception('Acmereg error: ' . $e->getMessage());
        }
    }

    public function doSendTransferKey($params): string
    {
        $userPackage = new UserPackage($params['userPackageId']);
        $this->sendTransferKey($this->buildRegisterParams($userPackage, $params));
        return 'Successfully sent auth info for ' . $userPackage->getCustomField('Domain Name');
    }

    public function sendTransferKey($params): void
    {
        try {
            $this->api($params)->post('/domains/' . rawurlencode($this->domainName($params)) . '/auth-code/send', []);
        } catch (Exception $e) {
            throw new CE_Exception('Acmereg error: ' . $e->getMessage());
        }
    }

    public function getEPPCode($params)
    {
        try {
            $response = $this->api($params)->get('/domains/' . rawurlencode($this->domainName($params)) . '/auth-code');
        } catch (Exception $e) {
            CE_Lib::log(4, 'plugin_acmereg::getEPPCode::error: ' . $e->getMessage());
            return 'N/A';
        }

        return (string) ($response['auth_code'] ?? 'N/A');
    }

    public function disablePrivateRegistration($params): void
    {
        try {
            $this->api($params)->put('/domains/' . rawurlencode($this->domainName($params)) . '/privacy', ['enabled' => false]);
        } catch (Exception $e) {
            throw new CE_Exception('Acmereg error: ' . $e->getMessage());
        }
    }

    public function fetchDomains($params)
    {
        $perPage = 100;
        $offset = isset($params['next']) ? max(0, (int) $params['next'] - 1) : 0;

        try {
            $response = $this->api($params)->get('/domains?limit=' . $perPage . '&offset=' . $offset);
        } catch (Exception $e) {
            throw new CE_Exception('Acmereg error: ' . $e->getMessage());
        }

        $domains = [];
        foreach ($response['domains'] as $domain) {
            $split = DomainNameGateway::splitDomain($domain['name']);
            $domains[] = [
                'id' => (string) $domain['id'],
                'sld' => $split[0],
                'tld' => $split[1],
                'exp' => (string) $domain['expires_at']
            ];
        }

        $total = (int) $response['total'];
        $end = $offset + count($domains);

        return [
            $domains,
            [
                'total' => $total,
                'start' => $offset + 1,
                'end' => $end,
                'next' => $end + 1,
                'numPerPage' => $perPage
            ]
        ];
    }

    public function getTLDsAndPrices($params)
    {
        try {
            $response = $this->api($params)->get('/pricing');
        } catch (Exception $e) {
            throw new CE_Exception('Acmereg error: ' . $e->getMessage());
        }

        $tlds = [];
        foreach ($response['tlds'] as $tld => $price) {
            $tlds[ltrim((string) $tld, '.')] = [
                'pricing' => [
                    'register' => $price['register'],
                    'transfer' => $price['transfer'],
                    'renew' => $price['renew']
                ]
            ];
        }

        return $tlds;
    }

    private function api($params): AcmeregApi
    {
        $apiKey = $params['API Key'] ?? $this->getVariable('API Key');
        $testMode = $params['Use testing server'] ?? $this->getVariable('Use testing server');

        return new AcmeregApi((string) $apiKey, $testMode == '1');
    }

    private function domainName($params): string
    {
        return strtolower($params['sld'] . '.' . $params['tld']);
    }

    private function wantsPrivacy($params): bool
    {
        if (isset($params['package_addons']['IDPROTECT']) && $params['package_addons']['IDPROTECT'] == 1) {
            return true;
        }

        return $this->getVariable('Default Privacy?') == '1';
    }

    private function registrantFromParams($params): array
    {
        return [
            'organization' => $params['RegistrantOrganizationName'],
            'first_name' => $params['RegistrantFirstName'],
            'last_name' => $params['RegistrantLastName'],
            'address1' => $params['RegistrantAddress1'],
            'city' => $params['RegistrantCity'],
            'state' => $params['RegistrantStateProvinceCode'] != '' ? $params['RegistrantStateProvinceCode'] : $params['RegistrantStateProvince'],
            'postal_code' => $params['RegistrantPostalCode'],
            'country' => $params['RegistrantCountry'],
            'email' => $params['RegistrantEmailAddress'],
            'phone' => $this->formatPhone($params['RegistrantPhone'], $params['RegistrantCountry'])
        ];
    }

    private function nameserversFromParams($params): array
    {
        if (($params['usedns'] ?? '') === 'default') {
            return [];
        }

        $nameservers = [];
        for ($i = 1; $i <= 11; $i++) {
            if (!isset($params['NS' . $i]['hostname'])) {
                break;
            }
            $nameservers[] = $params['NS' . $i]['hostname'];
        }

        return $nameservers;
    }

    private function formatPhone(string $phone, string $country): string
    {
        $phone = trim($phone);
        if (preg_match('/^\+[0-9]{1,3}\.[0-9]{4,14}$/', $phone) === 1) {
            return $phone;
        }

        $digits = preg_replace('/[^0-9]/', '', $phone);
        if ($digits === '') {
            return '';
        }

        // Acmereg expects +<country code>.<number>. Without a country code in the input,
        // only NANP numbers can be completed safely; adjust this for your registrar.
        if (strlen($digits) === 10 && in_array(strtoupper($country), ['US', 'CA'], true)) {
            return '+1.' . $digits;
        }

        if (strlen($digits) === 11 && $digits[0] === '1' && in_array(strtoupper($country), ['US', 'CA'], true)) {
            return '+1.' . substr($digits, 1);
        }

        throw new CE_Exception($this->user->lang('Please enter the phone number in the format +CountryCode.Number, for example +44.2071234567.'));
    }
}
```

### `plugins/registrars/acmereg/AcmeregApi.php`

```php
<?php

class AcmeregApi
{
    private string $apiKey;
    private string $baseUrl;

    public function __construct(string $apiKey, bool $testMode)
    {
        $this->apiKey = trim($apiKey);
        $this->baseUrl = $testMode ? 'https://sandbox.acmereg.example/v1' : 'https://api.acmereg.example/v1';
    }

    public function get(string $path): array
    {
        return $this->request('GET', $path);
    }

    public function post(string $path, array $body): array
    {
        return $this->request('POST', $path, $body);
    }

    public function put(string $path, array $body): array
    {
        return $this->request('PUT', $path, $body);
    }

    private function request(string $method, string $path, array $body = []): array
    {
        if ($this->apiKey === '') {
            throw new CE_Exception('Acmereg API key is not configured.');
        }

        $ch = curl_init();
        curl_setopt($ch, CURLOPT_URL, $this->baseUrl . $path);
        curl_setopt($ch, CURLOPT_CUSTOMREQUEST, $method);
        curl_setopt($ch, CURLOPT_RETURNTRANSFER, true);
        curl_setopt($ch, CURLOPT_SSL_VERIFYPEER, true);
        curl_setopt($ch, CURLOPT_SSL_VERIFYHOST, 2);
        curl_setopt($ch, CURLOPT_CAINFO, \Composer\CaBundle\CaBundle::getSystemCaRootBundlePath());
        curl_setopt($ch, CURLOPT_TIMEOUT, 60);
        curl_setopt($ch, CURLOPT_HTTPHEADER, [
            'Authorization: Bearer ' . $this->apiKey,
            'Accept: application/json',
            'Content-Type: application/json'
        ]);
        if ($method !== 'GET') {
            curl_setopt($ch, CURLOPT_POSTFIELDS, json_encode($body));
        }

        $response = curl_exec($ch);
        if ($response === false) {
            $error = curl_error($ch);
            curl_close($ch);
            throw new CE_Exception('Connection to Acmereg failed: ' . $error);
        }

        $status = curl_getinfo($ch, CURLINFO_HTTP_CODE);
        curl_close($ch);

        $decoded = json_decode($response, true);
        if (!is_array($decoded)) {
            throw new CE_Exception('Acmereg returned an unexpected response (HTTP ' . $status . ').');
        }

        if ($status < 200 || $status >= 300) {
            throw new CE_Exception($decoded['error'] ?? ('HTTP ' . $status));
        }

        return $decoded;
    }
}
```

Add `logo.png` and an empty `index.html`.

---

## 8. Checklist before release

- [ ] Settings read from `$params['<Setting Name>']` or `$this->getVariable()`, never `plugin_<dir>_...`.
- [ ] `checkDomain()` returns `['result' => [...]]` with `domain`, `tld`, `status`, and handles the `namesuggest` list.
- [ ] `registerDomain()`, `renewDomain()`, and `initiateTransfer()` throw `CE_Exception` on failure and refuse `NumYears` of `0`.
- [ ] `Registrar Order Id` set after register, renew, and transfer; `Transfer Status` set to exactly `Completed` when a transfer finishes.
- [ ] `getGeneralInfo()` returns every key, with an accurate `expiration`.
- [ ] `getNameServers()` uses integer keys plus `usesDefault` and `hasDefault`.
- [ ] `getDNS()` returns `default => true` only when the registrar's zone is live; `setDNS()` validates record types and treats the list as the complete zone.
- [ ] `getContactInformation()` and `setContactInformation()` use the same field keys.
- [ ] Unsupported features throw `MethodNotImplemented`, not a generic exception.
- [ ] Phone numbers normalized to the registrar's format.
- [ ] `fetchDomains()` returns usable paging metadata, even for single-page results.
- [ ] TLS verification on, using the Composer CA bundle.
- [ ] No empty `registerNS` / `editNS` / `deleteNS` / `checkNSStatus` stubs.

---

## 9. Modernizing an older registrar

1. **Errors:** replace returned `CE_Error` objects and `array(-1, 'message')` codes with thrown `CE_Exception`. Keep `MethodNotImplemented` for unsupported features.
2. **Missing keys:** make `getGeneralInfo()` return `id`, `domain`, `is_registered`, and `is_expired`, and make `fetchDomains()` return paging metadata. PHP 8 warns on every undefined array key core reads.
3. **TLS:** remove `CURLOPT_SSL_VERIFYPEER => false` and set the Composer CA bundle.
4. **Dead stubs:** delete empty `registerNS`, `editNS`, `deleteNS`, and `checkNSStatus`.
5. **Style:** conventions, section 11.

---

## Canonical reference files

| What | File |
| --- | --- |
| Base class, param builders, actions | `modules/admin/models/RegistrarPlugin.php` |
| Bridge from core to the plugin, return normalization | `modules/clients/models/DomainNameGateway.php` |
| Domain search, `checkDomain()` result handling | `modules/admin/models/TopLevelDomainGateway.php` (`search_domain()`) |
| Client area domain screens | `modules/clients/controllers/ProductspublicController.php` |
| Domain and price import | `modules/clients/controllers/IndexController.php` |
| Automatic renewal after payment | `modules/billing/models/BillingGateway.php` (`handleDomainRenewalAfterInvoiceIsPaid()`) |
| Import contract | `modules/domains/models/ICanImportDomains.php` |
| Compact current plugin | `plugins/registrars/namesilo/` |
