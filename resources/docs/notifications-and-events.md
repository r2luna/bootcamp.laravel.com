[TOC]

# <b>07.</b> Notifications & Events

Let's take Chirper to the next level by sending [email notifications](https://laravel.com/docs/notifications#introduction) when a new Chirp is created.

In addition to support for sending email, Laravel provides support for sending notifications across a variety of delivery channels, including email, SMS, and Slack. Plus, a variety of community built notification channels have been created to send notification over dozens of different channels! Notifications may also be stored in a database so they may be displayed in your web interface.

## Creating the notification

Artisan can, once again, do all the hard work for us with the following command:

```shell
./vendor/bin/sail artisan make:notification NewChirp
```

This will create a new notification at `app/Notifications/NewChirp.php` that is ready for us to customize.

Let's open the `NewChirp` class and allow it to accept the `Chirp` that was just created, and then customize the message to include the author's name and a snippet from the message:

```php filename=app/Notifications/NewChirp.php
<?php
namespace App\Notifications;

use App\Models\Chirp;// [tl! add]
use Illuminate\Bus\Queueable;
use Illuminate\Support\Str;// [tl! add]
use Illuminate\Contracts\Queue\ShouldQueue;
use Illuminate\Notifications\Messages\MailMessage;
use Illuminate\Notifications\Notification;

class NewChirp extends Notification
{
    use Queueable;

    /**
     * Create a new notification instance.
     *
     * @return void
     */
    public function __construct()// [tl! remove]
    public function __construct(public Chirp $chirp)// [tl! add]
    {
        //
    }
    // [tl! collapse:start]
    /**
     * Get the notification's delivery channels.
     *
     * @param  mixed  $notifiable
     * @return array
     */
    public function via($notifiable)
    {
        return ['mail'];
    }
    // [tl! collapse:end]
    /**
     * Get the mail representation of the notification.
     *
     * @param  mixed  $notifiable
     * @return \Illuminate\Notifications\Messages\MailMessage
     */
    public function toMail($notifiable)
    {
        return (new MailMessage)
                    ->line('The introduction to the notification.')// [tl! remove]
                    ->action('Notification Action', url('/'))// [tl! remove]
                    ->subject("New Chirp from {$this->chirp->user->name}")// [tl! add]
                    ->greeting("New Chirp from {$this->chirp->user->name}")// [tl! add]
                    ->line(Str::limit($this->chirp->message, 50))// [tl! add]
                    ->action('Go to Chirper', url('/'))// [tl! add]
                    ->line('Thank you for using our application!');
    }
    // [tl! collapse:start]
    /**
     * Get the array representation of the notification.
     *
     * @param  mixed  $notifiable
     * @return array
     */
    public function toArray($notifiable)
    {
        return [
            //
        ];
    }
    // [tl! collapse:end]
}
```

We could send the notification directly from the `store` method on our `ChirpController` class, but that adds more work for the controller, which in turn can slow down the request, especially as we'll be querying the database and sending emails.

Instead, let's dispatch an event that we can listen for and process in a background queue to keep our application snappy.

## Creating an event

Events are a great way to decouple various aspects of your application, since a single event can have multiple listeners that do not depend on each other.

Let's create our new event with the following command:

```shell
./vendor/bin/sail artisan make:event ChirpCreated
```

This will create a new event class at `app/Events/ChirpCreated.php`.

Since we'll be dispatching events for each new Chirp that is created, let's update our `ChirpCreated` event to accept the newly created `Chirp` so we may pass it on to our notification:

```php filename=app/Events/ChirpCreated.php
<?php

namespace App\Events;

use App\Models\Chirp;// [tl! add]
use Illuminate\Broadcasting\Channel;
use Illuminate\Broadcasting\InteractsWithSockets;
use Illuminate\Broadcasting\PresenceChannel;
use Illuminate\Broadcasting\PrivateChannel;
use Illuminate\Contracts\Broadcasting\ShouldBroadcast;
use Illuminate\Foundation\Events\Dispatchable;
use Illuminate\Queue\SerializesModels;

class ChirpCreated
{
    use Dispatchable, InteractsWithSockets, SerializesModels;

    /**
     * Create a new event instance.
     *
     * @return void
     */
    public function __construct()// [tl! remove]
    public function __construct(public Chirp $chirp)// [tl! add]
    {
        //
    }
    // [tl! collapse:start]
    /**
     * Get the channels the event should broadcast on.
     *
     * @return \Illuminate\Broadcasting\Channel|array
     */
    public function broadcastOn()
    {
        return new PrivateChannel('channel-name');
    }
    // [tl! collapse:end]
}
```

## Dispatching the event

Now that we have our event class, we're ready to dispatch it any time a Chirp is created. You may [dispatch events](https://laravel.com/docs/events#dispatching-events) anywhere in your application lifecycle, but as our event relates to the creation of an Eloquent model, we can configure our `Chirp` model to dispatch the event for us.

```php filename=app/Models/Chirp.php
<?php

namespace App\Models;

use App\Events\ChirpCreated;// [tl! add]
use Illuminate\Database\Eloquent\Factories\HasFactory;
use Illuminate\Database\Eloquent\Model;

class Chirp extends Model
{
    use HasFactory;

    protected $fillable = [
        'message',
    ];

    protected $dispatchesEvents = [// [tl! add:start]
        'created' => ChirpCreated::class,
    ];// [tl! add:end]

    public function user()
    {
        return $this->belongsTo(User::class);
    }
}
```

Now any time a new `Chirp` is created, the `ChirpCreated` event will be dispatched.

## Creating an event listener

Now that we're dispatching an event, we're ready to listen for that event and send our notification.

Let's create a listener that subscribes to our `ChirpCreated` event:

```sail
./vendor/bin/sail artisan make:listener SendChirpCreatedNotifications --event=ChirpCreated
```

The new listener will be placed at `app/Listeners/SendChirpCreatedNotifications.php`. Let's update the listener to send our notifications.

```php filename=app/Http/Listeners/SendChirpCreatedNotifications.php
<?php

namespace App\Listeners;

use App\Events\ChirpCreated;
use App\Models\User;// [tl! add]
use App\Notifications\NewChirp;// [tl! add]
use Illuminate\Contracts\Queue\ShouldQueue;
use Illuminate\Queue\InteractsWithQueue;

class SendChirpCreatedNotifications// [tl! remove]
class SendChirpCreatedNotifications implements ShouldQueue// [tl! add]
{
    // [tl! collapse:start]
    /**
     * Create the event listener.
     *
     * @return void
     */
    public function __construct()
    {
        //
    }
    // [tl! collapse:end]
    /**
     * Handle the event.
     *
     * @param  \App\Events\ChirpCreated  $event
     * @return void
     */
    public function handle(ChirpCreated $event)
    {
        //
        foreach (User::whereNot('id', $event->chirp->user_id)->cursor() as $user) {// [tl! remove:-1,1 add:start]
            $user->notify(new NewChirp($event->chirp));
        }// [tl! add:end]
    }
}
```

We've marked our listener with the `ShouldQueue` interface, which tells Laravel that the listener should be run in a [queue](https://laravel.com/docs/queues). By default, the "sync" queue will be used to process jobs synchronously; however, you may configure a queue worker to process jobs in the background.

We've also configured our listener to send notifications to every user in the platform, except the author of the Chirp. In reality, this might annoy users, so you may want to implement a "following" feature so users only receive notifications for accounts they follow.

We've used a [database cursor](https://laravel.com/docs/eloquent#cursors) to avoid loading every user into memory at once.

> **Note**
> In a production application you should add the ability for your users to unsubscribe from notifications like these.

### Registering the event listener

Finally, let's bind our event listener to the event. This will tell Laravel to invoke our event listener when the corresponding event is dispatched. We can do this within our `EventServiceProvider` class:

```php filename=App\Providers\EventServiceProvider.php
<?php

namespace App\Providers;

use App\Events\ChirpCreated;// [tl! add]
use App\Listeners\SendChirpCreatedNotifications;// [tl! add]
use Illuminate\Auth\Events\Registered;
use Illuminate\Auth\Listeners\SendEmailVerificationNotification;
use Illuminate\Foundation\Support\Providers\EventServiceProvider as ServiceProvider;
use Illuminate\Support\Facades\Event;

class EventServiceProvider extends ServiceProvider
{
    /**
     * The event to listener mappings for the application.
     *
     * @var array<class-string, array<int, class-string>>
     */
    protected $listen = [
        ChirpCreated::class => [// [tl! add:start]
            SendChirpCreatedNotifications::class,
        ],// [tl! add:end]

        Registered::class => [
            SendEmailVerificationNotification::class,
        ],
    ];
    // [tl! collapse:start]
    /**
     * Register any events for your application.
     *
     * @return void
     */
    public function boot()
    {
        //
    }

    /**
     * Determine if events and listeners should be automatically discovered.
     *
     * @return bool
     */
    public function shouldDiscoverEvents()
    {
        return false;
    }
    // [tl! collapse:end]
}
```

## Testing it out

Laravel Sail includes [MailHog](https://github.com/mailhog/MailHog), an email testing tool that catches any emails coming from your application so you may view them.

We've configured our notification not to send to the Chirp author, so be sure to register at least two users accounts. Then, go ahead and post a new Chirp to trigger a notification.

Open MailHog in your web browser by navigating to [http://localhost:8025/](http://localhost:8025/) where you'll find an inbox with the notification for the message you just chirped!

<img src="/img/screenshots/mailhog.png" alt="MailHog" class="rounded-lg border dark:border-none shadow-lg" />

### Sending emails in production

To send real emails in production, you will need an SMTP server, or a transactional email provider, such as Mailgun, Postmark, or Amazon SES. Laravel supports all of these out of the box. For more information, take a look at the [Mail documentation](https://laravel.com/docs/mail#introduction).

[Continue to learn about deploying your application...](/deploying)
