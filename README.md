# Meta Pixel integration for Laravel

[![Latest Version on Packagist](https://img.shields.io/packagist/v/combindma/laravel-facebook-pixel.svg?style=flat-square)](https://packagist.org/packages/combindma/laravel-facebook-pixel)
[![GitHub Tests Action Status](https://img.shields.io/github/actions/workflow/status/combindma/laravel-facebook-pixel/run-tests.yml?branch=main&label=tests&style=flat-square)](https://github.com/combindma/laravel-facebook-pixel/actions?query=workflow%3ATests+branch%3Amaster)
[![GitHub Code Style Action Status](https://img.shields.io/github/actions/workflow/status/combindma/laravel-facebook-pixel/fix-php-code-style-issues.yml?branch=main&label=code%20style&style=flat-square)](https://github.com/combindma/laravel-facebook-pixel/actions?query=workflow%3A"Check+%26+fix+styling"+branch%3Amaster)
[![Total Downloads](https://img.shields.io/packagist/dt/combindma/laravel-facebook-pixel.svg?style=flat-square)](https://packagist.org/packages/combindma/laravel-facebook-pixel)

A Complete Meta pixel implementation for your Laravel application.

## About Combind Agency

[Combine Agency](https://combind.ma?utm_source=github&utm_medium=banner&utm_campaign=package_name) is a leading web development agency specializing in building innovative and high-performance web applications using modern technologies. Our experienced team of developers, designers, and project managers is dedicated to providing top-notch services tailored to the unique needs of our clients.

If you need assistance with your next project or would like to discuss a custom solution, please feel free to [contact us](mailto:hello@combind.ma) or visit our [website](https://combind.ma?utm_source=github&utm_medium=banner&utm_campaign=package_name) for more information about our services. Let's build something amazing together!


## Introduction

This package provides a smooth integration of Meta Pixel, along with a straightforward implementation of the latest Conversions API, enhancing your overall experience.
- [Documentation for version 4.0 - older version](README-v4.md)

## Upgrading to Version 5

If you are upgrading from version 4, please refer to our [Version 5 Migration Guide](upgrade-v4-to-v5.md) for detailed instructions on how to make a smooth transition.


## Pre-requisites

### Register a Meta Pixel

To get started with the Meta pixel, you must have a pixel registered: [Read this guide](https://web.facebook.com/business/help/952192354843755).

### Conversions API

If you plan to use Conversions API then you need to:

#### Obtain An Access Token
To use the Conversions API, you need to generate an access token, which will be passed as a parameter in every API call.

Refer to [Conversions API Guide](https://developers.facebook.com/docs/marketing-api/conversions-api/get-started) to learn more.


## Installation

You can install the package via composer:

```bash
composer require combindma/laravel-facebook-pixel
```

Optionally, you can publish the views using:

```bash
php artisan vendor:publish --tag="meta-pixel-views"
```

You can publish the config file with:

```bash
php artisan vendor:publish --tag="meta-pixel-config"
```

This is the contents of the published config file:

```php
return [
   /*
     * The Meta pixel id, should be a code that looks something like "1202417153106158".
     */
    'pixel_id' => env('META_PIXEL_ID', ''),

    /*
     * The key under which data is saved to the session with flash.
     */
    'session_key' => env('META_PIXEL_SESSION_KEY', config('app.name').'_metaPixel'),

    /*
     * Only if you plan using Conversions API for server events
     * To use the Conversions API, you need an access token. For Documentation please see: https://developers.facebook.com/docs/marketing-api/conversions-api/get-started
     */
    'token' => env('META_PIXEL_TOKEN', ''),

    /*
     * Enable or disable advanced matching. Useful for adjusting user privacy.
     */
    'advanced_matching_enabled' => env('META_PIXEL_ADVANCED_MATCHING_ENABLED', true),

    /*
     * Enable or disable script rendering. Useful for local development.
     */
    'enabled' => env('META_PIXEL_ENABLED', false),
    
    /*
     * Enable or disable logging. Useful for debugging.
     */
    'logging' => env('META_PIXEL_LOGGING', false),

    /*
     * This is used to test server events
     */
    'test_event_code' => env('META_TEST_EVENT_CODE'),
];
```

If you plan on using the [flash-functionality](#flashing-data-for-the-next-request) you must register the MetaPixelMiddleware:

Global Middleware:
```php
->withMiddleware(function (Middleware $middleware) {
     $middleware->append(MetaPixelMiddleware::class);
})
``` 
Or Assigning Middleware to Routes (recommended if you have admin routes):
```php
Route::group(['middleware' => [MetaPixelMiddleware::class]], static function () {

});
``` 

## Usage - Meta Pixel

### Include scripts in Blade

Insert head view after opening head tag, and body view after opening body tag

```html
<!DOCTYPE html>
<html>
<head>
    <x-metapixel-head/>
</head>
<body>
    <x-metapixel-body/>
</body>
```

If you use UUID as user id you should add userIdAsString attribute to the head component:
```html
<head>
    <x-metapixel-head :userIdAsString="true"/>
</head>
```

To add an event, use the `track()` function.

```php
// CheckoutController.php
use Combindma\FacebookPixel\Facades\MetaPixel;

public function index()
{
    MetaPixel::track('Purchase', ['currency' => 'USD', 'value' => 30.00]);
    return view('thank-you');
}
```

This renders:

```html
<html>
  <head>
    <script>/* Meta pixel's base script */</script>
    <!-- ... -->
  </head>
  <body>
  <script>fbq('track', 'Purchase', {"currency":"USD","value":30});</script>
  <!-- ... -->
</html>
```

You can also specify a unique event ID for any of your events so that, if you plan using the conversions API you avoid duplications.

```php
//For example your order id
MetaPixel::track('Purchase', ['currency' => 'USD', 'value' => 30.00], '123456');

//Or create a unique id using
$eventId = uniqid('ViewContent_', true);
MetaPixel::track('ViewContent', [], $eventId);
```


#### Flashing data for the next request

The package can also set event to render on the next request. This is useful for setting data after an internal redirect.

```php
// ContactController.php
use Combindma\FacebookPixel\Facades\MetaPixel;

public function postContact()
{
    // Do contact form stuff...
    MetaPixel::flashEvent('Lead', [
        'content_name' => 'Auto Insurance',
        'content_category' => 'Quote',
        'value' => 400.00,
        'currency' => 'USD'
    ]);
    return redirect()->action('ContactController@getContact');
}
```

After a form submit, the following event will be parsed on the contact page:

```html
<html>
<head>
    <script>/* Meta pixel's base script */</script>
    <!-- ... -->
</head>
<body>
<script>
    fbq(
        'track', 'Lead', {
            'content_name': 'Auto Insurance',
            'content_category': 'Quote',
            'value': 40.00,
            'currency': 'USD'
        }
    );
</script>
<!-- ... -->
</html>
```

### Available Methods

```php
use Combindma\FacebookPixel\Facades\MetaPixel;

// Retrieve your Pixel id
$id = MetaPixel::pixelId();
// Set Pixel id on the fly
MetaPixel::setPixelId('XXXXXXXX');
// Set Pixel token on the fly
MetaPixel::setToken('XXXXXXXX');
// Set Pixel testEventCode on the fly
MetaPixel::setTestEventCode('XXXXXXXX');
// Check whether script rendering is enabled
$enabled = MetaPixel::isEnabled(); // true|false
// Enable and disable script rendering on the fly
MetaPixel::enable();
MetaPixel::disable();
// Add event to the event layer (automatically renders right before the pixel script). Setting new values merges them with the previous ones.
MetaPixel::track('eventName', ['attribute' => 'value']);
MetaPixel::track('eventName', ['attribute' => 'value'], 'event_id'); //with an event id
MetaPixel::track('eventName'); //without properties 
MetaPixel::track('eventName', [], 'event_id'); //with an event id but without properties
// Flash event for the next request. Setting new values merges them with the previous ones.
MetaPixel::flashEvent('eventName', ['attribute' => 'value']);
MetaPixel::flashEvent('eventName', ['attribute' => 'value'], 'event_id'); //with an event id
MetaPixel::flashEvent('eventName'); //without properties
//Clear the event layer.
MetaPixel::clear();
```

### Custom Events

You can also track a specific custom event on your website. This feature is not available for flashed events.

```php
use Combindma\FacebookPixel\Facades\MetaPixel;

// In your controller
MetaPixel::trackCustom('CUSTOM-EVENT-NAME', ['custom_parameter' => 'ABC', 'value' => 10.00, 'currency' => 'USD']);
//With an event ID
MetaPixel::trackCustom('CUSTOM-EVENT-NAME', ['custom_parameter' => 'ABC', 'value' => 10.00, 'currency' => 'USD'], 'EVENT_ID');
```

This renders:

```html
<html>
  <head>
    <script>/* Meta Pixel's base script */</script>
    <!-- ... -->
  </head>
  <body>
  <script>
      fbq('trackCustom', 'CUSTOM-EVENT-NAME', {'custom_parameter': 'ABC', 'value': 10.00, 'currency': 'USD'});
      /* If you specify the event ID */
      fbq('trackCustom', 'CUSTOM-EVENT-NAME', {'custom_parameter': 'ABC', 'value': 10.00, 'currency': 'USD'}, { eventID : 'EVENT_ID' });
  </script>
  <!-- ... -->
</html>
```

### Advanced matching

This package provides by default advanced matching. We retrieve the email and the user id from authenticated user and include it in the Pixel base code fbq('init') function call as a third parameter.
You can disable it by adding META_PIXEL_ADVANCED_MATCHING_ENABLED=false in your .env file

```html
<html>
<head>
    <script>
        /* Meta pixel's base script */
        <!-- ... -->
        fbq('init', '{PixelID}', {
            em: 'email@email.com', //Email provided with Auth::user()->email
            external_id: 12345, //User id provided with Auth::id()
        });
    </script>
    <!-- ... -->
</head>
<body>
<!-- ... -->
</html>
```

### Macroable

Adding events to pages can become a repetitive process. Since this package isn't supposed to be opinionated on what your events should look like, the MetaPixel is macroable.

```php
use Combindma\FacebookPixel\Facades\MetaPixel;

//include this in your macrobale file
MetaPixel::macro('purchase', function ($product) {
    MetaPixel::track('Purchase', [
        'currency' => 'EUR',
        'value' => $product->price
    ]);
});

//in your controller
MetaPixel::purchase($product);
```


## Usage - Conversions API

If you plan on using [Conversions API](https://developers.facebook.com/docs/marketing-api/conversions-api/get-started) functionalities. Yous should specify the token in your .env file first.

For every request yous should specify a unique event id for handling Pixel Duplicate Events and Conversions API.

This is how you can start:

```php
use Combindma\FacebookPixel\Facades\MetaPixel;
use FacebookAds\Object\ServerSide\Content;
use FacebookAds\Object\ServerSide\CustomData;
use FacebookAds\Object\ServerSide\DeliveryCategory;
use FacebookAds\Object\ServerSide\UserData;

//Prepare User Data first.
// By default, the IP Address, User Agent and fbc/fbp cookies are added.
$user_data = MetaPixel::userData()->setEmail('joe@eg.com')->setPhone('12345678901');

$content = (new Content())
    ->setProductId('product123')
    ->setQuantity(1)
    ->setDeliveryCategory(DeliveryCategory::HOME_DELIVERY);
    
$custom_data = (new CustomData())
    ->setContents(array($content))
    ->setCurrency('usd')
    ->setValue(123.45);
    
$eventId = uniqid('prefix_');
    
//send request
MetaPixel::send('Purchase', $eventId ,$custom_data, $user_data);
```

If you don't specify the $user_data parameter, by default we retrieve the email & the id from Auth::user() if the user is authenticated.
We use the user id as a same external_id in Meta Pixel and conversions API

```php
MetaPixel::send('Purchase', $eventId, $custom_data);
```

If you want to test server events, you need to specify the FACEBOOK_TEST_EVENT_CODE in your .env file. By default, this test code will be sent in all API request. 

So Don't forget to delete after you finish your server tests.

You can use the [Playload Helper](https://developers.facebook.com/docs/marketing-api/conversions-api/payload-helper) to learn more about the requests to send.

## Testing

```bash
composer test
```

## Contributing

Please see [CONTRIBUTING](CONTRIBUTING.md) for details.

## Security Vulnerabilities

Please review [our security policy](../../security/policy) on how to report security vulnerabilities.

## Credits

- [Combind](https://github.com/Combindma)
- [All Contributors](../../contributors)

## License

The MIT License (MIT). Please see [License File](LICENSE.md) for more information.
