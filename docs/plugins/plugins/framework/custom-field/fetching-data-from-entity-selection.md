# Fetching data from "entity selection" custom field

## Overview

If you set up a custom field with an entity selection in the administration, you may need a data resolver to resolve the ID to an entity object.

## Prerequisites

This guide will not explain how to create custom field in general, so head over to the official guide about [custom field](add-custom-field) to learn this first.

## Fetching data

In this example we assume that we already set up a custom field called `custom_linked_product`, which is assigned to the products entity. The type of the custom field `custom_linked_product` is also a product.

If you now update a product in the administration and select a value for `custom_linked_product` only the `id` of the selected product entity gets store in the custom field.

To resolve the `id` and getting access to the product we have linked here, we can create a `ProductSubscriber` which listens to the `ProductEvents::PRODUCT_LOADED_EVENT`. The event will be triggered, when the associated main product will be loaded. So we can easily resolve the id in the custom field.

Lets create a `ProductSubscriber` first which will listen to the `ProductEvents::PRODUCT_LOADED_EVENT`.

<CodeBlock title="<plugin root>/src/Subscriber/ProductSubscriber.php">

```php
<?php declare(strict_types=1);

namespace Swag\BasicExample\Subscriber;

use Shopware\Core\Framework\DataAbstractionLayer\Event\EntityLoadedEvent;
use Symfony\Component\EventDispatcher\EventSubscriberInterface;
use Shopware\Core\Content\Product\ProductEvents;

class ProductSubscriber implements EventSubscriberInterface
{
    public static function getSubscribedEvents(): array
    {
        return [
            ProductEvents::PRODUCT_LOADED_EVENT => 'onProductLoaded'
        ];
    }

    public function onProductLoaded(EntityLoadedEvent $event): void
    {
    }
}
```

</CodeBlock>

For this subscriber to work we need to register it in the service container via the `services.xml` file:

<CodeBlock title="<plugin root>/src/Resources/config/services.xml">

```xml
<?xml version="1.0" ?>

<container xmlns="http://symfony.com/schema/dic/services"
           xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
           xsi:schemaLocation="http://symfony.com/schema/dic/services http://symfony.com/schema/dic/services/services-1.0.xsd">

    <services>
        <service id="Swag\BasicExample\Subscriber\ProductSubscriber">
            <tag name="kernel.event_subscriber"/>
        </service>
    </services>
</container>
```

</CodeBlock>

Now our `ProductSubscriber` should be called every time a product is loaded, so we can resolve the custom field `custom_linked_product`.

<CodeBlock title="<plugin root>/src/Subscriber/ProductSubscriber.php">

```php
<?php declare(strict_types=1);

namespace Swag\BasicExample\Subscriber;

use Shopware\Core\Framework\DataAbstractionLayer\Event\EntityLoadedEvent;
use Symfony\Component\EventDispatcher\EventSubscriberInterface;
use Shopware\Core\Content\Product\ProductEvents;

class ProductSubscriber implements EventSubscriberInterface
{
    public static function getSubscribedEvents(): array
    {
        return [
            ProductEvents::PRODUCT_LOADED_EVENT => 'onProductLoaded'
        ];
    }

    public function onProductLoaded(EntityLoadedEvent $event): void
    {

        // loop through all loaded product      
        /** @var ProductEntity $productEntity */
        foreach ($event->getEntities() as $productEntity) {
            $customFields = $productEntity->getCustomFields();

            // loop through each product's custom fields
            foreach($customFields as $name => $value) {
                if ($name !== 'custom_linked_product' || empty($value)) {
                    continue;
                }

               // resolve the $value here
            }

            $productEntity->setCustomFields($customFields);
        }
    }
}
```

</CodeBlock>

Inside the `onProductLoaded` method we can get access to the loaded product entities by calling `$event->getEntities()`. Now for every product we look for our `custom_linked_product` custom field.

But, how we can load the linked product by its `id` if the custom field was set? We have to inject the product repository to achieve it.

First we update the `services.xml` and inject the product repository.

<CodeBlock title="<plugin root>/src/Resources/config/services.xml">

```xml
<?xml version="1.0" ?>

<container xmlns="http://symfony.com/schema/dic/services"
           xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
           xsi:schemaLocation="http://symfony.com/schema/dic/services http://symfony.com/schema/dic/services/services-1.0.xsd">

    <services>
        <service id="Swag\BasicExample\Subscriber\ProductSubscriber">
            <argument type="service" id="product.repository"/>
            <tag name="kernel.event_subscriber"/>
        </service>
    </services>
</container>
```

</CodeBlock>

Now we can use the product repository in our subscriber.

<CodeBlock title="<plugin root>/src/Subscriber/ProductSubscriber.php">

```php
<?php declare(strict_types=1);

namespace Swag\BasicExample\Subscriber;

use Shopware\Core\Content\Product\ProductEntity;
use Shopware\Core\Framework\DataAbstractionLayer\EntityRepositoryInterface;
use Shopware\Core\Framework\DataAbstractionLayer\Event\EntityLoadedEvent;
use Shopware\Core\Framework\DataAbstractionLayer\Search\Criteria;
use Symfony\Component\EventDispatcher\EventSubscriberInterface;
use Shopware\Core\Content\Product\ProductEvents;

class ProductSubscriber implements EventSubscriberInterface
{
    private EntityRepositoryInterface $productRepository;

    public function __construct(
        EntityRepositoryInterface $productRepository
    ) {
        $this->productRepository = $productRepository;
    }

   //...
}
```

</CodeBlock>

As you can see, the product repository was injected and is now available to the `ProductRepository`. The last step is to resolve the `custom_linked_product` value inside the `onProductLoaded` method.

Let's have a look at the final implementation of the subscriber.

<CodeBlock title="<plugin root>/src/Subscriber/ProductSubscriber.php">

```php
<?php declare(strict_types=1);

namespace Swag\BasicExample\Subscriber;

use Shopware\Core\Content\Product\ProductEntity;
use Shopware\Core\Framework\DataAbstractionLayer\EntityRepositoryInterface;
use Shopware\Core\Framework\DataAbstractionLayer\Event\EntityLoadedEvent;
use Shopware\Core\Framework\DataAbstractionLayer\Search\Criteria;
use Symfony\Component\EventDispatcher\EventSubscriberInterface;
use Shopware\Core\Content\Product\ProductEvents;

class ProductSubscriber implements EventSubscriberInterface
{
    private EntityRepositoryInterface $productRepository;

    public function __construct(
        EntityRepositoryInterface $productRepository
    ) {
        $this->productRepository = $productRepository;
    }

    public static function getSubscribedEvents(): array
    {
        return [
            ProductEvents::PRODUCT_LOADED_EVENT => 'onProductLoaded'
        ];
    }

    public function onProductLoaded(EntityLoadedEvent $event): void
    {
        // loop through all loaded products
        /** @var ProductEntity $productEntity */
        foreach ($event->getEntities() as $productEntity) {
            $customFields = $productEntity->getCustomFields();

            // loop through each product's custom fields
            foreach($customFields as $name => $value) {
                if ($name !== 'custom_demo_test' || emtpy($value)) {
                    continue;
                }

                $context = $event->getContext();

                // search the entity via the repository here
                /** @var ProductEntity $productEntity */
                $product = $this->productRepository
                    ->search(new Criteria([$value]), $context)->first();

                if($product) {
                    // replace the custom field's value with the actual entity
                    $customFields[$name] = $product;
                }
            }

            $productEntity->setCustomFields($customFields);
        }
    }
}
```

</CodeBlock>
