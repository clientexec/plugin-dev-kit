# Creating a Clientexec Payment Gateway Plugin

A gateway plugin takes payments for invoices and, where the processor supports it, refunds them. Clientexec calls it when a client pays an invoice, when an order is placed at signup, when the invoice processor charges stored payment methods, and when an admin refunds an invoice. Processors that report results asynchronously call back into the plugin through a callback endpoint.

Load `reference/common/conventions.md` first. This file assumes it.

**Examples to copy from:** `mollie` (offsite checkout plus webhook, very small), `stripe` (onsite payment form plus stored payment methods), `stripecheckout` (hosted checkout plus stored payment methods), `offlinebanktransfer` (offline method). `paypal` is complete but large, because it also handles subscriptions; read it only when you need subscriptions.

---

## 1. Pick the kind of gateway first

The kind of gateway decides which methods you implement and which flags you set in `getVariables()`.

| Kind | How the client pays | Examples | What you implement |
| --- | --- | --- | --- |
| **Offline** | Outside Clientexec (bank transfer, cheque); an admin marks the invoice paid | `offlinebanktransfer`, `offlinecheck` | `getVariables()` with `Dummy Plugin = 1`, stub `singlepayment()` and `credit()` |
| **Offsite / hosted** | Redirected to the processor's page, which reports back through a callback | `mollie`, `paypal`, `coinbasecommerce` | `singlepayment()` redirects; a callback class records the result |
| **Onsite form** | Pays on a Clientexec page using the processor's JavaScript; the result is posted back with the form | `stripe`, `razorpay`, `squarepayment` | `getForm()` renders the form; `singlepayment()` verifies and records the result server-side |
| **Stored payment method** | Charged without the client present (invoice processor, autopay) using a token held by the processor | `stripe`, `stripecheckout`, `authnetcim` | `singlepayment()` charges the stored token; profile flags and `Billing-Profile-ID` storage |

Onsite and stored are often the same plugin: the form collects the card once, and later invoices are charged against the stored token.

---

## 2. Files

```
plugins/gateways/acmepay/
├── PluginAcmepay.php          # the gateway (required)
├── PluginAcmepayCallback.php  # callback handler (offsite, webhooks)
├── callback.php               # public callback entry point (offsite, webhooks)
├── index.html
├── logo.png
└── form.phtml                 # onsite form view (any name you like)
```

View file names are your own choice. `stripe` renders its payment form from `sca.phtml` and its signup form from `form.phtml`; the names mean nothing to core.

### `callback.php`

The processor posts to `plugins/gateways/<dir>/callback.php`. The file only routes the request into Clientexec, which loads your callback class and calls `processCallback()`. Use it exactly as shipped plugins do:

```php
<?php

$_GET['fuse'] = 'billing';
$_GET['action'] = 'gatewaycallback';
$_GET['plugin'] = basename(dirname(__FILE__));

chdir('../../..');

require_once dirname(__FILE__) . '/../../../library/front.php';
```

The public URL to give the processor:

```php
$callbackUrl = rtrim(CE_Lib::getSoftwareURL(), '/') . '/plugins/gateways/acmepay/callback.php';
```

You can add query parameters to distinguish callback kinds (`stripe` uses `?isElements=1`, `?initStripe=1`). Treat every query parameter as attacker controlled.

---

## 3. The base classes

### `GatewayPlugin`

`modules/admin/models/GatewayPlugin.php`. Extends `NE_Plugin`, so you get `$this->user`, `$this->settings`, `$this->db`, `$this->view`, and `$this->getVariable()`.

It declares two abstract methods, so every gateway must define both:

- `singlePayment($params)`: take a payment.
- `credit($params)`: refund a payment.

PHP method names are case insensitive. The dispatcher calls `singlepayment`, and nearly every shipped plugin spells it that way.

It also defines `getCommonVariables()`, which adds three settings to every gateway: **In Signup**, **One-Time Payments**, and **Available in Profile Billing Tab**. Don't declare these yourself.

### `PluginCallback`

`modules/admin/models/PluginCallback.php`. Your `PluginAcmepayCallback` extends it. It provides `$this->user`, `$this->settings`, `$this->db`, and `processCallback()`.

The base `processCallback()` returns `true` if the gateway is enabled (any of the three common settings is on) or still assigned to at least one non-admin, non-fraud client, and `false` otherwise. **Call it first and stop if it returns false**, so a disabled, unused gateway can't be driven by forged callbacks:

```php
public function processCallback()
{
    if (!parent::processCallback()) {
        return;
    }

    // ...
}
```

### `Plugin` (the result recorder)

`modules/billing/models/class.gateway.plugin.php` defines a class named `Plugin`. You don't extend it; you instantiate it to record what happened to an invoice. Covered in section 7.

---

## 4. Settings

Gateway settings are read with **spaces kept** in the key (unlike server plugins):

| Where | How to read `API Key` on `acmepay` |
| --- | --- |
| `singlepayment()` / `credit()` | `$params['plugin_acmepay_API Key']` or `$this->getVariable('API Key')` |
| `getForm()` and other plugin methods | `$this->getVariable('API Key')` |
| Callback class | `$this->settings->get('plugin_acmepay_API Key')` |

`getVariable()` returns the coded `value` for `hidden` settings and the stored value for everything else, which is why core reads flags such as `Auto Payment` through it.

Use plain, untranslated setting names (conventions, section 3).

Don't use `encryptable` on gateway settings: it has no effect for gateways. Use the `password` type for secrets so they're masked in the admin UI.

### Settings core reads

These are read by core by name. Declare the ones that apply; leave the rest out.

| Setting | Type | Meaning |
| --- | --- | --- |
| `Plugin Name` | hidden | How Clientexec refers to the plugin in admin screens. |
| `Signup Name` | text | Name shown to clients at signup and in payment method choices. |
| `Dummy Plugin` | hidden | `1` for offline methods that only label how a client pays; `0` for a real gateway. |
| `Auto Payment` | hidden | `1` if the gateway can charge without the client present (stored card or token). Invoices for these clients are charged by the invoice processor, and no "click here to pay" link is generated. |
| `CC Stored Outside` | hidden | `1` if the card or token lives at the processor rather than in Clientexec. The invoice processor charges these clients without a Clientexec passphrase. |
| `Billing Profile ID` | hidden | `1` if the plugin stores a processor customer/profile reference for each client (section 9). |
| `Form` | hidden | `1` if the gateway shows its own form (`getForm()`). Direct payment links send these clients to an invoice form page instead of calling `singlepayment()` immediately. |
| `Accept CC Number` | hidden | `1` if Clientexec itself collects and stores the card number. New gateways should use processor-side tokens and leave this `0`. |
| `Check CVV2` | hidden | `1` to collect CVV2 with a card number. Only meaningful with `Accept CC Number`. |
| `Visa`, `MasterCard`, `AmericanExpress`, `Discover` | yesno | Card types accepted, for gateways that use `Accept CC Number`. |
| `Invoice After Signup` | yesno | Send the client an invoice after signup completes. |
| `Update Gateway` | hidden | `1` to have `UpdateGateway()` called when a client switches to or from this gateway, their profile changes, or they're deleted (section 9). |
| `Call on updateGatewayInformation` | hidden | Serialized array naming a plugin method to call when the client saves their payment method and posted `plugincustomfields` match given conditions (section 9). |
| `openHandler` | hidden | `1` to call the form's `openHandler()` JavaScript in "Edit Your Payment Method" when the client has no stored profile yet. |
| `Iframe Configuration` | text | Attributes for an iframe-based payment form (`width="100%" height="600" ...`). Like `openHandler`, it marks the gateway as one whose form opens on the payment method page. Used by `authnetcim` and `braintree`. |

Everything else (API keys, test mode, display options) is your own setting.

---

## 5. What `$params` contains

`BillingGateway::actOnInvoices()` builds `$params` for `singlepayment()` and `credit()`. This is the complete list:

| Key | Value |
| --- | --- |
| `invoiceNumber` | invoice ID |
| `invoiceTotal` | **amount to charge**: the invoice's balance due. For `credit()`, the amount of the last transaction |
| `invoiceRawAmount` | amount excluding tax when nothing has been paid yet; otherwise the balance due |
| `invoiceTaxes` | tax charged when nothing has been paid yet; otherwise `0` |
| `invoiceSetup` | setup fees on the invoice |
| `invoiceDescription` | `"Invoice: <id>"` |
| `invoiceDueDate` | due date as a Unix timestamp |
| `invoiceTaxRate` | tax rate |
| `invoiceRefundTransactionId` | for `credit()`: the processor transaction ID of the invoice's last transaction; empty otherwise |
| `invoicePackageUnproratedFee`, `invoiceExcludedRecurrings`, `invoiceProratingDays`, `billingCycle`, `sameCycleAmongEntries`, `usingRecurringPlugin` | subscription data (only `paypal` uses these) |
| `isSignup` | true when paying during signup |
| `fromDirectLink` | true when paying from an emailed "click here to pay" link |
| `userID` | `"CE" . <client id>` (handy as a processor reference) |
| `CustomerID` | client ID |
| `userFirstName`, `userLastName`, `userOrganization`, `userEmail`, `userPhone` | client details |
| `userAddress`, `userCity`, `userState`, `userZipcode`, `userCountry` | client address |
| `userCurrency` | invoice currency code |
| `currencytype` | currency as configured for this processor |
| `userCCNumber`, `userCCExp`, `userCCCVV2` | only populated for gateways with `Accept CC Number`. For `credit()`, `userCCNumber` holds the last four digits from the last transaction |
| `companyName`, `companyURL`, `companyBillingEmail` | from Clientexec settings |
| `clientExecURL` | the Clientexec base URL |
| `invoiceviewURLSuccess` | invoice page URL with `paid=1`, for returning the client after success |
| `invoiceviewURLCancel` | invoice page URL with `cancel=1`, for returning the client after cancelling |
| `plugincustomfields` | values posted from your form (section 8), when present |
| `plugin_<dir>_<Setting Name>` | every one of your settings |

Before calling the plugin, core has already run the fraud check. If the balance due is zero, it marks the invoice paid without calling the plugin at all.

---

## 6. `singlepayment()` and `credit()`

### Return values

| Return | Meaning |
| --- | --- |
| `''` (or nothing) | Success. Shown as "Payment Accepted". |
| a string | Failure. Shown as "Payment Rejected" with your string as the reason. |
| `['AMOUNT' => $amount]` | Success, reporting the amount actually processed. Use this from `credit()`. |
| `['FORM' => $html]` | Hand back a gateway form for core to display. |

Offsite gateways don't return; they redirect the browser (`header('Location: ...')`) or print an auto-submitting form, then `exit`.

**Don't throw from these methods.** `actOnInvoices()` doesn't catch exceptions from the plugin. Catch processor errors yourself, record the rejection (section 7), and return the message string. This is the opposite of the server plugin convention.

### When `singlepayment()` is called

- A client pays an invoice from the client area, or a direct payment link.
- An order is placed at signup (`$params['isSignup']` is true).
- An admin processes invoices, or the invoice processor runs for clients with autopay on. For stored-payment gateways, this is how scheduled charges happen, so `singlepayment()` must be able to charge the stored token with no client present.

`stripe` and `stripecheckout` implement the stored charge in a method named `autopayment()` and call it from `singlepayment()`. That's an internal convention of those plugins.

### When `credit()` is called

An admin refunds an invoice. Core uses the gateway recorded on the invoice (the one that took the payment), passes the last transaction's processor ID in `invoiceRefundTransactionId`, and its amount in `invoiceTotal`.

Return `['AMOUNT' => $refunded]` on success. Core then updates the invoice to Refunded (or Partially Paid for a partial refund) and records a refund transaction. Don't change the invoice status yourself. Shipped gateways also record the processor's refund ID through the result recorder with `setAction('refund')` (section 7); that adds a second transaction row with the processor reference, which is useful for reconciliation.

If the processor can't refund, return a message:

```php
public function credit($params)
{
    return $this->user->lang('This payment gateway does not support refunds.');
}
```

---

## 7. Recording results: the `Plugin` class

Every charge, rejection, pending state, and refund is recorded on the invoice through `Plugin`:

```php
require_once 'modules/billing/models/class.gateway.plugin.php';

$cPlugin = new Plugin($invoiceId, 'acmepay', $this->user);
$cPlugin->setAction('charge');
$cPlugin->setAmount($amount);
$cPlugin->setTransactionID($processorTransactionId);
$cPlugin->PaymentAccepted($amount, $this->user->lang('Acmepay payment of %s was accepted (transaction %s)', $amount, $processorTransactionId), $processorTransactionId);
```

The second constructor argument is your plugin's directory name. It's recorded as the gateway used for the invoice, and later decides which gateway `credit()` is called on.

### Setters (call before recording)

| Method | Purpose |
| --- | --- |
| `setAction('charge')` / `setAction('refund')` | **Required.** The default is `'NA'`, and only `'charge'` changes the invoice's paid state. A `PaymentAccepted()` without `setAction('charge')` records a transaction but leaves the invoice unpaid. |
| `setAmount($amount)` | Amount of this transaction. |
| `setTransactionID($id)` | Processor transaction ID. Stored on the transaction, used for duplicate detection and for refunds later. |
| `setLast4($digits)` | Card last four, if the processor gives it to you. |

### Recording methods

| Method | Effect |
| --- | --- |
| `PaymentAccepted($amount, $description, $processorId = '')` | With action `charge`: marks the invoice paid if `$amount` covers the balance due, otherwise partially paid. Overpayments go to the client's credit balance and open a ticket. Sends the receipt email. If an inactive or cancelled client pays, opens a ticket asking an admin to reactivate them. |
| `PaymentRejected($description, $updateInvoice = true)` | Records a failed attempt. With action `charge` and `$updateInvoice` true, sets the invoice back to unpaid and sends the rejection email. Pass `false` when the invoice may already be paid (for example a late failure callback for an older attempt). |
| `PaymentPending($description, $processorId = '')` | Marks the invoice pending (payment started but not confirmed, for example a bank transfer in progress). |
| `PaymentRefunded($amount, $description, $processorId = '')` | Requires action `refund`. Adds the amount back to the balance due and sets the invoice to Refunded or Partially Paid. For refunds the processor reports through a callback (initiated outside Clientexec), not for `credit()`. |

`$description` becomes the transaction text admins see on the invoice. Include the amount and the processor's transaction ID.

### Lookup helpers

| Method | Use |
| --- | --- |
| `TransExists($processorId)` | `true` if any invoice already has a transaction with this processor ID. |
| `retrieveInvoiceForTransaction($processorId)` | Loads the invoice that has this transaction ID into the object. Returns `true` if found. |
| `IsPaid()` / `IsUnpaid()` | State of the loaded invoice. |
| `$cPlugin->m_Invoice` | The loaded `Invoice`, for currency, balance, and client checks. |
| `ForwardUser($isSignup)` | Redirects the browser to the signup completion page when `$isSignup` is `1`. |

### Duplicate protection is narrow

`PaymentAccepted()` skips recording only when the **same invoice** already has a transaction with the same transaction ID, response text, action, last four, and amount. It does not stop one processor payment from being applied to two different invoices, and a retried callback that builds a slightly different description is recorded twice.

So before accepting any payment that arrives from a callback or a form post, check it hasn't been used:

```php
if ($cPlugin->TransExists($paymentId)) {
    CE_Lib::log(4, 'plugin_acmepay: payment ' . $paymentId . ' already recorded, ignoring');
    return;
}
```

---

## 8. Never trust the client or the callback

This is where gateway plugins go wrong. A forged or replayed request that reaches `PaymentAccepted()` marks an invoice paid for free.

**Rules:**

1. **Get the facts from the processor, not the request.** A callback or form post should give you an identifier only. Fetch that payment from the processor's API with your secret key, and use the amount, currency, status, and invoice reference from that response. `mollie` does exactly this: its webhook receives only an ID and everything else comes from `$mollie->payments->get()`.
2. **If the processor pushes the full payload, verify its signature** with the processor's documented method before reading anything from it.
3. **Tie the payment to this invoice using something you set server-side.** Put the invoice ID in the processor's metadata when you create the payment, or store the processor's order ID on the invoice (`$invoice->SetProcessorID($orderId)`) and compare. Never accept an invoice ID chosen by the browser on its own.
4. **Check the currency** matches `$cPlugin->m_Invoice->getCurrency()`.
5. **Record the amount the processor reports**, not the invoice total. `PaymentAccepted()` handles partial payment correctly when the amount is lower.
6. **Check `TransExists()`** before recording (section 7).
7. **Fail closed.** If you can't reach the processor, can't verify, or get a response you don't recognize, record a rejection or do nothing. Never fall back to treating the invoice total as paid.

### Onsite form fields

Values from your form arrive in `$params['plugincustomfields']`. Name the inputs `<dir>_plugincustomfields[<key>]`:

```html
<input type="hidden" id="acmepay_payment_id" name="acmepay_plugincustomfields[payment_id]" value="">
```

arrives as `$params['plugincustomfields']['payment_id']`. Everything in it was posted by the browser, so rules 1 to 7 apply in full.

---

## 9. Onsite forms and stored payment methods

### `getForm($params)`

With `Form = 1`, Clientexec calls `getForm()` and inserts the HTML it returns. `$params['from']` says where:

| `from` | Page |
| --- | --- |
| `invoice` | the invoice payment page |
| `signup` | the order form |
| `paymentmethod` | the client's "Edit Your Payment Method" page |

On the invoice page, `$params` contains:

| Key | Value |
| --- | --- |
| `from` | `'invoice'` |
| `invoiceId` | invoice ID |
| `invoiceBalanceDue` | balance due |
| `currency` | invoice currency |
| `companyName` | company name |
| `panellabel` | translated "Pay" label |
| `userZipcode` | the logged-in client's postal code |

Shipped signup forms also read `currency`, `loggedIn`, and `termsConditions`.

**The invoice page calls `getForm()` on every gateway with `Form = 1`, on every view,** not only on the client's own gateway. So:

- Return `''` quickly when the gateway isn't configured, or isn't the one this client should see.
- Be deliberate about creating processor-side objects here. `stripe` creates a PaymentIntent and `razorpay` creates an order on each render, and `razorpay` overwrites the invoice's stored processor ID each time. If your processor charges for or rate-limits object creation, create the payment when the client actually starts paying (an AJAX call to your callback endpoint, as `stripe`'s `initStripe` does) rather than on page load.

The view's script path already includes your plugin directory, so render directly:

```php
$this->view->publishableKey = $this->getVariable('Publishable Key');
$this->view->from = $params['from'];
return $this->view->render('form.phtml');
```

To let `singlepayment()` confirm the client paid a specific order, store the processor's order or payment ID server-side when you create it (`$invoice->SetProcessorID($orderId)`), then compare against it when the result is posted back. `razorpay` shows the shape of this flow.

Never output a secret key into a view. Only publishable or public keys belong in the page.

### Stored payment profiles

Gateways that keep a processor customer and payment method for each client (`Billing Profile ID = 1`) store them in the client's `Billing-Profile-ID` custom field. The value is a serialized array keyed by plugin directory, so several gateways can share the field:

```php
$profiles = [];
$stored = '';
if ($user->getCustomFieldsValue('Billing-Profile-ID', $stored) && $stored != '') {
    $profiles = unserialize($stored);
}

if (!is_array($profiles)) {
    $profiles = [];
}

$profiles['acmepay'] = $customerId . '|' . $paymentMethodId;
$user->updateCustomTag('Billing-Profile-ID', serialize($profiles));
$user->save();
```

`stripe` and `stripecheckout` store `customer|payment_method`; the format of your own entry is up to you.

### Updating the payment method: `Call on updateGatewayInformation`

When a client saves "Edit Your Payment Method", Clientexec reads the `Call on updateGatewayInformation` setting of the gateway they're switching to. If it names a method and every condition matches the posted `plugincustomfields`, that method is called.

The setting is a serialized array:

```php
'Call on updateGatewayInformation' => [
    'type' => 'hidden',
    'description' => lang('Function name to be called in this plugin when given conditions are meet while updateGatewayInformation is invoked'),
    'value' => serialize([
        'function' => 'updatePaymentMethod',
        'plugincustomfields conditions' => [
            [
                'field name' => 'acmepayUpdatePaymentMethod',
                'operator' => '==',
                'value' => '1'
            ]
        ]
    ])
],
```

- Every condition must match. If a named field wasn't posted, or an operator isn't one of `==`, `!=`, `<`, `<=`, `>`, `>=`, the method isn't called.
- `field name` is the key inside `plugincustomfields`, so a form input named `acmepay_plugincustomfields[acmepayUpdatePaymentMethod]` matches `acmepayUpdatePaymentMethod`.
- Core replaces `&quot;` with `"` before unserializing, because saving the setting HTML-encodes the quotes. Keep the serialized value free of other characters that get encoded (`<`, `>`, `&`, `'`).

The method receives:

| Key | Value |
| --- | --- |
| `CustomerID` | client ID |
| `userFirstName`, `userLastName`, `userOrganization`, `userEmail`, `userPhone` | client details |
| `userAddress`, `userCity`, `userState`, `userZipcode`, `userCountry` | client address |
| `location` | `'updateGatewayInformation'` |
| `clientExecURL` | Clientexec base URL |
| `invoiceviewURLCancel` | the payment method page URL |
| `plugincustomfields` | posted form values |

Return `['error' => true, 'detail' => $message]` to show the client "There was an issue updating your Credit Card" with your detail and send them back to the payment method page. Anything else is treated as success.

### `UpdateGateway($params)` and `CustomerRemove($params)`

With `Update Gateway = 1`, Clientexec calls `UpdateGateway()` on the client's current gateway when their profile changes, and when they switch away from the gateway, change status, or are deleted. `$params` contains:

| Key | Value |
| --- | --- |
| `User ID` | client ID (note the space) |
| `Status` | client's current status ID |
| `Gateway` | the gateway's directory name |
| `Action` | `'update'` or `'delete'` |

Shipped plugins handle `update` by removing the processor customer only when the client is now inactive, cancelled, or fraud, and handle `delete` by removing it outright. They do the removal in a `CustomerRemove($params)` helper, which reads `$params['User ID']`. See `stripecheckout` for the full pattern, including a `Delete Client From Gateway` setting that lets admins keep processor records.

---

## 10. Callbacks in full

A callback class has one job: turn a processor notification into exactly one correct `Plugin` call.

```php
<?php

require_once 'modules/admin/models/PluginCallback.php';
require_once 'modules/billing/models/class.gateway.plugin.php';

class PluginAcmepayCallback extends PluginCallback
{
    public function processCallback()
    {
        if (!parent::processCallback()) {
            return;
        }

        // 1. Identifier only, from the request
        // 2. Fetch the payment from the processor with the secret key
        // 3. Invoice reference from the processor's response
        // 4. TransExists, currency check
        // 5. One Plugin call based on the processor's status
    }
}
```

Processors retry callbacks they think failed, so a callback must be safe to receive twice (that's what the `TransExists()` check is for).

For a callback that returns the client's browser (rather than a server-to-server webhook), redirect when you're done: to `order.php?step=complete&pass=1` for signup, or to the invoice page with `paid=1` or `cancel=1`.

---

## 11. Offline gateways

An offline method needs only settings and stubs:

```php
<?php

require_once 'modules/admin/models/GatewayPlugin.php';

class PluginOfflinewire extends GatewayPlugin
{
    public function getVariables()
    {
        $variables = [
            'Plugin Name' => [
                'type' => 'hidden',
                'description' => lang('How CE sees this plugin (not to be confused with the Signup Name)'),
                'value' => lang('Wire Transfer')
            ],
            'Signup Name' => [
                'type' => 'text',
                'description' => lang('Name to display in the signup process for this payment type.'),
                'value' => 'Wire Transfer'
            ],
            'Invoice After Signup' => [
                'type' => 'yesno',
                'description' => lang('Select YES if you want an invoice sent to the client after signup is complete.'),
                'value' => '1'
            ],
            'Dummy Plugin' => [
                'type' => 'hidden',
                'description' => lang('1 = Only used to specify a billing type for a client. 0 = full fledged plugin requiring complete functions'),
                'value' => '1'
            ],
            'Auto Payment' => [
                'type' => 'hidden',
                'description' => lang('No description'),
                'value' => '0'
            ]
        ];

        return $variables;
    }

    public function singlepayment($params)
    {
        return '';
    }

    public function credit($params)
    {
        return $this->user->lang('This payment gateway does not support refunds.');
    }
}
```

---

## 12. Complete example: offsite checkout with a webhook

A fictional processor called Acmepay with a hosted checkout page and a webhook that sends only the payment ID. This is the safest shape for a new gateway and covers most modern processors.

### `plugins/gateways/acmepay/PluginAcmepay.php`

```php
<?php

require_once 'modules/admin/models/GatewayPlugin.php';
require_once 'modules/billing/models/class.gateway.plugin.php';
require_once __DIR__ . '/AcmepayApi.php';

class PluginAcmepay extends GatewayPlugin
{
    public function getVariables()
    {
        $variables = [
            'Plugin Name' => [
                'type' => 'hidden',
                'description' => lang('How CE sees this plugin (not to be confused with the Signup Name)'),
                'value' => 'Acmepay'
            ],
            'Secret Key' => [
                'type' => 'password',
                'description' => lang('Secret API key from your Acmepay dashboard. Use your test key while testing.'),
                'value' => ''
            ],
            'Test Mode' => [
                'type' => 'yesno',
                'description' => lang('Send requests to the Acmepay sandbox.'),
                'value' => '0'
            ],
            'Signup Name' => [
                'type' => 'text',
                'description' => lang('Name to display in the signup process for this payment type.'),
                'value' => 'Acmepay'
            ],
            'Invoice After Signup' => [
                'type' => 'yesno',
                'description' => lang('Select YES if you want an invoice sent to the client after signup is complete.'),
                'value' => '1'
            ],
            'Dummy Plugin' => [
                'type' => 'hidden',
                'description' => lang('1 = Only used to specify a billing type for a client. 0 = full fledged plugin requiring complete functions'),
                'value' => '0'
            ],
            'Auto Payment' => [
                'type' => 'hidden',
                'description' => lang('No description'),
                'value' => '0'
            ]
        ];

        return $variables;
    }

    public function singlepayment($params)
    {
        $api = new AcmepayApi($params['plugin_acmepay_Secret Key'], $params['plugin_acmepay_Test Mode'] == '1');
        $baseUrl = rtrim($params['clientExecURL'], '/');

        if ($params['isSignup']) {
            $returnUrl = $baseUrl . '/order.php?step=complete&pass=1';
            $cancelUrl = $baseUrl . '/order.php?step=3';
        } else {
            $returnUrl = $params['invoiceviewURLSuccess'];
            $cancelUrl = $params['invoiceviewURLCancel'];
        }

        try {
            $checkout = $api->post('/checkouts', [
                'amount' => sprintf('%01.2f', round($params['invoiceTotal'], 2)),
                'currency' => $params['userCurrency'],
                'description' => $this->user->lang('Invoice #%s', $params['invoiceNumber']),
                'customer_email' => $params['userEmail'],
                'return_url' => $returnUrl,
                'cancel_url' => $cancelUrl,
                'webhook_url' => $baseUrl . '/plugins/gateways/acmepay/callback.php',
                'metadata' => [
                    'invoice_id' => (string) $params['invoiceNumber']
                ]
            ]);
        } catch (Exception $e) {
            CE_Lib::log(1, 'plugin_acmepay::singlepayment::error for invoice ' . $params['invoiceNumber'] . ': ' . $e->getMessage());
            return $this->user->lang('We could not start your Acmepay payment. Please try again later.');
        }

        if (empty($checkout['url'])) {
            CE_Lib::log(1, 'plugin_acmepay::singlepayment::error for invoice ' . $params['invoiceNumber'] . ': no checkout URL in response');
            return $this->user->lang('We could not start your Acmepay payment. Please try again later.');
        }

        header('Location: ' . $checkout['url'], true, 303);
        exit;
    }

    public function credit($params)
    {
        $api = new AcmepayApi($params['plugin_acmepay_Secret Key'], $params['plugin_acmepay_Test Mode'] == '1');
        $amount = sprintf('%01.2f', round($params['invoiceTotal'], 2));

        $cPlugin = new Plugin($params['invoiceNumber'], 'acmepay', $this->user);
        $cPlugin->setAction('refund');
        $cPlugin->setAmount($amount);

        if ($params['invoiceRefundTransactionId'] == '') {
            return $this->user->lang('This invoice has no Acmepay transaction to refund.');
        }

        try {
            $refund = $api->post('/payments/' . rawurlencode($params['invoiceRefundTransactionId']) . '/refunds', [
                'amount' => $amount
            ]);
        } catch (Exception $e) {
            $message = $this->user->lang('Acmepay refund failed: %s', $e->getMessage());
            $cPlugin->PaymentRejected($message, false);
            return $message;
        }

        if (($refund['status'] ?? '') !== 'succeeded') {
            $message = $this->user->lang('Acmepay did not accept the refund.');
            $cPlugin->PaymentRejected($message, false);
            return $message;
        }

        $refunded = sprintf('%01.2f', round($refund['amount'], 2));
        $cPlugin->setAmount($refunded);
        $cPlugin->setTransactionID($refund['id']);
        $cPlugin->PaymentAccepted($refunded, $this->user->lang('Acmepay refund of %s was processed (refund %s)', $refunded, $refund['id']), $refund['id']);

        return ['AMOUNT' => $refunded];
    }
}
```

### `plugins/gateways/acmepay/PluginAcmepayCallback.php`

```php
<?php

require_once 'modules/admin/models/PluginCallback.php';
require_once 'modules/billing/models/class.gateway.plugin.php';
require_once __DIR__ . '/AcmepayApi.php';

class PluginAcmepayCallback extends PluginCallback
{
    public function processCallback()
    {
        if (!parent::processCallback()) {
            return;
        }

        $body = json_decode((string) file_get_contents('php://input'), true);
        $paymentId = is_array($body) ? (string) ($body['payment_id'] ?? '') : '';
        if (preg_match('/^[A-Za-z0-9_-]{1,64}$/', $paymentId) !== 1) {
            CE_Lib::log(4, 'plugin_acmepay callback: missing or malformed payment ID');
            return;
        }

        $api = new AcmepayApi(
            $this->settings->get('plugin_acmepay_Secret Key'),
            $this->settings->get('plugin_acmepay_Test Mode') == '1'
        );

        try {
            $payment = $api->get('/payments/' . rawurlencode($paymentId));
        } catch (Exception $e) {
            CE_Lib::log(1, 'plugin_acmepay callback: could not fetch payment ' . $paymentId . ': ' . $e->getMessage());
            return;
        }

        $invoiceId = (int) ($payment['metadata']['invoice_id'] ?? 0);
        if ($invoiceId <= 0) {
            CE_Lib::log(1, 'plugin_acmepay callback: payment ' . $paymentId . ' has no invoice reference');
            return;
        }

        $cPlugin = new Plugin($invoiceId, 'acmepay', $this->user);
        if (!$cPlugin->m_Invoice->getId()) {
            CE_Lib::log(1, 'plugin_acmepay callback: invoice ' . $invoiceId . ' for payment ' . $paymentId . ' does not exist');
            return;
        }

        if (strtoupper((string) $payment['currency']) !== strtoupper((string) $cPlugin->m_Invoice->getCurrency())) {
            CE_Lib::log(1, 'plugin_acmepay callback: currency mismatch on payment ' . $paymentId . ' for invoice ' . $invoiceId);
            return;
        }

        $amount = sprintf('%01.2f', round((float) $payment['amount'], 2));
        $cPlugin->setAction('charge');
        $cPlugin->setAmount($amount);
        $cPlugin->setTransactionID($paymentId);

        switch ($payment['status']) {
            case 'paid':
                if ($cPlugin->TransExists($paymentId)) {
                    CE_Lib::log(4, 'plugin_acmepay callback: payment ' . $paymentId . ' already recorded');
                    return;
                }
                $cPlugin->PaymentAccepted($amount, 'Acmepay payment of ' . $amount . ' was accepted (payment ' . $paymentId . ')', $paymentId);
                break;

            case 'pending':
                $cPlugin->PaymentPending('Acmepay payment of ' . $amount . ' is pending (payment ' . $paymentId . ')', $paymentId);
                break;

            case 'failed':
            case 'expired':
            case 'canceled':
                $cPlugin->PaymentRejected('Acmepay payment of ' . $amount . ' was not completed: ' . $payment['status'] . ' (payment ' . $paymentId . ')', $cPlugin->IsUnpaid());
                break;

            default:
                CE_Lib::log(4, 'plugin_acmepay callback: ignoring status ' . $payment['status'] . ' for payment ' . $paymentId);
                break;
        }
    }
}
```

The `TransExists()` check sits inside the `paid` case because a pending notification for the same payment ID may legitimately have been recorded first.

### `plugins/gateways/acmepay/AcmepayApi.php`

```php
<?php

class AcmepayApi
{
    private string $secretKey;
    private string $baseUrl;

    public function __construct(string $secretKey, bool $testMode)
    {
        $this->secretKey = trim($secretKey);
        $this->baseUrl = $testMode ? 'https://sandbox.acmepay.example/v1' : 'https://api.acmepay.example/v1';
    }

    public function get(string $path): array
    {
        return $this->request('GET', $path);
    }

    public function post(string $path, array $body): array
    {
        return $this->request('POST', $path, $body);
    }

    private function request(string $method, string $path, array $body = []): array
    {
        if ($this->secretKey === '') {
            throw new CE_Exception('Acmepay secret key is not configured.');
        }

        $ch = curl_init();
        curl_setopt($ch, CURLOPT_URL, $this->baseUrl . $path);
        curl_setopt($ch, CURLOPT_CUSTOMREQUEST, $method);
        curl_setopt($ch, CURLOPT_RETURNTRANSFER, true);
        curl_setopt($ch, CURLOPT_SSL_VERIFYPEER, true);
        curl_setopt($ch, CURLOPT_SSL_VERIFYHOST, 2);
        curl_setopt($ch, CURLOPT_CAINFO, \Composer\CaBundle\CaBundle::getSystemCaRootBundlePath());
        curl_setopt($ch, CURLOPT_TIMEOUT, 30);
        curl_setopt($ch, CURLOPT_HTTPHEADER, [
            'Authorization: Bearer ' . $this->secretKey,
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
            throw new CE_Exception('Connection to Acmepay failed: ' . $error);
        }

        $status = curl_getinfo($ch, CURLINFO_HTTP_CODE);
        curl_close($ch);

        $decoded = json_decode($response, true);
        if (!is_array($decoded)) {
            throw new CE_Exception('Acmepay returned an unexpected response (HTTP ' . $status . ').');
        }

        if ($status < 200 || $status >= 300) {
            throw new CE_Exception('Acmepay API error: ' . ($decoded['error'] ?? ('HTTP ' . $status)));
        }

        return $decoded;
    }
}
```

### `plugins/gateways/acmepay/callback.php`

```php
<?php

$_GET['fuse'] = 'billing';
$_GET['action'] = 'gatewaycallback';
$_GET['plugin'] = basename(dirname(__FILE__));

chdir('../../..');

require_once dirname(__FILE__) . '/../../../library/front.php';
```

Add `logo.png` and an empty `index.html`.

---

## 13. Checklist before release

- [ ] `singlepayment()` and `credit()` both exist; neither throws.
- [ ] Every `Plugin` object gets `setAction()` before any `Payment*()` call.
- [ ] The amount recorded is what the processor reports.
- [ ] Callbacks and form posts fetch or signature-verify the payment at the processor before recording anything.
- [ ] The payment is tied to the invoice by something set server-side (metadata or stored processor/order ID).
- [ ] Currency checked against the invoice.
- [ ] `TransExists()` checked before `PaymentAccepted()` on any callback or form-post path.
- [ ] Every failure path records a rejection or does nothing; none falls back to the invoice total.
- [ ] Callback class calls `parent::processCallback()` first.
- [ ] TLS verification on, using the Composer CA bundle.
- [ ] No secret key in any view, log line, or transaction description.
- [ ] Settings core reads (section 4) set correctly for the kind of gateway.
- [ ] Setting names are plain strings, not `lang()` calls.
- [ ] `credit()` returns `['AMOUNT' => ...]` on success and doesn't change invoice status itself.

---

## 14. Modernizing an older gateway

Common issues in older gateways, in priority order:

1. **Unverified results.** Find every path into `PaymentAccepted()` and apply section 8. Pay special attention to values taken from `$_POST`, `$_GET`, or `plugincustomfields`, and to `catch` blocks that fall back to `$params['invoiceTotal']`.
2. **TLS:** remove `CURLOPT_SSL_VERIFYPEER => false` / `0` and set `CURLOPT_CAINFO` from the Composer CA bundle. Some older plugins set the CA bundle and then disable verification anyway; remove the disable.
3. **Stored card numbers:** gateways using `Accept CC Number = 1` keep card data in Clientexec. Moving to processor tokens (`CC Stored Outside = 1`) takes Clientexec out of card storage.
4. **Missing `setAction()`:** add it before every `Payment*()` call.
5. **Unreachable or undefined code** after copy-paste: variables used in transaction descriptions that were never set, and status branches that can't be reached because an earlier branch already matches.
6. **Translated setting keys:** wrapping keys in `lang()` can't be removed without migrating stored values, because the stored key changes. Plan it as a data migration, not a code cleanup.
7. **Style:** conventions, section 11.

---

## Canonical reference files

| What | File |
| --- | --- |
| Gateway base class, common settings | `modules/admin/models/GatewayPlugin.php` |
| Callback base class | `modules/admin/models/PluginCallback.php` |
| Result recorder (`Plugin`) | `modules/billing/models/class.gateway.plugin.php` |
| Dispatcher, `$params`, return handling | `modules/billing/models/BillingGateway.php` (`actOnInvoices()`, `processRefunds()`) |
| Settings access, view setup | `library/CE/NE_Plugin.php` |
| Minimal offsite plus webhook | `plugins/gateways/mollie/` |
| Onsite form plus stored payment methods | `plugins/gateways/stripe/` |
| Hosted checkout plus stored payment methods | `plugins/gateways/stripecheckout/` |
| Offline | `plugins/gateways/offlinebanktransfer/` |
