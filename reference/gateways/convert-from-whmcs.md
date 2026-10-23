# Converting a WHMCS Payment Gateway Module to a Clientexec Gateway

Load `reference/common/conventions.md` and `reference/gateways/create.md` first. This file only maps WHMCS concepts onto them.

## The short version

| WHMCS | Clientexec |
| --- | --- |
| `modules/gateways/acmepay.php` (functions prefixed `acmepay_`) | `plugins/gateways/acmepay/PluginAcmepay.php` (`class PluginAcmepay extends GatewayPlugin`) |
| `modules/gateways/callback/acmepay.php` | `plugins/gateways/acmepay/callback.php` (fixed boilerplate) plus `PluginAcmepayCallback.php` (`extends PluginCallback`) |
| `acmepay_config()` | `getVariables()` |
| `acmepay_MetaData()` | settings core reads (`create.md`, section 4) |

Decide which kind of gateway you're porting (`create.md`, section 1) before writing anything. WHMCS signals this with which functions exist: `_link` means offsite, `_capture` means merchant or stored token, `_remoteinput` / `_storeremote` mean tokenized.

---

## 1. Function mapping

| WHMCS function | Clientexec equivalent |
| --- | --- |
| `_config()` | `getVariables()` |
| `_MetaData()` with `TokenisedStorage` | `Auto Payment = 1`, `CC Stored Outside = 1`, `Billing Profile ID = 1` |
| `_link($params)` (returns an HTML form or button) | `singlepayment()` redirects (`header('Location: ...')`) or prints an auto-submitting form, then `exit` |
| `_capture($params)` (charge a card or token) | `singlepayment()`, charging the stored token |
| `_refund($params)` | `credit()`, returning `['AMOUNT' => $refunded]` |
| `_remoteinput($params)` (onsite tokenization form) | `getForm()` with `Form = 1` |
| `_remoteupdate($params)` (update stored card form) | `getForm()` when `$params['from'] == 'paymentmethod'`, plus a `Call on updateGatewayInformation` method to save the result (`create.md`, section 9) |
| `_storeremote($params)` with `action` create / update / delete | Create/update: the `Call on updateGatewayInformation` method. Delete: `UpdateGateway()` / `CustomerRemove()` with `Update Gateway = 1`. Tokens live in the `Billing-Profile-ID` client field |
| `_nolocalcc()` | `Accept CC Number = 0` |
| `_adminstatusmsg()` | No equivalent |
| `_cancelSubscription()` | Only `paypal` implements subscriptions; read it before porting subscription support |

If a WHMCS function has no row here, ask rather than inventing a Clientexec hook.

---

## 2. Settings: `_config()` to `getVariables()`

```php
// WHMCS
function acmepay_config()
{
    return [
        'FriendlyName' => ['Type' => 'System', 'Value' => 'Acmepay'],
        'secretKey' => ['FriendlyName' => 'Secret Key', 'Type' => 'password'],
        'testMode' => ['FriendlyName' => 'Test Mode', 'Type' => 'yesno'],
    ];
}
// used as $params['secretKey'], $params['testMode']
```

```php
// Clientexec
public function getVariables()
{
    $variables = [
        'Plugin Name' => [
            'type' => 'hidden',
            'description' => lang('How CE sees this plugin (not to be confused with the Signup Name)'),
            'value' => 'Acmepay'
        ],
        'Signup Name' => [
            'type' => 'text',
            'description' => lang('Name to display in the signup process for this payment type.'),
            'value' => 'Acmepay'
        ],
        'Secret Key' => [
            'type' => 'password',
            'description' => lang('Secret API key from your Acmepay dashboard.'),
            'value' => ''
        ],
        'Test Mode' => [
            'type' => 'yesno',
            'description' => lang('Send requests to the Acmepay sandbox.'),
            'value' => '0'
        ],
        'Dummy Plugin' => [
            'type' => 'hidden',
            'description' => lang('1 = Only used to specify a billing type for a client. 0 = full fledged plugin requiring complete functions'),
            'value' => '0'
        ]
    ];

    return $variables;
}
// used as $params['plugin_acmepay_Secret Key'], $params['plugin_acmepay_Test Mode']
```

| WHMCS field type | Clientexec type |
| --- | --- |
| `System` (`FriendlyName`) | `Plugin Name` (hidden) and `Signup Name` (text) |
| `text` | `text` |
| `password` | `password` |
| `yesno` | `yesno` |
| `dropdown` with `Options` | `options` with an `'options' => [...]` array |
| `textarea` | `textarea` |
| `radio` | `options` |

WHMCS keys settings by a code (`secretKey`). Clientexec keys them by the setting name itself, and keeps the spaces: `$params['plugin_acmepay_Secret Key']`. In a callback class, read them with `$this->settings->get('plugin_acmepay_Secret Key')` instead of `getGatewayVariables()`.

Don't port WHMCS's "Convert To For Processing" option: Clientexec passes the invoice's own currency as `userCurrency`.

---

## 3. `$params` mapping

| WHMCS `$params` | Clientexec `$params` |
| --- | --- |
| `invoiceid` | `invoiceNumber` |
| `amount` | `invoiceTotal` (the balance due) |
| `currency` | `userCurrency` |
| `description` | `invoiceDescription` |
| `clientdetails['firstname']` / `['lastname']` | `userFirstName` / `userLastName` |
| `clientdetails['email']` | `userEmail` |
| `clientdetails['address1']`, `['city']`, `['state']`, `['postcode']`, `['country']` | `userAddress`, `userCity`, `userState`, `userZipcode`, `userCountry` |
| `clientdetails['phonenumber']` | `userPhone` |
| `clientdetails['companyname']` | `userOrganization` |
| `clientdetails['userid']` / `['id']` | `CustomerID` |
| `systemurl` | `clientExecURL` |
| `returnurl` | `invoiceviewURLSuccess` (or `order.php?step=complete&pass=1` when `isSignup`) |
| `companyname` | `companyName` |
| `transid` (in `_refund`) | `invoiceRefundTransactionId` |
| `gatewayid` (stored token) | your entry in the client's `Billing-Profile-ID` field (`create.md`, section 9) |
| `cardnum`, `cardexp`, `cccvv` | `userCCNumber`, `userCCExp`, `userCCCVV2` (only with `Accept CC Number = 1`) |
| `<setting code>` | `plugin_<dir>_<Setting Name>` |

---

## 4. Return values

| WHMCS | Clientexec |
| --- | --- |
| `_link` returns HTML | `singlepayment()` redirects or prints, then `exit` |
| `_capture` returns `['status' => 'success', 'transid' => $id, 'rawdata' => $data]` | Record with `Plugin::PaymentAccepted()`, return `''` |
| `_capture` returns `['status' => 'declined', ...]` | Record with `Plugin::PaymentRejected()`, return the reason string |
| `_refund` returns `['status' => 'success', 'transid' => $id, 'fees' => $fee]` | Return `['AMOUNT' => $refunded]` |
| `_refund` returns `['status' => 'error', ...]` | Return the reason string |

WHMCS records the payment for you from the returned array. Clientexec doesn't: for charges, **you** call `Plugin::PaymentAccepted()` or `PaymentRejected()` (`create.md`, section 7). A capture that returns success without recording leaves the invoice unpaid.

Don't throw exceptions out of `singlepayment()` or `credit()`. Catch, record, and return a string.

---

## 5. Callbacks

| WHMCS callback helper | Clientexec |
| --- | --- |
| `require '../../../init.php'` and friends | the fixed `callback.php` boilerplate (`create.md`, section 2) |
| `getGatewayVariables('acmepay')` | `$this->settings->get('plugin_acmepay_<Setting Name>')` |
| `if (!$gatewayParams['type']) die('Module Not Activated')` | `if (!parent::processCallback()) { return; }` |
| `validateCallbackInvoiceID($invoiceId, $gatewayName)` | `$cPlugin = new Plugin($invoiceId, 'acmepay', $this->user);` then check `$cPlugin->m_Invoice` |
| `checkCbTransID($transactionId)` | `if ($cPlugin->TransExists($transactionId)) { return; }` |
| `logTransaction($gatewayName, $data, $status)` | the `$description` argument of the `Payment*()` call, plus `CE_Lib::log()` for raw payloads |
| `addInvoicePayment($invoiceId, $transId, $amount, $fee, $gateway)` | `setAction('charge')`, `setAmount()`, `setTransactionID()`, then `PaymentAccepted($amount, $description, $transId)` |
| `callback3DSecureRedirect($invoiceId, $success)` | redirect to `invoiceviewURLSuccess` / `invoiceviewURLCancel`, or `order.php?step=complete&pass=1` for signup |

### Example

```php
// WHMCS callback
$gatewayParams = getGatewayVariables('acmepay');
if (!$gatewayParams['type']) {
    die('Module Not Activated');
}

$invoiceId = validateCallbackInvoiceID($_POST['invoice_id'], 'acmepay');
checkCbTransID($_POST['transaction_id']);
logTransaction('acmepay', $_POST, 'Success');
addInvoicePayment($invoiceId, $_POST['transaction_id'], $_POST['amount'], 0, 'acmepay');
```

Notice that this common WHMCS pattern takes the invoice ID, transaction ID, and amount straight from `$_POST`. Many WHMCS modules rely on a signature check elsewhere in the file to make that safe; some don't. Don't port the trust: in Clientexec, fetch the payment from the processor by ID or verify the payload signature, and use the invoice reference, amount, and currency from the verified data (`create.md`, section 8). The `PluginAcmepayCallback` class in `create.md`, section 12 is the Clientexec version of this callback.

---

## 6. Things not to port

- **WHMCS tables.** `tblinvoices`, `tblaccounts`, `tblclients` and `localAPI('AddInvoicePayment', ...)` have no place in a Clientexec gateway. Use the `Plugin` recorder, `Invoice`, and `User`.
- **`$_SESSION` lookups for the invoice.** Put the invoice ID in the processor's metadata when you create the payment.
- **Disabled TLS verification.** Use the Composer CA bundle (conventions, section 6).
- **Local card storage.** If the WHMCS module stores card numbers locally and the processor supports tokens, port to tokens (`CC Stored Outside = 1`).

---

## 7. Porting checklist

- [ ] Kind of gateway decided (`create.md`, section 1).
- [ ] `_config()` mapped to `getVariables()` with plain setting names, plus the core-read settings for that kind.
- [ ] `_link` / `_capture` mapped to `singlepayment()`, which records through `Plugin` and never throws.
- [ ] `_refund` mapped to `credit()`, returning `['AMOUNT' => ...]`.
- [ ] Callback rebuilt on `PluginCallback`, calling `parent::processCallback()` first.
- [ ] Every value used to record a payment comes from the processor's API or a verified signature, never directly from the request.
- [ ] `TransExists()` and a currency check before `PaymentAccepted()`.
- [ ] No WHMCS functions, tables, or globals left.
- [ ] Everything in the `create.md` release checklist.
