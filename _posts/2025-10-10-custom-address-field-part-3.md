---
title: "Magento 2: Adding Custom Shipping Address Field Tutorial (Part 3 - Order Placement and Enhancements)"
date: 2025-10-10
tags:
  - Magento 2
  - Checkout
  - Development
---

## Magento 2: Adding Custom Shipping Address Field Tutorial (Part 3 - Order Placement and Enhancements)

### Challenges

We now have our custom attribute data stored in the quote address, but we need to configure Magento to convert this data into the sales order address. Additionally, there are several minor issues to address:

- Attempting to save the address to the address book generates an error
- Creating an account after guest checkout does not handle our custom attribute
- Attempting to place an order as a guest generates an error

### Step 1. Introduce sales_order_address.custom_field Column

Update the `db_schema.xml` file:

```xml
<?xml version="1.0"?>
<schema xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance" xsi:noNamespaceSchemaLocation="urn:magento:framework:Setup/Declaration/Schema/etc/schema.xsd">
    <table name="quote_address">
        <column xsi:type="varchar" name="custom_field" nullable="true" length="255" comment="Custom Address Field"/>
    </table>
    <!-- add new column -->
    <table name="sales_order_address">
        <column xsi:type="varchar" name="custom_field" nullable="true" length="255" comment="Custom Address Field"/>
    </table>
</schema>
```

Run the database upgrade:

```shell
php bin/magento setup:upgrade
```

In theory, there is a class responsible for copying data from `quote_address` to `sales_order_address` called [\Magento\Quote\Model\Quote\Address\ToOrderAddress](https://github.com/magento/magento2/blob/2.4-develop/app/code/Magento/Quote/Model/Quote/Address/ToOrderAddress.php), which utilizes [\Magento\Framework\DataObject\Copy](https://github.com/magento/magento2/blob/2.4-develop/lib/internal/Magento/Framework/DataObject/Copy.php). We need to add copy instructions to the `fieldset.xml` file for `sales_convert_quote_address`.

**File:** `etc/fieldset.xml`

```xml
<?xml version="1.0"?>
<config xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance" xsi:noNamespaceSchemaLocation="urn:magento:framework:Data/etc/fieldset.xsd">
    <scope id="global">
        <fieldset id="sales_convert_quote_address">
            <field name="custom_field">
                <aspect name="to_order_address"/>
            </field>
        </fieldset>
    </scope>
</config>
```

However, this approach does not work as expected, so I added an additional plugin to this class and handled it manually.

**File:** `etc/di.xml`

```xml
<?xml version="1.0"?>
<config xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance" xsi:noNamespaceSchemaLocation="urn:magento:framework:ObjectManager/etc/config.xsd">
    <type name="\Magento\Checkout\Block\Checkout\AttributeMerger">
        <plugin name="sn_fix_data_scope" type="SergiyNezbritskiy\CustomAddressField\Plugin\Checkout\AttributeMergerPlugin"/>
    </type>
    <type name="\Magento\Quote\Model\Quote\Address">
        <plugin name="sn_convert_extension_attributes" type="SergiyNezbritskiy\CustomAddressField\Plugin\Quote\AddressPlugin"/>
    </type>
    <!-- added new plugin -->
    <type name="\Magento\Quote\Model\Quote\Address\ToOrderAddress">
        <plugin name="sn_quote_to_order" type="SergiyNezbritskiy\CustomAddressField\Plugin\QuoteAddressToOrderAddress\ToOrderAddressConverterPlugin"/>
    </type>
</config>
```

**File:** `Plugin/QuoteAddressToOrderAddress/ToOrderAddressConverterPlugin.php`

```php
<?php

declare(strict_types=1);

namespace SergiyNezbritskiy\CustomAddressField\Plugin\QuoteAddressToOrderAddress;

use Magento\Quote\Model\Quote\Address;
use Magento\Quote\Model\Quote\Address\ToOrderAddress;
use Magento\Sales\Api\Data\OrderAddressInterface;

class ToOrderAddressConverterPlugin
{
    /**
     * @see ToOrderAddress::convert
     */
    public function afterConvert(ToOrderAddress $subject, OrderAddressInterface $result, Address $object): OrderAddressInterface
    {
        /** @var \Magento\Sales\Model\Order\Address $result */
        $result->setData('custom_field', $object->getData('custom_field'));
        return $result;
    }
}
```

Clean the cache before proceeding:

```shell
php bin/magento cache:clean
```

#### Verification

At this point, you should be able to see your attribute on the order view page and in the `sales_order_address.custom_field` column.

### Step 2. Fix Saving Address to Address Book as a Registered User

Again, we need to configure the `\Magento\Framework\DataObject\Copy` object.

**File:** `etc/fieldset.xml`

```xml
<?xml version="1.0"?>
<config xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance" xsi:noNamespaceSchemaLocation="urn:magento:framework:Data/etc/fieldset.xsd">
    <scope id="global">
        <fieldset id="sales_convert_quote_address">
            <field name="custom_field">
                <aspect name="to_order_address"/>
                <aspect name="to_customer_address"/><!-- this line has been added -->
            </field>
        </fieldset>
    </scope>
</config>
```

#### Verification

At this point, the option to save the address to the address book should work correctly for both shipping and billing addresses.

### Step 3. Fix Placing Order as a Guest User

Now let's address the guest user scenario. For guest users, the data from quote address to sales order address is copied from custom attributes. Let's add a plugin to set them whenever they are called.

**File:** `Plugin/Quote/AddressPlugin.php`

```php
/**
 * @see \Magento\Quote\Model\Quote\Address::getCustomAttributes
 */
public function afterGetCustomAttributes(\Magento\Quote\Model\Quote\Address $subject, array $customAttributes): array
{
    if (!array_key_exists('custom_field', $customAttributes)) {
        $customAttributes['custom_field'] = new \Magento\Framework\Api\AttributeValue([
            'attribute_code' => 'custom_field',
            'value' => $subject->getData('custom_field'),
        ]);
    }
    return $customAttributes;
}
```

#### Verification

At this point, you should be able to see the success page after placing the order.

![](/assets/images/custom-address-field/09-success-page.png)

### Step 4. Fix Creating Account After Guest Order

The final issue to address is the "Create an Account" button on the success page. Currently, after clicking that button and attempting to register an account, we will receive an error stating that "Custom Field" is a required value.

How does this work? When we click the "Create an Account" button, Magento saves customer data (including address data) into session storage. You can see this in `\Magento\Customer\Model\Delegation\Storage::storeNewOperation`, specifically in the line `$this->session->setCustomerFormData($customerData)`. What actually happens is that Magento serializes the object and then unserializes it when we submit the registration form by calling the non-existent `getDelegatedNewCustomerData` method from the Session object. Our custom data gets lost during serialization. Let's fix this by adding one more plugin.

First, we need to configure Magento (the `\Magento\Framework\DataObject\Copy` object) on how to handle `custom_field` when copying sales order address into customer address.

**File:** `etc/fieldset.xml`

```xml
<?xml version="1.0"?>
<config xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance" xsi:noNamespaceSchemaLocation="urn:magento:framework:Data/etc/fieldset.xsd">
    <scope id="global">
        <fieldset id="sales_convert_quote_address">
            <field name="custom_field">
                <aspect name="to_order_address"/>
                <aspect name="to_customer_address"/>
            </field>
        </fieldset>
        <!-- This section has been added -->
        <fieldset id="order_address">
            <field name="custom_field">
                <aspect name="to_customer_address"/>
            </field>
        </fieldset>
    </scope>
</config>
```

**File:** `etc/di.xml`

```xml
<?xml version="1.0"?>
<config xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance" xsi:noNamespaceSchemaLocation="urn:magento:framework:ObjectManager/etc/config.xsd">
    <type name="\Magento\Checkout\Block\Checkout\AttributeMerger">
        <plugin name="sn_fix_data_scope" type="SergiyNezbritskiy\CustomAddressField\Plugin\Checkout\AttributeMergerPlugin"/>
    </type>
    <type name="\Magento\Quote\Model\Quote\Address">
        <plugin name="sn_convert_extension_attributes" type="SergiyNezbritskiy\CustomAddressField\Plugin\Quote\AddressPlugin"/>
    </type>
    <type name="\Magento\Quote\Model\Quote\Address\ToOrderAddress">
        <plugin name="sn_quote_to_order" type="SergiyNezbritskiy\CustomAddressField\Plugin\QuoteAddressToOrderAddress\ToOrderAddressConverterPlugin"/>
    </type>
    <!-- This section has been added -->
    <type name="\Magento\Customer\Model\Session">
        <plugin name="sn_restore_custom_attributes" type="SergiyNezbritskiy\CustomAddressField\Plugin\Session\OrderToCustomerPlugin"/>
    </type>
</config>

```

**File:** `Plugin/Session/OrderToCustomerPlugin.php`

```php
<?php

declare(strict_types=1);

namespace SergiyNezbritskiy\CustomAddressField\Plugin\Session;

use Magento\Customer\Model\Session;

class OrderToCustomerPlugin
{
    /**
     * @see Session::__call
     */
    public function after__call(Session $subject, mixed $result, string $method): mixed
    {
        if ($method === 'getDelegatedNewCustomerData') {
            if (is_array($result) && array_key_exists('addresses', $result)) {
                foreach ($result['addresses'] as &$address) {
                    $address['custom_attributes']['custom_field'] = $address['custom_field'];
                }
            }
        }
        return $result;
    }
}
```