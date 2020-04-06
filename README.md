OAuth 2.0 Client plugin for Craft CMS 3
===

This plugin provides developers with an easy centralized approach to managing and storing OAuth 2.0
clients and tokens.

It exposes an easy to use API and frontend for authorizing tokens for internal business logic. What it does not do is
act as an authentication provider for users to login to the CMS.

## Features
- Simple API for integrating League OAuth Providers
- Lots of events for developers
- CLI for refreshing tokens
- Project config support
- 1-line Twig helper for generating authentication UI in your module

## Example Use Cases
- Building a custom CRM integration
- Reading from and writing to Google Sheets
- Querying data on a business' Facebook page

## Example Non-Use Cases
- Logging-in users on the frontend
- Allowing users to access the CP via social accounts
- Keeping track of many CMS users' social accounts

## Requirements

This plugin should work on Craft CMS 3.1.34.3 or later

## Installation

To install the plugin, follow these instructions.

1. Open your terminal and go to your Craft project:

        cd /path/to/project

2. Then tell Composer to load the plugin:

        composer require venveo/craft-oauthclient

3. In the Control Panel, go to Settings → Plugins and click the “Install” button for OAuth 2.0 Client.

4. Configure on Craft settings page

---

## Providers

A provider in this context is an OAuth 2.0 server that is exposing an API via token authorization. Out of the box, this
plugin ships with the following providers:
- Google
- Facebook
- GitHub

The plugin utilizes the widely used `oauth2-client` project by thephpleague in order to make adding providers as
painless as possible. We add an additional layer to this abstraction in order to mix in requirements for Craft.

### Creating a Provider

Assuming a League provider already exists for your service, you can easily create your own implementation. In your
module or plugin, create a file for the provider and follow this outline:

```php
<?php
use League\OAuth2\Client\Provider\YOUR_PROVIDER_CLASS as LeagueProvider;
use venveo\oauthclient\base\Provider;
class MyProvider extends Provider
{
    /**
     * @inheritDoc
     */
    public static function displayName(): string
    {
        // This is what is displayed in the CP when registering an App
        return 'My Provider';
    }

    public static function getProviderClass(): string
    {
        // Return the class name for the league provider
        return LeagueProvider::class;
    }
}
```

And now you need only register it in your plugin or module's init function:

```php
```php
<?php
use venveo\oauthclient\services\Providers;
use craft\events\RegisterComponentTypesEvent;
use MyProvider;
// [...]
Event::on(Providers::class, Providers::EVENT_REGISTER_PROVIDER_TYPES, function (RegisterComponentTypesEvent $event) {
    $event->types[] = MyProvider:class
});
```

Once you have registered the provider, it will appear in the provider dropdown list when you click "Register New App" in
the Control Panel.

The example above is the bare-minimum for registering a provider. Should you need to, there are many methods you can
override and implement in order to customize the authorization flow of your provider as needed. 

See `venveo\oauthclient\base\Provider` for information on the possibilities here. 


## Apps

In this plugin, an App represents the implementation of a provider with a registered OAuth provider. Most people will
probably only need one or two, but you can create as many as needed. To register an app, you'll need to start on the
providers website developer options and create an OAuth 2.0 application. At some point, you'll be asked for the redirect
URI of the login flow. This URI is generated by the plugin once an app is registered. You'll need to set it to something
temporary until you've saved the app - at which point you'll need to update it with the provider.

After an app is registered, you should be able to visit the OAuth Apps overview page and click the "+" button to the
right of the app listing to create your first token.

## API Usage

This plugin assumes you're performing your actual logic in your module or plugin.

The plugin has the following services available:

- `venveo\oauthclient\services\Apps` - used for retrieving an Apps and their configuration
- `venveo\oauthclient\services\Credentials` - used for retrieving and managing tokens for apps
- `venveo\oauthclient\services\Providers` - used for managing available providers
- `venveo\oauthclient\services\Tokens` - used for managing tokens

Generally, you'll only find yourself using the `Apps` and `Credentials` services. 

## Controlling Authentication Flow

Often, you'll find yourself needing to tweak the parameters of the authentication process providers
depending on the situation. For example, in some cases, you may want to force Google to prompt
for consent so you can acquire a refresh token on previously authenticated individuals. This may 
also be useful if you'd like to tweak the requested scopes as the user makes their way through the app.

Typically, this might mean writing a lot of repetitive code; however, we've approached this problem
by introducing "contexts" to an authentication process. For example, I might use the Twig helper to render
a connector in my module and want to ensure the user has offline access to Google. I would first render my
connector with the context parameter set and then register an event handler in my module to 
tweak the authentication URL depending on the context.

The context for the connection button on the Applications list in the control panel
is `plugin.cp`

[See Example](#modifying-the-authentication-flow-conditionally)

## Events

#### `venveo\oauthclient\services\Apps`

- `Apps:EVENT_BEFORE_APP_SAVED` 
    - `venveo\oauthclient\events\AppEvent`
- `Apps:EVENT_AFTER_APP_SAVED` 
    - `venveo\oauthclient\events\AppEvent`
- `Apps:EVENT_BEFORE_APP_DELETED` 
    - `venveo\oauthclient\events\AppEvent`
- `Apps:EVENT_AFTER_APP_DELETED` 
    - `venveo\oauthclient\events\AppEvent`
- `Apps:EVENT_GET_URL_OPTIONS` 
    - `venveo\oauthclient\events\AuthorizationUrlEvent`

#### `venveo\oauthclient\services\Tokens`

- `Tokens:EVENT_BEFORE_TOKEN_SAVED` 
    - `venveo\oauthclient\events\TokenEvent`
- `Tokens:EVENT_BEFORE_TOKEN_SAVED` 
    - `venveo\oauthclient\events\TokenEvent`

#### `venveo\oauthclient\services\Credentials`

- `Credentials:EVENT_BEFORE_REFRESH_TOKEN`
    - `venveo\oauthclient\events\TokenEvent`
- `Credentials:EVENT_AFTER_REFRESH_TOKEN` 
    - `venveo\oauthclient\events\TokenEvent`
- `Credentials::EVENT_TOKEN_REFRESH_FAILED`
    - `venveo\oauthclient\events\TokenEvent`

#### `venveo\oauthclient\base\Provider`
- `venveo\oauthclient\base\Provide::EVENT_CREATE_TOKEN_MODEL_FROM_RESPONSE`
    - `venveo\oauthclient\events\TokenEvent`
    
#### `venveo\oauthclient\controllers\AuthorizeController`
- `venveo\oauthclient\controllers\AuthorizeController::EVENT_BEFORE_AUTHENTICATE`
    - `venveo\oauthclient\events\AuthorizationEvent`
- `venveo\oauthclient\controllers\AuthorizeController::EVENT_AFTER_AUTHENTICATE`
    - `venveo\oauthclient\events\AuthorizationEvent`

## Twig Variable

There's a helpful Twig variable, `craft.oauth` exposed by the OAuth Client plugin to help you build your UI.

`craft.oauth.getAppByHandle('handle')` returns an App model if it exists

## Command Line Interface (CLI)

If you would like to refresh all tokens, you can utilize the CLI to automate the process.

` ./craft oauthclient/apps/refresh-tokens <app handle>`

Returns status code 1 if there were errors and 0 if successful

---

## Examples

### Interact with Google Sheets

If you wanted to manage some data in your Google Sheets account, you could easily require the Google_Client composer
package and make the necessary requests; however, token management adds a lot of overhead and complexity. That's where
this plugin comes in. Assuming you've already required the Google_Client, you could utilize this plugin like so:

```php
```php
use venveo\oauthclient\Plugin;
// [...]

// Get the plugin instance. Note: make sure you do this after the application has been inited, such as in a route or
// event.
$plugin = Plugin::$plugin;
// Let's grab a valid token - we could pass the current user ID in here to limit it
$tokens = $plugin->credentials->getValidTokensForAppAndUser('google');
// Get the app from the apps service
$app = $plugin->apps->getAppByHandle('google');

// Show time! Note: you should add some error checking.
$client = new Google_Client();
$client->setAccessToken($tokens[0]->accessToken);
$client->setClientId($app->getClientId());
$client->setClientSecret($app->getClientSecret());

$service = new Google_Service_Sheets($client);
$sheet = $service->spreadsheets->get('some-google-sheet');
```

### Using the Twig variable to check if the current user is connected

```twig
{% set app = craft.oauth.getAppByHandle('google') %}
{% if app %}
    {{ app.name }}
    {% set tokens = app.getValidTokensForUser() %}
    {% if tokens|length %}
        Connected!
    {% else %}
        {# This will render some boilerplate UI to connect the app #}
        {{ app.renderConnector() }}
    {% endif %}
{% else %}
    Could not find app
{% endif %}
```

### Modifying the authentication flow conditionally

#### Example 1: Modifying the OAuth Provider Settings
In this example, we'll render the connector with a context and register an event
to modify the authorization parameters before the connection URL is rendered.

```twig
{% set app = craft.oauth.getAppByHandle('google') %}
{{ app.renderConnector('cp') }}

{# I can also just render the link URL #}
<a href="{{ app.getRedirectUrl('cp') }}">Login</a>
```

```php
use venveo\oauthclient\events\AuthorizationUrlEvent;
use venveo\oauthclient\services\Apps;
use yii\base\Event;
// [...]
Event::on(Apps::class, Apps::EVENT_GET_URL_OPTIONS, function (AuthorizationUrlEvent $e) {
    if ($e->context === 'cp' && $e->app->handle === 'google') {
        // Force re-consent during OAuth 
        $e->options['prompt'] = 'consent';
        $e->options['access_type'] = 'offline';
        $e->options['approval_prompt'] = null;
    }
});
```

#### Example 2: Modifying the OAuth flow based on context

If we wanted to adjust the return URL the user is brought to after authenticating, we have two approaches. The first is quite simple and uses standard Craft form redirects:
```twig
<form method="post" action="{{ app.getRedirectUrl(context) }}">
        {{ csrfInput() }}
        {# In this case, we'll just send the user back to this page #}
        {{ redirectInput(craft.app.request.url) }}
        <button type="submit" class="btn formsubmit">Connect</button>
</form>
```

The 2nd case is more complicated, and that's when we don't have a form, but instead just a button to the `app.getRedirectUrl()`

Since it's not a POST request, we need to make use of `context` to modify the flow. We'll call our context "my-custom-context" and invoke the redirect URL with that context:
```twig
<a href="{{ app.getRedirectUrl('my-custom-context') }}>Login</a>
```

Now in our module or plugin, we can just register an event handler for the AuthorizeController controller:
```php
Event::on(AuthorizeController::class, AuthorizeController::EVENT_BEFORE_AUTHENTICATE, function (AuthorizationEvent $event) {
            if ($event->context === 'my-custom-context') {
                $event->returnUrl = 'https://google.com';
            }
});
```

Success! Now upon logging in, the user will be sent to Google.

Brought to you by [Venveo](https://www.venveo.com)
