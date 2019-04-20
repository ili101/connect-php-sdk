![Connect PHP SDK](./assets/connect-logo.png)

# Connect PHP SDK

[![Build Status](https://travis-ci.com/ingrammicro/connect-php-sdk.svg?branch=master)](https://travis-ci.com/ingrammicro/connect-php-sdk) [![Latest Stable Version](https://poser.pugx.org/apsconnect/connect-sdk/v/stable)](https://packagist.org/packages/apsconnect/connect-sdk) [![License](https://poser.pugx.org/apsconnect/connect-sdk/license)](https://packagist.org/packages/apsconnect/connect-sdk) [![codecov](https://codecov.io/gh/ingrammicro/connect-php-sdk/branch/master/graph/badge.svg)](https://codecov.io/gh/ingrammicro/connect-php-sdk)
[![PHP Version](https://img.shields.io/packagist/php-v/apsconnect/connect-sdk.svg?style=flat&branch=master)](https://packagist.org/packages/apsconnect/connect-sdk)
[![PHP Eye](https://img.shields.io/php-eye/apsconnect/connect-sdk.svg?style=flat&branch=master&label=PHP-Eye%20tested)](https://php-eye.com/package/apsconnect/connect-sdk)

## Getting Started
Connect PHP SDK allows an easy and fast integration with [Connect](http://connect.cloud.im/) fulfillment API and usage API. Thanks to it you can automate the fulfillment of orders generated by your products and report usage for it.

In order to use this library, please ensure that you have read first the documentation available on Connect knowladge base article located [here](http://help.vendor.connect.cloud.im/support/solutions/articles/43000030735-fulfillment-management-module), this one will provide you a great information on the rest api that this library implements.

## Class Features

This library may be consumed in your project in order to automate the fulfillment of requests and perform the usage reporting, this class once imported into your project will allow you to:

- Connect to Connect using your api credentials
- List all requests, and even filter them:
    - for a concrete product
    - for a concrete status
    - for a concrete asset
    - etc..
- Process each request and obtain full details of the request
- Modify for each request the activation parameters in order to:
    - Inquiry for changes
    - Store information into the fulfillment request
- Change the status of the requests from it's initial pending state to either inquiring, failed or approved.
- Generate and upload usage files to report usage for active contracts and listings
- Process usage file status changes
- Generate logs
- Collect debug logs in case of failure

Your code may use any scheduler to execute, from a simple cron to a cloud scheduler like the ones available in Azure, Google, Amazon or other cloud platforms.

## Installation & loading
Connect PHP SDK is available on [Packagist](https://packagist.org/packages/apsconnect/connect-sdk) (using semantic versioning), and installation via [Composer](https://getcomposer.org) is the recommended way to install Connect PHP SDK. Just add this line to your `composer.json` file:

```json
{
  "require": {
    "apsconnect/connect-sdk": "^15.0"
    }
}
```

or run

```sh
composer require apsconnect/connect-sdk --no-dev --prefer-dist --classmap-authoritative
```

Note that the `vendor` folder and the `vendor/autoload.php` script are generated by Composer

## A Simple Example of fulfillment

```php
<?php

require_once "vendor/autoload.php";

class ProductRequests extends \Connect\FulfillmentAutomation
{
    
    public function processRequest($request)
    {
        $this->logger->info("Processing Request: " . $request->id . " for asset: " . $request->asset->id);
        switch ($request->type) {
            case "purchase":
                if($request->asset->params['email']->value == ""){
                    throw new \Connect\Inquire(array(
                        $request->asset->params['email']->error("Email address has not been provided, please provide one")
                    ));
                }
                foreach ($request->asset->items as $item) {
                    if ($item->quantity > 1000000) {
                        $this->logger->info("Is Not possible to purchase product " . $item->id . " more than 1000000 time, requested: " . $item->quantity);
                        throw new \Connect\Fail("Is Not possible to purchase product " . $item->id . " more than 1000000 time, requested: " . $item->quantity);
                    }
                    else {
                        //Do some provisoning operation
                        //Update the parameters to store data
                        $paramsUpdate[] = new \Connect\Param('ActivationKey', 'somevalue');
                        //We may use a template defined on vendor portal as activation response, this will be what customer sees on panel
                        return new \Connect\ActivationTemplateResponse("TL-497-535-242");
                        // We may use arbitrary output to be returned as approval, this will be seen on customer panel. Please see that output must be in markup format
                        return new \Connect\ActivationTileResponse('\n# Welcome to Fallball!\n\nYes, you decided to have an account in our amazing service!\n\n');
                        // If we return empty, is approved with default message
                        return;
                    }
                }
            case "cancel":
                //Handle cancellation request
            case "change":
                //Handle change request
            default:
                throw new \Connect\Fail("Operation not supported:".$request->type);
        }
    }

    public function processTierConfigRequest($tierConfigRequest){
        //This method allows processing Tier Requests, in same manner as simple requests.
        // Is required to be implemented since v15
    }
}

//Main Code Block

try {
    
    $requests = new ProductRequests(new \Connect\Config([
        'apiKey' => 'Key_Available_in_ui',
        'apiEndpoint' => 'https://api.connect.cloud.im/public/v1',
        'products' => 'CN-631-322-641' #Optional value
    ]));
    
    $requests->process();
    
} catch (Exception $e) {
    
    print "Error processing requests:" . $e->getMessage();
}
```

## A Simple Example of reporting Usage Files

```php
<?php

require_once "vendor/autoload.php";

class UploadUsage extends \Connect\UsageAutomation
{

    public function processUsageForListing($listing)
    {
        //Detect concrete Provider Contract
        if($listing->contract->id === 'CRD-41560-05399-123') {
            //This is for Provider XYZ, also can be seen from $listing->provider->id and parametrized further via marketplace available at $listing->marketplace->id
            date_default_timezone_set('UTC'); //reporting must be always based on UTC
            $usages = [];
            array_push($usages, new Connect\Usage\FileUsageRecord([
                'record_id' => 'unique record value',
                'item_search_criteria' => 'item.mpn', //Possible values are item.mpn or item.local_id
                'item_search_value' => 'SKUA', //Value defined as MPN on vendor portal
                'quantity' => 1, //Quantity to be reported
                'start_time_utc' => date('d-m-Y H:i:s', strtotime("-1 days")), //From when to report
                'end_time_utc' => date("Y-m-d H:i:s"), //Till when to report
                'asset_search_criteria' => 'parameter.param_b', //How to find the asset on Connect, typical use case is to use a parameter provided by vendor, in this case called param_b, additionally can be used asset.id in case you want to use Connect identifiers
                'asset_search_value' => 'tenant2'
            ]));
            $usageFile = new \Connect\Usage\File([
                'name' => 'sdk test',
                'product' => new \Connect\Product(
                    ['id' => $listing->product->id]
                ),
                'contract' => new \Connect\Contract(
                    ['id' => $listing->contract->id]
                )
            ]);
            $this->submitUsage($usageFile, $usages);
            return "processing done"
        }
        else{
            //Do Something different
        }
    }

}

//Main Code Block

try {
    
    $requests = new UploadUsage();
    $requests->process();

} catch (Exception $e) {
    print "Error processing usage for active listing requests:" . $e->getMessage();
}
```

## A Simple Example of automating workflow of Usage Files

```php
<?php

require_once "vendor/autoload.php";

class UsageFilesWorkflow extends \Connect\UsageFileAutomation
{
    
    public function processUsageFiles($usageFile)
    {
        switch ($usageFile->status){
            case 'invalid':
                //vendor and provider may handle invalid cases different, probably notifying their staff
                throw new \Connect\Usage\Delete("Not needed anymore");
                break;
            case 'ready':
                //Vendor may move to file to provider
                throw new \Connect\Usage\Submit("Ready for Provider");
            case 'pending':
                //Provider use case, needs to be reviewed and accept it
                throw new \Connect\Usage\Accept("File looks good");
            default:
                throw new \Connect\Usage\Skip("not controled status");
        }
    }
}

//Main Code Block

try {
    
    $usageWorkflow = new UsageFilesWorkflow();
    
    // is possible to ask to process all via parsing true, only applicable for
    // providers who automates own products
    $usageWorkflow->process(); 

} catch (Exception $e) {
    print "Error processing usage for active listing requests:" . $e->getMessage();
}
```