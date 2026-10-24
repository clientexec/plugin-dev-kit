# Clientexec Events

Every event Clientexec fires, with the data it passes. A snapin listens to events through its `$listeners` property (`create.md`, section 9).

```php
public $listeners = [
    ['Invoice-Paid', 'onInvoicePaid'],
];

public function onInvoicePaid($e): void
{
    $params = $e->getParams();      // the "Data" column below
    $target = $e->getTarget();      // the object that fired the event
}
```

**Listeners run in every request handled by a Clientexec controller, including scheduled service runs (`cron.php` routes through one), but not REST API requests.** So events fired by services, such as `Ticket-Closed` from auto close and `Service-AutoSuspend-*`, reach snapins too.

**Listeners run after the change has happened.** There are no "before" events, and a listener can't cancel or alter the action.

**Data keys are case sensitive and not fully consistent** (`userId` vs `userid`, `ticketId` vs `ticketid`). Copy them from this table.

**Objects in the data** (`user`, `invoice`, `userPackage`) are the live objects core was using. Use them for reading, and re-load by ID before changing anything.

---

## Clients and accounts

| Event | Fired when | Data |
| --- | --- | --- |
| `Client-Create` | A client is created (admin, orders, signup) | `userid`, `user` (`User`) |
| `Client-Update` | A client's profile is saved (admin, client area, API) | `userid`, `user` (`User`) |
| `Client-Delete` | A client is deleted (admin, API) | `userid`, `user` (`User`) |
| `Client-PasswordChange` | A client's password is changed (admin, client area, password reset) | `userId`, `password` (**the new password in plain text**; never log it) |
| `Account-Login` | A user logs in | target is the `User` |
| `Account-Logoff` | A user logs out | `userid`, `type` (`'staff'` for admin logouts) |

## Orders and packages

| Event | Fired when | Data |
| --- | --- | --- |
| `Order-NewPackage` | A package is created from an order | `userid`, `userPackage` (`UserPackage`), `userPackageId` |
| `UserPackage-Activate` | A package is activated | `userPackageId`, `userPackage` |
| `UserPackage-Suspend` | A package is suspended | `userPackageId`, `userPackage` |
| `UserPackage-UnSuspend` | A package is unsuspended | `userPackageId`, `userPackage` |
| `UserPackage-RequestCancel` | A cancellation is requested | `userPackageId`, `userPackage` |
| `UserPackage-UnRequestCancel` | A cancellation request is withdrawn | `userPackageId`, `userPackage` |
| `UserPackage-Cancel` | A package is cancelled | `userPackageId`, `userPackage` |
| `UserPackage-Delete` | A package is deleted | `userPackageId` |
| `UserPackage-StatusChange` | A package's status changes | `userPackageId`, `oldStatus`, `newStatus` |
| `Package-Update` | A package is updated | the `UserPackage` itself (not an array). Core reads the first listener's return value. |
| `Plugin-Action-<Action>` | After a server or registrar plugin action runs on a package through the action buttons or `callPluginAction()`, for example `Plugin-Action-Create`, `Plugin-Action-Suspend`, `Plugin-Action-Register` | `userPackage` |
| `Service-AutoSuspend-Suspend` | The auto suspend service suspends a package | `userPackage`, `userPackageId` |
| `Service-AutoSuspend-UnSuspend` | The auto suspend service unsuspends a package | `userPackage`, `userPackageId` |

To react to a package changing state however it happened, prefer the `UserPackage-*` events. `Plugin-Action-*` only fires for actions run through the plugin action path, and `Service-AutoSuspend-*` only for the auto suspend service.

## Invoices

| Event | Fired when | Data |
| --- | --- | --- |
| `Invoice-Create` | An invoice is created | `invoiceId`, `invoice` (`Invoice`), `overrideCustomField` |
| `Invoice-Update` | An invoice is updated | `invoiceId`, `invoice` |
| `Invoice-Merge` | Invoices are merged into one | `invoiceId` (the invoice kept), `invoice` |
| `Invoice-Sent` | An invoice email is sent | `invoiceId`, `invoiceTemplate` |
| `Invoice-Paid` | An invoice is marked paid | `invoiceId` |
| `Invoice-PartiallyPaid` | An invoice is marked partially paid | `invoiceId` |
| `Invoice-Pending` | An invoice is marked pending | `invoiceId` |
| `Invoice-Unpaid` | An invoice is marked unpaid | `invoiceId` |
| `Invoice-Refunded` | An invoice is marked refunded | `invoiceId` |
| `Invoice-Credited` | An invoice is marked credited | `invoiceId` |
| `Invoice-Voided` | An invoice is voided | `invoiceId` |
| `Invoice-Delete` | An invoice is deleted | `invoiceId` |

`Invoice-Paid` fires partway through payment processing, before the receipt email and other follow-up work. A listener that throws or hangs there affects the payment itself, so keep it fast and catch everything.

## Support

| Event | Fired when | Data |
| --- | --- | --- |
| `Ticket-CreateByClient` | A client opens a ticket (client area or email) | `ticketId` |
| `Ticket-CreateByAdmin` | Staff open a ticket | `ticketId` |
| `Ticket-ReplyByCustomer` | A client replies (client area or email) | `ticketid`, `isadmin` (`false`), `user`, `message`. Target is the ticket. |
| `Ticket-ReplyByAdmin` | Staff reply (admin or email) | `ticketid`, `isadmin` (`true`), `user`, `message`. Target is the ticket. |
| `Ticket-Closed` | A ticket is closed (admin, or the auto close service) | `ticketId` |
| `Ticket-Deleted` | A ticket is deleted | `ticketId` |

## Servers

| Event | Fired when | Data |
| --- | --- | --- |
| `Server-Add` | A server is added | `serverId` |
| `Server-Edit` | A server is edited | `serverId` |
| `Server-Delete` | A server is deleted | `serverId` |

## System

| Event | Fired when | Data |
| --- | --- | --- |
| `Service-CurrencyRateUpdate` | The currency converter service updates exchange rates | `currencies` (`code => rate`). Core reads the first listener's return value: return `[code => rate]` to override rates. |

---

## Return values

`CE_Lib::trigger()` returns the first listener's return value, and core ignores it except for:

- **`Package-Update`**
- **`Service-CurrencyRateUpdate`**: an array of `currency code => rate` replaces the fetched rates for those currencies.

For every other event, return nothing.
