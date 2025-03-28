# Asana PHP API

[![Build Status][github-actions-image]][github-actions-url]
[![Packagist Version][packagist-image]][packagist-url]

> [!CAUTION]
> The Asana PHP SDK is no longer supported. While it remains available on [Packagist](https://packagist.org/packages/asana/asana), it will not receive bug fixes or enhancements (including new endpoints or data model changes). We recommend using PHP's [cURL](https://www.php.net/manual/en/book.curl.php) or similar HTTP libraries to make direct API requests instead.

Official PHP client library for the Asana API v1

## Installation

### Composer

If you use [Composer](https://getcomposer.org/) to manage dependencies you can include the "asana/asana" package as a depedency.
```json
{
    "require": {
        "asana/asana": "^1.0.7"
    }
}
```

Alternatively you can specify the version as `dev-master` to get the latest master branch in GitHub.

### Local

If you have downloaded this repository to the "php-asana" directory, for example, you can run `composer install` within "php-asana" then include the following lines at the top of a PHP file (in the same directory) to begin using it:
```php
<?php
require 'php-asana/vendor/autoload.php';
```

## Test

After running `composer install` run the tests using:
```shell
./vendor/bin/phpunit --configuration tests/phpunit.xml
```

You can also run the phpcs linter:
```shell
./vendor/bin/phpcs --standard=PSR2 --extensions=php src tests
```

## Authentication

### Personal Access Token

Create a client using a personal access token:
```php
<?php
$client = Asana\Client::accessToken('ASANA_PERSONAL_ACCESS_TOKEN');
```

### OAuth 2

Asana supports OAuth 2. `asana` handles some of the details of the OAuth flow for you.

Create a client using your OAuth Client ID and secret:
```php
<?php
$client = Asana\Client::oauth(array(
    'client_id'     => 'ASANA_CLIENT_ID',
    'client_secret' => 'ASANA_CLIENT_SECRET',
    'redirect_uri'  => 'https://yourapp.com/auth/asana/callback',
));
```

Redirect the user to the authorization URL obtained from the client's `session` object:
```php
<?php
$url = $client->dispatcher->authorizationUrl();
```

`authorizationUrl` takes an optional `state` parameter, passed by reference, which will be set to a random number if null, or passed through if not null:
```php
<?php
$state = null;
$url = $client->dispatcher->authorizationUrl($state);
// $state will be a random number
```

Or:
```php
<?php
$state = 'foo';
$url = $client->dispatcher->authorizationUrl($state);
// $state will still be foo
```

When the user is redirected back to your callback, check the `state` URL parameter matches, then pass the `code` parameter to obtain a bearer token:
```php
<?php
if ($_GET['state'] == $state) {
  $token = $client->dispatcher->fetchToken($_GET['code']);
  // ...
} else {
  // error! possible CSRF attack
}
```

For webservers, it is common practice to store the `state` in a secure-only, http-only cookie so that it will automatically be sent by the browser in the callback.

Note: if you're writing a non-browser-based application (e.x. a command line tool) you can use the special redirect URI `urn:ietf:wg:oauth:2.0:oob` to prompt the user to copy and paste the code into the application.

## Usage

The client's methods are divided into several resources: `attachments`, `events`, `projects`, `stories`, `tags`, `tasks`, `teams`, `users`, and `workspaces`.

Methods that return a single object return that object directly:
```php
<?php
$me = $client->users->getUser("me");
echo "Hello " . $me->name;

$workspaceGid = $me->workspaces[0]->gid;
$project = $client->projects->createProjectForWorkspace($workspaceGid, array('name' => 'new project'));
echo "Created project with gid: " . $project->gid;
```

Methods that return multiple items (e.x. `getTasks`, `getProjects`, `getPortfolios`, etc.) return an items iterator by default. See the "Collections" section

## Options

Various options can be set globally on the `Client.DEFAULTS` object, per-client on `client.options`, or per-request as additional named arguments. For example:
```php
<?php
// global:
Asana\Client::$DEFAULTS['page_size'] = 1000;

// per-client:
$client->options['page_size'] = 1000;

// per-request:
$client->tasks->getTasks(array('project' => 1234), array('page_size' => 1000));
```

### Available options

* `base_url` (default: "https://app.asana.com/api/1.0"): API endpoint base URL to connect to
* `max_retries` (default: 5): number to times to retry if API rate limit is reached or a server error occures. Rate limit retries delay until the rate limit expires, server errors exponentially backoff starting with a 1 second delay.
* `full_payload` (default: false): return the entire JSON response instead of the 'data' propery (default for collection methods and `events.get`)
* `fields` and `expand`: array of field names to include in the response, or sub-objects to expand in the response. For example `array('fields' => array('followers', 'assignee'))`. See [API documentation](https://asana.com/developers/documentation/getting-started/input-output-options)

Collections (methods returning an array as it's 'data' property):

* `iterator_type` (default: "items"): specifies which type of iterator (or not) to return. Valid values are "items" and `null`.
* `item_limit` (default: null): limits the total number of items of a collection to return (spanning multiple requests in the case of an iterator).
* `page_size` (default: 50): limits the number of items per page to fetch at a time.
* `offset`: offset token returned by previous calls to the same method (in `response->next_page->offset`)

Events:

* `poll_interval` (default: 5): polling interval for getting new events via `events->getNext` and `events->getIterator`
* `sync`: sync token returned by previous calls to `events->get` (in `response->sync`)

## Asana Change Warnings
You will receive warning logs if performing requests that may be affected by a deprecation. The warning contains a link that explains the deprecation.

If you receive one of these warnings, you should:

Read about the deprecation.
Resolve sections of your code that would be affected by the deprecation.
Add the deprecation flag to your "asana-enable" header.
You can place it on the client for all requests, or place it on a single request.

    $client = Asana\Client::accessToken('ASANA_PERSONAL_ACCESS_TOKEN', 
        array('headers' => array('asana-disable' => 'string_ids')))
    
or

    $client = Asana\Client::accessToken('ASANA_PERSONAL_ACCESS_TOKEN', 
        array('headers' => array('asana-enable' => 'string_ids,new_sections')))

If you would rather suppress these warnings, you can set
    
    $client = Asana\Client::accessToken('ASANA_PERSONAL_ACCESS_TOKEN', 
        array('log_asana_change_warnings' => false))

## Collections

### Items Iterator

By default, methods that return a collection of objects return an item iterator:
```php
<?php
$workspaces = $client->workspaces->getWorkspaces();
foreach ($workspaces as $workspace) {
    var_dump($workspace);
}
```

Internally the iterator may make multiple HTTP requests, with the number of requested results per page being controlled by the `page_size` option.

### Raw API

You can also use the raw API to fetch a page at a time:
```php
<?php
$offset = null;
while (true) {
    $page = $client->workspaces->getWorkspaces(null, array('offset' => $offset, 'iterator_type' => null, 'page_size' => 2));
    var_dump($page);
    if (isset($page->next_page)) {
        $offset = $page->next_page->offset;
    } else {
        break;
    }
}
```

## Contributing

Feel free to fork and submit pull requests for the code! Please follow the existing code as an example of style and make sure that all your code passes lint and tests.

To develop:

* `git clone git@github.com:Asana/php-asana.git`
* `composer install`
* `phpunit --configuration tests/phpunit.xml`

### Code generation

The specific Asana resource classes in the Gen folder (`Tag`, `Workspace`, `Task`, etc) are
generated code, hence they shouldn't be modified by hand.

### Deployment

**Repo Owners Only.** Take the following steps to issue a new release of the library.

1. Merge in the desired changes into the `master` branch and commit them.
2. Clone the repo, work on master.
3. Bump the package version in the `VERSION` file to indicate the [semantic version](http://semver.org/) change.
4. Update the `README.md` package depedency version in the "Installation" section
5. Commit the change.
6. Tag the commit with `v` plus the same version number you set in the file. `git tag v1.2.3`
7. Push changes to origin, including tags: `git push origin master --tags`
8. Log into [packagist.org](https://packagist.org/login/) and click on the update button

The rest is automatically done by Composer / Packagist. Visit [the asana package](https://packagist.org/packages/asana/asana) to verify the package was published.

NOTE: If the package did not update on Packagist, log into [Packagist](https://packagist.org/packages/asana/asana) and click on the update button to manually update the package

[github-actions-url]: https://github.com/Asana/php-asana/actions
[github-actions-image]: https://github.com/Asana/php-asana/workflows/Build/badge.svg

[packagist-url]: https://packagist.org/packages/asana/asana
[packagist-image]: https://img.shields.io/packagist/v/asana/asana.svg?style=flat-square

[asana-docs]: https://developers.asana.com/docs
