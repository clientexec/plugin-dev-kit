# Converting a WHMCS Registrar Module to a Clientexec Registrar Plugin

Load `reference/common/conventions.md` and `reference/registrars/create.md` first. This file only maps WHMCS concepts onto them.

## The short version

| WHMCS | Clientexec |
| --- | --- |
| `modules/registrars/acmereg/acmereg.php` (functions prefixed `acmereg_`) | `plugins/registrars/acmereg/PluginAcmereg.php` (`class PluginAcmereg extends RegistrarPlugin`) |
| `acmereg_getConfigArray()` | `getVariables()` |
| `acmereg_MetaData()` | `$features` and the `Actions` settings |
| Return `['error' => $msg]` | `throw new CE_Exception($msg)` |
| Return `['success' => true]` | Return normally |

The API client usually ports unchanged. Move it into its own file, then rebuild each function as a plugin method using the return shapes in `create.md`, section 6. Those shapes differ from WHMCS's in almost every method.

---

## 1. Function mapping

| WHMCS function | Clientexec method |
| --- | --- |
| `_getConfigArray()` | `getVariables()` |
| `_CheckAvailability($params)` | `checkDomain($params)`, returning `['result' => [...]]` |
| `_GetDomainSuggestions($params)` | `checkDomain()` with `nameSuggest` enabled; the TLDs to check arrive in `$params['namesuggest']` |
| `_RegisterDomain($params)` | `doRegister()` + `registerDomain()` |
| `_TransferDomain($params)` | `doDomainTransferWithPopup()` + `initiateTransfer()` |
| `_RenewDomain($params)` | `doRenew()` + `renewDomain()` |
| `_GetNameservers($params)` | `getNameServers()` |
| `_SaveNameservers($params)` | `setNameServers()` |
| `_GetRegistrarLock($params)` | `getRegistrarLock()`, returning `1` / `0` |
| `_SaveRegistrarLock($params)` | `doSetRegistrarLock()` + `setRegistrarLock()` |
| `_GetContactDetails($params)` | `getContactInformation()` |
| `_SaveContactDetails($params)` | `setContactInformation()` |
| `_GetEPPCode($params)` | `getEPPCode()` (display) and/or `doSendTransferKey()` + `sendTransferKey()` (email it to the registrant) |
| `_GetDNS($params)` | `getDNS()` |
| `_SaveDNS($params)` | `setDNS()` |
| `_IDProtectToggle($params)` | a custom action (for example `doTogglePrivacy()` listed as `togglePrivacy (Toggle Privacy)`), plus `disablePrivateRegistration()` |
| `_Sync($params)` | `getGeneralInfo()` (expiration and status) and `canAutoRenewDomain()` |
| `_TransferSync($params)` | `getTransferStatus()`, setting `Transfer Status` to `Completed` when done |
| `_GetTldPricing($params)` | `getTLDsAndPrices()` with `importPrices` enabled |
| domain import / bulk sync | `fetchDomains()` with `importDomains` enabled |
| `_RequestDelete($params)` | No core equivalent |
| `_RegisterNameserver`, `_ModifyNameserver`, `_DeleteNameserver` | No core equivalent |
| `_GetEmailForwarding`, `_SaveEmailForwarding` | No core equivalent |
| `_AdminCustomButtonArray()` | entries in `Registered Actions`, each with a `do<Command>()` method |
| `_ClientAreaCustomButtonArray()` | entries in `Registered Actions For Customer` |

If a WHMCS function has no row here, ask rather than inventing a Clientexec hook.

---

## 2. Settings

WHMCS settings are keyed by a code (`apiKey`). Clientexec keys them by the setting name itself, and for registrars passes them in `$params` under that **bare name**:

```php
// WHMCS
function acmereg_getConfigArray()
{
    return [
        'apiKey' => ['FriendlyName' => 'API Key', 'Type' => 'password'],
        'testMode' => ['FriendlyName' => 'Test Mode', 'Type' => 'yesno'],
    ];
}
// $params['apiKey'], $params['testMode']
```

```php
// Clientexec, inside getVariables()
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
// $params['API Key'], $params['Use testing server']
```

Add the `Plugin Name`, `Actions`, `Registered Actions`, and `Registered Actions For Customer` settings (`create.md`, section 3).

---

## 3. `$params` mapping

| WHMCS `$params` | Clientexec `$params` |
| --- | --- |
| `sld`, `tld` | `sld`, `tld` (no leading dot on `tld` in either) |
| `domainname` | `sld . '.' . tld` |
| `domainid` | `userPackageId` |
| `regperiod` | `NumYears` (can be `0`; see `create.md`) |
| `firstname`, `lastname`, `companyname`, `email` | `RegistrantFirstName`, `RegistrantLastName`, `RegistrantOrganizationName`, `RegistrantEmailAddress` |
| `address1` | `RegistrantAddress1` |
| `address2` | none: `buildRegisterParams()` doesn't provide a second address line |
| `city`, `postcode`, `countrycode` | `RegistrantCity`, `RegistrantPostalCode`, `RegistrantCountry` |
| `state` / `statecode` | `RegistrantStateProvince` / `RegistrantStateProvinceCode` |
| `phonenumber` / `fullphonenumber` | `RegistrantPhone`, unformatted: normalize it yourself |
| `admin*` contact fields | none: Clientexec passes one registrant contact. Use it for every contact role the registrar requires. |
| `ns1` ... `ns5` | `NS1['hostname']` ... `NS11['hostname']`, or `usedns = 'default'` |
| `idprotection` | `package_addons['IDPROTECT'] == 1` |
| `dnsmanagement`, `emailforwarding` | the matching entry in `package_addons`, if the product defines one |
| `eppcode` / `transfersecret` | `eppCode` (from `buildTransferParams()`) |
| `additionalfields` | `ExtendedAttributes` |
| `original` (pre-IDN-conversion values) | none |

`$params` only contains the registrant, nameserver, and period keys when you built it with `buildRegisterParams()`, `buildTransferParams()`, or `buildRenewParams()` inside the matching `do<Action>()` method. Direct calls such as `getNameServers()` only get settings, `sld`, `tld`, and `userPackageId`.

---

## 4. Return value mapping

| WHMCS returns | Clientexec returns |
| --- | --- |
| `_CheckAvailability`: `ResultsList` of `SearchResult` objects with `STATUS_NOT_REGISTERED` / `STATUS_REGISTERED` | `['result' => [['domain' => $sld, 'tld' => $tld, 'status' => 0 or 1], ...]]` |
| `_GetNameservers`: `['ns1' => ..., 'ns2' => ...]` | `[0 => ..., 1 => ..., 'usesDefault' => bool, 'hasDefault' => bool]` |
| `_SaveNameservers` receives `ns1` ... `ns5` | `setNameServers()` receives `ns` keyed from `1`, and `default` |
| `_GetRegistrarLock`: `'locked'` / `'unlocked'` | `1` / `0` |
| `_SaveRegistrarLock` receives `lockenabled` (`'locked'` / `'unlocked'`) | `setRegistrarLock()` receives `lock` (`1` / `0`) via `buildLockParams()` |
| `_GetContactDetails`: `['Registrant' => ['First Name' => 'Jane']]` | `['Registrant' => ['FirstName' => ['First Name', 'Jane']]]` |
| `_SaveContactDetails` receives `contactdetails['Registrant']['First Name']` | `setContactInformation()` receives `Registrant_FirstName` |
| `_GetEPPCode`: `['eppcode' => $code]` | `getEPPCode()` returns the code string, or `'N/A'` |
| `_GetDNS`: list of `hostname` / `type` / `address` / `priority` | `['records' => [[id, hostname, address, type]], 'types' => [...], 'default' => true]` |
| `_SaveDNS` receives `dnsrecords` | `setDNS()` receives `records` (the complete zone, with `new` flags) |
| `_Sync`: `['expirydate' => ..., 'active' => bool, 'expired' => bool]` | `getGeneralInfo()`: `expiration`, `is_registered`, `is_expired`, plus `id`, `domain`, `registrationstatus`, `purchasestatus`, `autorenew` |
| `_TransferSync`: `['completed' => true, 'expirydate' => ...]` | `getTransferStatus()` returns a status string, and sets the `Transfer Status` field to `Completed` |
| `_GetTldPricing`: `ResultsList` of `ImportItem` | `['com' => ['pricing' => ['register' => ..., 'transfer' => ..., 'renew' => ...]]]` |

WHMCS lets contact detail keys contain spaces (`'First Name'`). In Clientexec the key becomes part of a form field name (`Registrant_FirstName`), so use the CamelCase keys listed in `create.md`.

---

## 5. Things not to port

- **`localAPI()` and WHMCS tables** (`tbldomains`, `tbldomainpricing`). Use the package custom fields in `create.md`, section 5.
- **`logModuleCall()`**. Use `CE_Lib::log()`, with credentials removed first.
- **Separate admin, tech, and billing contacts** built from WHMCS's `admin*` params. Clientexec passes one registrant; if the registrar requires other roles, fill them from the registrant.
- **Hooks for domain events** in `hooks.php`. Those belong in a snapin.
- **Disabled TLS verification.**

---

## 6. Porting checklist

- [ ] Every WHMCS function mapped using section 1; anything without a mapping raised as a question.
- [ ] Settings read under their bare names.
- [ ] Every `['error' => ...]` return replaced with `throw new CE_Exception(...)`; unsupported features throw `MethodNotImplemented`.
- [ ] Return shapes converted using section 4, especially nameservers, contacts, DNS, and availability.
- [ ] Register, renew, and transfer use the matching `build*Params()` helper.
- [ ] Phone numbers normalized from the raw `RegistrantPhone`.
- [ ] Everything in the `create.md` release checklist.
