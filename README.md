# Zammad API Client (PHP)

## API version support
This client supports API version 1.0.

## Installation

### Requirements
The API client needs [composer](https://getcomposer.org/). For installation have a look at its [documentation](https://getcomposer.org/download/).
Additionally, the API client needs PHP 5.6 or newer.

### Integration into your project
Add the following to the "require" section of your project's composer.json file:
```json
"martini/zammad_api_client_php": "1.0.*"
```

### Installing the API client's dependencies
Fetch the API client's code and its dependencies by updating your project's dependencies with composer:
```
$ composer update
```

Once installed, you have to include the generated autoload.php into your project's code:
```php
require_once dirname(__DIR__).'/vendor/autoload.php';
```

## How to use the API client

### The Client object
Your starting point is the `Client` object:
```php
use ZammadAPIClient\Client;
$client = new Client([
    'url'           => 'https://myzammad.com', # URL to your Zammad installation
    'username'      => 'myuser@myzammad.com',  # Username to use for authentication
    'password'      => 'mypassword',           # Password to use for authentication
]);
```
Besides using a combination of `username` and `password`, you can alternatively give an `http_token` or an `oauth2_token`.
**Important:** You have to activate API access in Zammad.

### Fetching a single Resource object
To fetch a `Resource` object by ID, e. g. a ticket with ID 34, use the `Client` object:
```php
use ZammadAPIClient\ResourceType;
$ticket = $client->resource( ResourceType::TICKET )->get(34);
```

`$ticket` now is a `Resource` object which holds the data of the ticket and provides all of the methods for setting/getting specific values (like the title of the ticket) and sending changed values to Zammad to update the ticket.

Note: Once you successfully called `get` on a `Resource` object, you cannot call it again, instead you have to create a new one with `resource`.

### Accessing values of Resource objects
You can access the values of a `Resource` object via its 'value' methods.
```php
$ticket->setValue( 'title', 'My ticket title' );
$title = $ticket->getValue('title');
$all_values = $ticket->getValues();
```

Please note that the API client does not provide checks for nor does it know about the available fields of the `Resource` objects. If you set or get a value of a non-existing field or set an invalid value, Zammad will ignore it or return an error.

So, how can you know which fields are available? Just fetch an existing `Resource` object and have a look at the returned fields. A fresh Zammad system always contains an object with ID 1 for every resource type.

### Updating Resource objects
If you fetched a `Resource` object and changed some values, you have to send your changes to Zammad. You do this with a simple call:
```php
$ticket->save();
```

`save()` will check it itself but if you somehow need to know if a `Resource` object has unsaved changes, you can check it with:
```php
if ( $ticket->isDirty() ) {...}
```

### Creating Resource objects
To create a new `Resource` object, use the following code (example):
```php
use ZammadAPIClient\ResourceType;

$ticket = $client->resource( ResourceType::TICKET );
$ticket-setValue( 'title', 'My new ticket' );
// ...
// Set additional values
// ...
$ticket->save(); // Will create a new ticket object in Zammad
```

### Searching Resource objects
Some types of resources can be searched, pagination is available.
```php
use ZammadAPIClient\ResourceType;

// Fulltext search
$tickets = $client->resource( ResourceType::TICKET )->search('some text');

// Field specific search
$tickets = $client->resource( ResourceType::TICKET )->search('title:My Title');

// Field specific search with more than one field
$tickets = $client->resource( ResourceType::TICKET )->search('title:My Title AND priority_id:1');

// Pagination: Page 1, 25 entries per page
$tickets = $client->resource( ResourceType::TICKET )->search( 'some text', 1, 25 );
```

Note that there is a configurable server-side limit for the number of returned objects (e. g. 500). This limit also applies to the number of entries per page. If you call search() with 1000 entries per page and the server-side limit is set to 500, the server-side limit will be applied.

A successful search (which might have zero results) returns an array of objects (or an empty array). If the result is the original caller object, there was an error (see error handling below).
Therefore, the code for searching should look like the following:

```php
use ZammadAPIClient\ResourceType;

$tickets = $client->resource( ResourceType::TICKET )->search('some text');
if ( !is_array($tickets) ) {
    // Error handling
    print $tickets->getError();
}
else {
    // Do something with $tickets array
}
```

Note: You cannot use a `Resource` object that contains data (either via `get`, `search`, `all` or by setting values on a new object) to execute a search. Use a new `Resource` object instead.


### Fetching 'all' Resource objects
For some types of resources, all available objects can be fetched, pagination is available.

```php
use ZammadAPIClient\ResourceType;

// Fetch all tickets (keep in mind the server-side limit, see 'Searching Resource objects')
$tickets = $client->resource( ResourceType::TICKET )->all();

// Fetch all tickets with pagination (keep in mind the server-side limit, see 'Searching Resource objects'), page 4, 50 entries per page
$tickets = $client->resource( ResourceType::TICKET )->all( 4, 50 );
```
A successful call of `all` (which might have zero results) returns an array of objects (or an empty array). If the result is the original caller object, there was an error (see error handling below).
Therefore, the code to use `all` should look like the following:

```php
use ZammadAPIClient\ResourceType;

$tickets = $client->resource( ResourceType::TICKET )->all( 4, 50 ); // pagination
if ( !is_array($tickets) ) {
    // Error handling
    print $tickets->getError();
}
else {
    // Do something with $tickets array
}
```

Note: You cannot use a `Resource` object that contains data (either via `get`, `search`, `all` or by setting values on a new object) to execute `all`. Use a new `Resource` object instead.

### Deleting a Resource object
To be able to delete a `Resource` object that exists in Zammad, you must first fetch it from Zammad, either via `get`, `all` or `search`.
You can also delete a newly created `Resource` object that has not been sent to Zammad yet. But this should only rarely be necessary because you can simply create a new `Resource` object via the `Client` object.
To delete a `Resource` object, simply call `delete` on it:
```php
$ticket->delete();
```

This clears the object from all data and if possible deletes it in Zammad. The PHP object itself remains. You can reuse it for another `Resource` object or simply drop it.

### Handling Zammad errors
When you access Zammad, you **always** will get a `Resource` object (or an array of such objects) in return, regardless if Zammad returned data or executed your request. In case of errors (e. g. that above ticket with ID 34 does not exist in Zammad), you will get a `Resource` object with a set error which can be checked with the following code:
```php
if ( $ticket->hasError() ) {
    print $ticket->getError();
}
```

If you additionally need more detailed information about connection/request errors, you can access the `Response` object of the `Client` object. It holds the response to the last request that was made.
```php
$last_response = $client->getLastResponse();
```
With this object, you can e. g. get the HTTP status code and the body of the last response.

## Available resource types and their access methods

To be able to use the 'short form' for the resource type, add a
```php
use ZammadAPIClient\ResourceType;
```

to your code. You then can reference the resource type like
```php
$client->resource( ResourceType::TICKET );
```

|Resource type|get|all|search|save|delete|
|-------------|:-:|:-:|:----:|:--:|:----:|
| TICKET|&#10004;|&#10004;|&#10004;|&#10004;|&#10004;|
| TICKET_ARTICLE|&#10004;|&ndash;|&#10004;|&#10004;|&#10004;|
| TICKET_STATE|&#10004;|&#10004;|&ndash;|&#10004;|&#10004;|
| TICKET_PRIORITY|&#10004;|&#10004;|&ndash;|&#10004;|&#10004;|
| ORGANIZATION|&#10004;|&#10004;|&#10004;|&#10004;|&#10004;|
| GROUP|&#10004;|&#10004;|&ndash;|&#10004;|&#10004;|
| USER|&#10004;|&#10004;|&#10004;|&#10004;|&#10004;|