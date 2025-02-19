---
title: Migrate Custom Email Templates | Commerce Frontend Development
description: Learn how to migrate custom email templates between versions of Adobe Commerce and Magento Open Source themes.
---

# Migrate custom email templates

This topic explains how to migrate custom email templates between versions of Adobe Commerce and Magento Open Source.

## Remove the legacy variable resolver

With the release of Adobe Commerce and Magento Open Source 2.4.4 and 2.4.3-p2, `\Magento\Framework\Filter\VariableResolver\LegacyResolver` and `\Magento\Framework\Filter\VariableResolver\StrategyResolver` have been removed and any legacy templates in the database will only be resolved using strict mode. Database templates can be checked using CLI commands to verify
compatibility with strict mode.

## Verify compatibility with strict mode

The following command scans all database email templates overridden using the Admin **Marketing** > Communications > **Email Templates** > **Add New Template** area for potential variable usage compatibility issues.

```bash
bin/magento dev:email:override-compatibility-check
```

To scan email templates overriden using a custom [theme](email.md), please consider using the [Upgrade Compatibility Tool](https://experienceleague.adobe.com/docs/commerce-operations/upgrade-guide/upgrade-compatibility-tool/install.html).

The following command scans newsletter templates for any potential variable usage compatibility issues.

```bash
bin/magento dev:email:newsletter-compatibility-check
```

## Changes to the custom email template workflow

As of Adobe Commerce and Magento Open Source 2.3.4, custom email templates are only allowed to use scalar values for variable data.
Direct calls to methods are no longer allowed.
To be more specific, methods can no longer be called from variables from either the `var` directive or when used as parameters.
For example `{{var order.getEmailCustomerNote()}}` or `{{something myVar=$obj.method()}}` will fail to resolve.

A 'custom email template' is any new template created in the Admin **Marketing** > Communications > **Email Templates** > **Add New Template** area.
Notice in the incorrect example, the `getConfirmationLink()` method is called directly.

-  Old way: `{{var subscriber.getConfirmationLink()}}`
-  New way: `{{var subscriber_data.confirmation_link}}`

Note that spaces should be not be used next to the braces:

-  Correct: `{{var subscriber_data.confirmation_link}}`
-  Incorrect: `{{ var subscriber_data.confirmation_link }}`

We refer to this as 'strict mode' for email templates.
All default templates have been converted to this strict mode.

<InlineAlert variant="info" slots="text"/>

All existing custom email templates will continue to work after upgrading to 2.3.4.
Any new email template created after installing 2.3.4 must be written in strict mode.
As of 2.4.0, the template filter, specifically anything that uses or extends `\Magento\Framework\Filter\Template`, will use strict mode by default.

## Abstraction example

Pre-2.3.4, the New Order email template had a line with a direct method call:

```html
<p class="greeting">{{trans "%customer_name", customer_name=$order.getCustomerName()}}</p>
```

As of 2.3.4, with the method call removed:

```html
<p class="greeting">{{trans "%customer_name", customer_name=$order_data.customer_name}}</p>
```

Below, within the `$transport` block, `customer_name` is defined in the `order_data` object and the method call place there.
This `order_data` object is passed to the view page as a `DataObject` and is referenced in the variable as above.

```php
public function send(Invoice $invoice, $forceSyncMode = false)
{
    $this->identityContainer->setStore($invoice->getStore());
    $invoice->setSendEmail($this->identityContainer->isEnabled());
    if (!$this->globalConfig->getValue('sales_email/general/async_sending') || $forceSyncMode) {
        $order = $invoice->getOrder();
        if ($this->checkIfPartialInvoice($order, $invoice)) {
            $order->setBaseSubtotal((float) $invoice->getBaseSubtotal());
            $order->setBaseTaxAmount((float) $invoice->getBaseTaxAmount());
            $order->setBaseShippingAmount((float) $invoice->getBaseShippingAmount());
        }
        $transport = [
            'order' => $order,
            'order_id' => $order->getId(),
            'invoice' => $invoice,
            'invoice_id' => $invoice->getId(),
            'comment' => $invoice->getCustomerNoteNotify() ? $invoice->getCustomerNote() : '',
            'billing' => $order->getBillingAddress(),
            'payment_html' => $this->getPaymentHtml($order),
            'store' => $order->getStore(),
            'formattedShippingAddress' => $this->getFormattedShippingAddress($order),
            'formattedBillingAddress' => $this->getFormattedBillingAddress($order),
            'order_data' => [
                'customer_name' => $order->getCustomerName(),
                'is_not_virtual' => $order->getIsNotVirtual(),
                'email_customer_note' => $order->getEmailCustomerNote(),
                'frontend_status_label' => $order->getFrontendStatusLabel()
            ]
        ];
        $transportObject = new DataObject($transport);
```

In this example, the `customer.name` is being computed within the [model](https://github.com/magento/magento2/blob/2.4/app/code/Magento/Sales/Model/Order/Email/Sender/InvoiceSender.php) file.
Depending on the particular instance, this data point can be appended within a custom module, directive or any manner of ways.

## Create a custom directive

The above examples show changes to default application files. We do not recommend editing core files as changes may be lost when upgrading.
Instead, if you need to call a method for a custom email template variable, create a custom directive.
In this example, we will create and pass a `lifetime_spend` custom value.

1. Create a class that implements `Magento\Framework\Filter\SimpleDirective\ProcessorInterface`:

   ```php
    declare(strict_types=1);
    namespace GadgetCorp\CustomEmailDirective\Model;
    use Magento\Framework\Filter\SimpleDirective\ProcessorInterface;
    use Magento\Framework\Pricing\PriceCurrencyInterface;
    /**
    * Calculates the lifetime spend of all customers
    */
    class LifetimeSpendDirective implements ProcessorInterface
    {
        /**
        * @var PriceCurrencyInterface
        */
        private $priceCurrency;
        /**
        * @param PriceCurrencyInterface $priceCurrency
        */
        public function __construct(PriceCurrencyInterface $priceCurrency)
        {
            $this->priceCurrency = $priceCurrency;
        }
        /**
        * @inheritDoc
        */
        public function getName(): string
        {
            return 'lifetime_spend';
        }
        /**
        * @inheritDoc
        */
        public function process($value, array $parameters, ?string $html): string
        {
            $shouldBold = !empty($parameters['should_bold']);
            $amount = $this->priceCurrency->getCurrencySymbol() . $this->calculateLifetimeSpend();
            return ($shouldBold ? '<strong>' . $amount . '</strong>' : $amount);
        }
        /**
        * @inheritDoc
        */
        public function getDefaultFilters(): ?array
        {
            // Make sure newlines are converted to <br /> tags by default
            return ['nl2br'];
        }
        /**
        * Calculate the total amount of money spent by all customers for all time
        *
        * @return float
        */
        private function calculateLifetimeSpend(): float
        {
            // Add code here to calculate the lifetime spend
            return 123.45;
        }
    }
   ```

   and save the file to `<Vendor>/<module>/Model`.

1. Add the new directive to the pool by adding this block to `di.xml`.

   ```xml
   <config xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance" xsi:noNamespaceSchemaLocation="urn:magento:framework:ObjectManager/etc/config.xsd">
    <type name="Magento\Framework\Filter\SimpleDirective\ProcessorPool">
        <arguments>
            <argument name="processors" xsi:type="array">
                <item name="lifetime_spend" xsi:type="object">GadgetCorp\CustomEmailDirective\Model\LifetimeSpendDirective</item>
            </argument>
        </arguments>
    </type>
   </config>
   ```

The new variable is now available within the email template as `{{lifetime_spend}}`.
Note in the class above, we also defined the parameter `shouldBold`. We can use that with `{{lifetime_spend should_bold=1}}`.
You may also use multiple filters within a var statement: `{{lifetime_spend should_bold=1 |escape|nl2br}}`.

## Data objects and getUrl

There are a couple of exceptions to strict mode.

One exception is for objects that extend from `DataObject`. These can still be called directly.
Even then, we do not actually call the getter method directly, but rather, resolve which key is needed and call `getData(<keyname>)` instead.

For example, if we have:

```php
$template->setVariables(['customer_data'=>new DataObject('my_key' => 'foo')]);
```

and in the template where we have

```php
{{somedir mydir mydir=$customer_data.getMyKey()}}
```

the directive will resolve to “foo”.

The same is true for `{{directive foo foo=$customer_data.my_key()}}`.
But note that in both cases the DataObject will not have `getMyKey` invoked but rather `getData(‘my_key’)` is invoked instead.

The second exception is for `getUrl`.
Directives that use the format `{{var this.getUrl(params)}}` will still continue to work for now.

## Advanced filtering

As part of this change, we have removed the limit of processing 2 filters per directive.
Now `{{var order_data.shipping_description|filter1|filter2|filter3}}` will work.

## Nested arrays

Getting data from nested arrays is now supported.
For example, if we have:

```php
$template->setVariables(['customer_data'=> ['name' => ['first_name' => 'John']]]);
```

and in the template:

```php
{{mydir test fname=$customer_data.name.first_name}}
```

it will resolve to “John”.

This new syntax also works in combination with the `DataObject` exception.
For example, if we have:

```php
$template->setVariables(['customer_data'=> ['name' => new DataObject('first_name' => 'John')]]);
```

and in the template we have:

```php
{{mydir dir fname=$customer_data.name.first_name}}
```

it will resolve to “John”.
