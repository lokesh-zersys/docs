# Laravel Pulse

- [Introduction](#introduction)
- [Installation](#installation)
    - [Configuration](#configuration)
- [Dashboard](#dashboard)
    - [Authorization](#dashboard-authorization)
    - [Customization](#dashboard-customization)
    - [Cards](#dashboard-cards)
- [Capturing Entries](#capturing-entries)
    - [Recorders](#recorders)
- [Performance](#performance)
    - [Using a Different Database](#using-a-different-database)
    - [Redis Ingest](#ingest)
    - [Sampling](#sampling)
    - [Trimming](#trimming)
    - [Handling Pulse Exceptions](#pulse-exceptions)

<a name="introduction"></a>
## Introduction

[Laravel Pulse](https://github.com/laravel/pulse) delivers at-a-glance insights into your application's performance and usage. With Pulse, you can track down bottlenecks like slow jobs and endpoints, find your most active users, and more.

For in-depth debugging of individual events, check out [Laravel Telescope](/docs/{{version}}/telescope).

<a name="installation"></a>
## Installation

> **Warning**  
> Pulse's first-party storage implementation currently requires a MySQL database. If you are using a different database engine, such as PostgreSQL, you will need a separate MySQL database for your Pulse data.

Since Pulse is currently in beta, you may need to adjust your application's `composer.json` file to allow beta package releases to be installed:

```json
"minimum-stability": "beta",
"prefer-stable": true
```

Then, you may use the Composer package manager to install Pulse into your Laravel project:

```sh
composer require laravel/pulse
```

After installing Pulse, you should run the `migrate` command in order to create the tables needed to store Pulse's data:

```sh
php artisan migrate
```

Once Pulse's database migrations have been run, you may access the Pulse dashboard via the `/pulse` route.

> **Note**
> If you do not want to store Pulse data in your application's primary database, you may [specify a dedicated database connection](#using-a-different-database).

<a name="configuration"></a>
### Configuration

Many of Pulse's configuration options can be controlled using environment variables. To see the available options, register new recorders, or configure advanced options, you may publish the `config/pulse.php` configuration file:

```sh
php artisan vendor:publish --tag=pulse-config
```

<a name="dashboard"></a>
## Dashboard

<a name="dashboard-authorization"></a>
### Authorization

The Pulse dashboard may be accessed via the `/pulse` route. By default, you will only be able to access this dashboard in the `local` environment, so you will need to configure authorization for your production environments by customizing the `'viewPulse'` authorization gate. You can accomplish this within your application's `app/Providers/AuthServiceProvider.php` file:

```php
use App\Models\User;
use Illuminate\Support\Facades\Gate;

/**
 * Register any authentication / authorization services.
 */
public function boot(): void
{
    Gate::define('viewPulse', function (User $user) {
        return $user->isAdmin();
    });

    // ...
}
```

<a name="dashboard-customization"></a>
### Customization

The Pulse dashboard cards and layout may be configured by publishing the dashboard view. The dashboard view will be published to `resources/views/vendor/pulse/dashboard.blade.php`:

```sh
php artisan vendor:publish --tag=pulse-dashboard
```

The dashboard is powered by [Livewire](https://livewire.laravel.com/), and allows you to customize the cards and layout without needing to rebuild any JavaScript assets.

Within this file, the `<x-pulse>` component is responsible for rendering the dashboard and provides a grid layout for the cards. If you would like the dashboard to span the full width of the screen, you may provide the `full-width` prop to the component:

```blade
<x-pulse full-width>
    ...
</x-pulse>
```

By default, the `<x-pulse>` component will create a 12 column grid, but you may customize this using the `cols` prop:

```blade
<x-pulse cols="16">
    ...
</x-pulse>
```

Each card accepts a `cols` and `rows` prop to control the space and positioning:

```blade
<livewire:pulse.usage cols="4" rows="2" />
```

Most cards also accept an `expand` prop to show the full card instead of scrolling:

```blade
<livewire:pulse.slow-queries expand />
```

<a name="dashboard-cards"></a>
### Cards

<a name="servers-card"></a>
#### Servers

The `<livewire:pulse.servers />` card displays system resource usage for all servers running the `pulse:check` command. Please refer to the documentation regarding the [servers recorder](#servers-recorder) for more information on system resource reporting.

<a name="application-usage-card"></a>
#### Application Usage

The `<livewire:pulse.usage />` card displays the top 10 users making requests to your application, dispatching jobs, and experiencing slow requests.

If you wish to view all usage metrics on screen at the same time, you may include the card multiple times and specify the `type` attribute:

```blade
<livewire:pulse.usage type="requests" />
<livewire:pulse.usage type="slow_requests" />
<livewire:pulse.usage type="jobs" />
```

By default, Pulse will resolve the `name` and `email` fields from the `User` model and display avatars using the Gravatar web service. However, you may customize the user resolution and display by invoking the `Pulse::users` method within application's `App\Providers\AppServiceProvider` class.

The `users` method accepts a closure which will receive the user IDs to be displayed and should return an array or collection containing the `id`, `name`, `extra`, and `avatar` for each user ID:

```php
use Laravel\Pulse\Facades\Pulse;

Pulse::users(function ($ids) {
    return User::findMany($ids)->map(fn ($user) => [
        'id' => $user->id,
        'name' => $user->name,
        'extra' => $user->email,
        'avatar' => $user->avatar_url,
    ]);
});
```

> **Note**
> If your application receives a lot of requests or dispatches a lot of jobs, you may wish to enable [sampling](#sampling). See the [user requests recorder](#user-requests-recorder), [user jobs recorder](#user-jobs-recorder), and [slow jobs recorder](#slow-jobs-recorder) documentation for more information.

<a name="exceptions-card"></a>
#### Exceptions

The `<livewire:pulse.exceptions />` card shows the frequency and recency of exceptions occurring in your application. By default, exceptions are grouped based on the exception class and location where it occurred. See the [exceptions recorder](#exceptions-recorder) documentation for more information.

<a name="queues-card"></a>
#### Queues

The `<livewire:pulse.queues />` card shows the throughput of the queues in your application, including the number of jobs queued, processing, processed, released, and failed. See the [queues recorder](#queues-recorder) documentation for more information.

<a name="slow-requests-card"></a>
#### Slow Requests

The `<livewire:pulse.slow-requests />` card shows incoming requests to your application that exceed the configured threshold, which is 1,000ms by default. See the [slow requests recorder](#slow-requests-recorder) documentation for more information.

<a name="slow-jobs-card"></a>
#### Slow Jobs

The `<livewire:pulse.slow-jobs />` card shows the queued jobs in your application that exceed the configured threshold, which is 1,000ms by default. See the [slow jobs recorder](#slow-jobs-recorder) documentation for more information.

<a name="slow-queries-card"></a>
#### Slow Queries

The `<livewire:pulse.slow-queries />` card shows the database queries in your application that exceed the configured threshold, which is 1,000ms by default.

By default, slow queries are grouped based on the SQL query (without bindings) and the location where it occurred, but you may choose to not capture the location if you wish to group solely on the SQL query.

See the [slow queries recorder](#slow-queries-recorder) documentation for more information.

<a name="slow-outgoing-requests-card"></a>
#### Slow Outgoing Requests

The `<livewire:pulse.slow-outgoing-requests />` card shows outgoing requests made using Laravel's [HTTP client](/docs/{{version}}/http-client) that exceed the configured threshold, which is 1,000ms by default.

By default, entries will be grouped by the full URL. However, you may wish to normalize or group similar outgoing requests using regular expressions. See the [slow outgoing requests recorder](#slow-outgoing-requests-recorder) documentation for more information.

<a name="cache-card"></a>
#### Cache

The `<livewire:pulse.cache />` card shows the cache hit and miss statistics for your application, both globally and for individual keys.

By default, entries will be grouped by key. However, you may wish to normalize or group similar keys using regular expressions. See the [cache interactions recorder](#cache-interactions-recorder) documentation for more information.

<a name="capturing-entries"></a>
## Capturing Entries

Most Pulse recorders will automatically capture entries based on framework events dispatched by Laravel. However, the [servers recorder](#servers-recorder) and some third-party cards must poll for information regularly. To use these cards, you must run the `pulse:check` daemon on all of your individual application servers:

```php
php artisan pulse:check
```

> **Note**  
> To keep the `pulse:check` process running permanently in the background, you should use a process monitor such as Supervisor to ensure that the command does not stop running.

<a name="recorders"></a>
### Recorders

Recorders are responsible for capturing entries from your application to be recorded in the Pulse database. Recorders are registered and configured in the `recorders` section of the [Pulse configuration file](#configuration).

<a name="cache-interactions-recorder"></a>
#### Cache Interactions

The `CacheInteractions` recorder captures information about the [cache](/docs/{{version}}/cache) hits and misses occurring in your application for display on the [Cache](#cache-card) card.

You may optionally adjust the [sample rate](#sampling) and ignored key patterns.

You may also configure key grouping so that similar keys are grouped as a single entry. For example, you may wish to remove unique IDs from keys caching the same type of information. Groups are configured using a regular expression to "find and replace" parts of the key. An example is included in the configuration file:

```php
Recorders\CacheInteractions::class => [
    // ...
    'groups' => [
        // '/:\d+/' => ':*',
    ],
],
```

The first pattern that matches will be used. If no patterns match, then the key will be captured as-is.

<a name="exceptions-recorder"></a>
#### Exceptions

The `Exceptions` recorder captures information about reportable exceptions occurring in your application for display on the [Exceptions](#exceptions-card) card.

You may optionally adjust the [sample rate](#sampling) and ignored exceptions patterns. You may also configure whether to capture the location that the exception originated from. The captured location will be displayed on the Pulse dashboard which can help to track down the exception origin; however, if the same exception occurs in multiple locations then it will appear multiple times for each unique location.

<a name="queues-recorder"></a>
#### Queues

The `Queues` recorder captures information about your applications queues for display on the [Queues](#queues-card).

You may optionally adjust the [sample rate](#sampling) and ignored jobs patterns.

<a name="slow-jobs-recorder"></a>
#### Slow Jobs

The `SlowJobs` recorder captures information about slow jobs occurring in your application for display on the [Slow Jobs](#slow-jobs-recorder) card.

You may optionally adjust the slow job threshold, [sample rate](#sampling), and ignored job patterns.

<a name="slow-outgoing-requests-recorder"></a>
#### Slow Outgoing Requests

The `SlowOutgoingRequests` recorder captures information about outgoing HTTP requests made using Laravel's [HTTP client](/docs/{{version}}/http-client) that exceed the configured threshold for display on the [Slow Outgoing Requests](#slow-outgoing-requests-card) card.

You may optionally adjust the slow outgoing request threshold, [sample rate](#sampling), and ignored URL patterns.

You may also configure URL grouping so that similar URLs are grouped as a single entry. For example, you may wish to remove unique IDs from URL paths or group by domain only. Groups are configured using a regular expression to "find and replace" parts of the URL. Some examples are included in the configuration file:

```php
Recorders\OutgoingRequests::class => [
    // ...
    'groups' => [
        // '#^https://api\.github\.com/repos/.*$#' => 'api.github.com/repos/*',
        // '#^https?://([^/]*).*$#' => '\1',
        // '#/\d+#' => '/*',
    ],
],
```

The first pattern that matches will be used. If no patterns match, then the URL will be captured as-is.

<a name="slow-queries-recorder"></a>
#### Slow Queries

The `SlowQueries` recorder captures any database queries in your application that exceed the configured threshold for display on the [Slow Queries](#slow-queries-card) card.

You may optionally adjust the slow query threshold, [sample rate](#sampling), and ignored query patterns. You may also configure whether to capture the query location. The captured location will be displayed on the Pulse dashboard which can help to track down the query origin; however, if the same query is made in multiple locations then it will appear multiple times for each unique location.

<a name="slow-requests-recorder"></a>
#### Slow Requests

The `Requests` recorder captures information about requests made to your application for display on the [Slow Requests](#slow-requests-card) and [Application Usage](#application-usage-card) cards.

You may optionally adjust the slow route threshold, [sample rate](#sampling), and ignored paths.

<a name="servers-recorder"></a>
#### Servers

The `Servers` recorder captures CPU, memory, and storage usage of the servers that power your application for display on the [Servers](#servers-card) card. This recorder requires the [`pulse:check` command](#capturing-entries) to be running on each of the servers you wish to monitor.

Each reporting server must have a unique name. By default, Pulse will use the value returned by PHP's `gethostname` function. If you wish to customize this, you may set the `PULSE_SERVER_NAME` environment variable:

```env
PULSE_SERVER_NAME=load-balancer
```

The Pulse configuration file also allows you to customize the directories that are monitored.

<a name="user-jobs-recorder"></a>
#### User Jobs

The `UserJobs` recorder captures information about the users dispatching jobs in your application for display on the [Application Usage](#application-usage-card) card.

You may optionally adjust the [sample rate](#sampling) and ignored job patterns.

<a name="user-requests-recorder"></a>
#### User Requests

The `UserRequests` recorder captures information about the users making requests to your application for display on the [Application Usage](#application-usage-card) card.
[Application Usage](#application-usage-card) card

You may optionally adjust the [sample rate](#sampling) and ignored job patterns.

<a name="performance"></a>
## Performance

Pulse has been designed to drop into an existing application without requiring any additional infrastructure. However, for high-traffic applications, there are several ways of removing any impact Pulse may have on your application's performance.

<a name="using-a-different-database"></a>
### Using a Different Database

For high-traffic applications, you may prefer to use a dedicated database connection for Pulse to avoid impacting your application database.

You may customize the [database connection](/docs/{{version}}/database#configuration) used by Pulse by setting the `PULSE_DB_CONNECTION` environment variable.

```env
PULSE_DB_CONNECTION=pulse
```

<a name="ingest"></a>
### Redis Ingest

By default, Pulse will store entries directly to the [configured database connection](#using-a-different-database) after the HTTP response has been sent to the client or a job has been processed; however, you may use Pulse's Redis ingest driver to send entries to a Redis stream instead. This can be enabled by configuring the `PULSE_INGEST_DRIVER` environment variable:

```
PULSE_INGEST_DRIVER=redis
```

Pulse will use your default [Redis connection](/docs/{{version}}/redis#configuration) by default, but you may customize this via the `PULSE_REDIS_CONNECTION` environment variable:

```
PULSE_REDIS_CONNECTION=pulse
```

When using the Redis ingest, you will need to run the `pulse:work` command to monitor the stream and move entries from Redis into Pulse's database tables.

```php
php artisan pulse:work
```

> **Note**  
> To keep the `pulse:work` process running permanently in the background, you should use a process monitor such as Supervisor to ensure that the Pulse worker does not stop running.

<a name="sampling"></a>
### Sampling

By default, Pulse will capture every relevant event that occurs in your application. For high-traffic applications, this can result in needing to aggregate millions of database rows in the dashboard, especially for longer time periods.

You may instead choose to enable "sampling" on certain Pulse data recorders. For example, setting the sample rate to `0.1` on the [`User Requests`](#user-requests-recorder) recorder will mean that you only record approximately 10% of the requests to your application. In the dashboard, the values will be scaled up and prefixed with a `~` to indicate that they are an approximation.

In general, the more entries you have for a particular metric, the lower you can safely set the sample rate without sacrificing too much accuracy.

<a name="trimming"></a>
### Trimming

Pulse will automatically trim its stored entries once they are outside of the dashboard window. Trimming occurs when ingesting data using a lottery system which may be customized in the Pulse [configuration file](#configuration).

<a name="pulse-exceptions"></a>
### Handling Pulse Exceptions

If an exception occurs while capturing Pulse data, such as being unable to connect to the storage database, Pulse will silently fail to avoid impacting your application.

If you wish to customize how these exceptions are handled, you may provide a closure to the `handleExceptionsUsing` method:

```php
use \Laravel\Pulse\Facades\Pulse;
use \Illuminate\Support\Facades\Log;

Pulse::handleExceptionsUsing(function ($e) {
    Log::debug('An exception happened in Pulse', [
        'message' => $e->getMessage(),
        'stack' => $e->getTraceAsString(),
    ]);
});
```
