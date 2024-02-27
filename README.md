
## Installing and Testing Laravel Websockets

### Reference URL
- For testing Laravel Websockets
  - https://christoph-rumpel.com/2020/11/laravel-real-time-notifications
- For Installing Laravel Websockets to sail
  - https://dterumalai.medium.com/add-laravel-websockets-to-sail-in-5-mins-71c8d9ceeb8a 

### Step installing and setting backend
#### sail install and composer packages install
```text
curl -s "https://laravel.build/ws-blog?with=mysql,redis" | bash

cd ws-blog
sail up -d
sail composer require pusher/pusher-php-server
sail composer require -W beyondcode/laravel-websockets
```

#### .env setting
```text
...
BROADCAST_DRIVER=pusher
...
PUSHER_APP_ID=12345
PUSHER_APP_KEY=12345
PUSHER_APP_SECRET=12345
PUSHER_APP_CLUSTER=mt1
PUSHER_HOST=127.0.0.1
PUSHER_PORT=80
PUSHER_SCHEME=http
...
```
#### config/broadcasting.php
Change pusher setting as below.
```text
'pusher' => [
    'driver' => 'pusher',
    'key' => env('PUSHER_APP_KEY'),
    'secret' => env('PUSHER_APP_SECRET'),
    'app_id' => env('PUSHER_APP_ID'),
    'options' => [
        'cluster' => env('PUSHER_APP_CLUSTER'),
        'useTLS' => false,
        'encrypted' => false,
        'host' => '127.0.0.1',
        'port' => 6001,
        'scheme' => 'http'
    ],
],
```

#### Publish
```text
sail artisan vendor:publish --provider="BeyondCode\LaravelWebSockets\WebSocketsServiceProvider" --tag="migrations"

sail artisan vendor:publish --provider="BeyondCode\LaravelWebSockets\WebSocketsServiceProvider" --tag="config"

sail artisan sail:publish
```

#### Dokcer setting
##### docker/8.3/supervisord.conf
Add following lines at the bottom of the file.
```
[program:websockets]
command=/usr/bin/php /var/www/html/artisan websockets:serve
numprocs=1
autostart=true
autorestart=true
user=sail
```

##### docker-compose.yml
Add following line into laravel.test container's `ports` part in docker-compose.yml file.
```text
- '${LARAVEL_WEBSOCKETS_PORT:-6001}:6001'
```

##### rebuild docker and migrate
```
sail down -v
sail build --no-cache
sail up -d
sail artisan migrate
```


#### Access to websockets dashboard
1. Access to http://localhost/laravel-websockets
2. Connect to websockets server by port `6001`

### Testing the function of websockets
#### Creating an event file
```text
sail artisan make:event RealTimeMessage
```

#### Modify app/Events/RealTimeMessage.php
```
<?php

namespace App\Events;

use Illuminate\Broadcasting\Channel;
use Illuminate\Contracts\Broadcasting\ShouldBroadcast;
use Illuminate\Queue\SerializesModels;

class RealTimeMessage implements ShouldBroadcast
{
    use SerializesModels;

    public string $message;

    public function __construct(string $message)
    {
        $this->message = $message;
    }

    public function broadcastOn(): Channel
    {
        return new Channel('events');
    }
}
```

#### Fire the event and test
```text
sail artisan tinker

// Below is a command to be typed in tinker console.
event(new App\Events\RealTimeMessage('Hello World! I am an event'));
```
##### Access to websockets dashboard
1. access to http://localhost/laravel-websockets
2. connect to websockets server by port `6001`
3. if you can see `Channel: events, Event: App\Events\RealTimeMessage`, test to fire the event is success.

### Listen websockets from Front-end
#### Install npm packages
Add these lines at devDependencies in `package.json`.
```text
        "laravel-echo": "^1.15.1",
        "pusher-js": "^8.2.0",
```
Run this command.
```text
npm install
```

#### Setting `resources/js/bootstrap.js`
```text
import Echo from 'laravel-echo';
import Pusher from 'pusher-js';

window.Pusher = Pusher;

window.Echo = new Echo({
    broadcaster: 'pusher',
    key: import.meta.env.VITE_PUSHER_APP_KEY,
    cluster: import.meta.env.VITE_PUSHER_APP_CLUSTER,
    forceTLS: false,
    wsPort: 6001,
    wsHost: window.location.hostname,
});
```

#### Making view to receive a message from the fired event
Please refer to `resources/views/ws.blade.php`

##### Adding route in `routes/web.php` 
```text
Route::get('/ws', function (){
    return view('ws');
});
```

#### Launch vite
```text
npm run dev
```

#### Fire the event and test
##### Access to the view `ws`
1. Access to http://localhost/ws on browser.
2. Fire the event from Tinker.
```text
sail artisan tinker

// Below is a command to be typed in tinker console.
event(new App\Events\RealTimeMessage('Hello World! I am an event'));
```
3. You should be able to see message in console and body on the browser.

