1. To setup webhooks in the laravel, firstly, the url should be publicly accessible but currently we are running the app on localhost, one method is to host the app on public servers but lets stick with free tools and instead we are going to use ngrok. Ngrok provides a secure public URL for local web server. Go to https://ngrok.com/ and signup and explain how to setup.
2. Open CMD and paste it to create a tunnel:

```bash
ngrok http --url=generous-oarfish-pure.ngrok-free.app 8000
```

1. Firstly, create a webhook endpoint in the Laravel web.php file. Make sure to use post method and to send 200 response.

```php
//import Request
use Illuminate\Http\Request;

//Paddle webhook
Route::post('webhook/paddle', function (Request $request) {
    // Handle Paddle webhook logic here
    
    // Example: Log the webhook data
    \Log::info('Paddle Webhook Received:', $request->all());

    // Respond with a 200 OK status
    return response()->json(['status' => 'success'], 200);
})->name('webhook.paddle');
```

1. Now we need to exclude this URI from the laravel CSRF protection. So, go to **`bootstrap/app.php`** and add this

```php
$middleware->validateCsrfTokens(except: [
            'webhook/*',
            'webhook/paddle',
        ]);
```

1. Update the webhook URL in the paddle.

```php
https://generous-oarfish-pure.ngrok-free.app/webhook/paddle
```

1. Now test the webhook
2. Now Update the User schema to include following fields as well:
    1. status: active | cancelled | past_due | none(default)
    2. subscription_type: basic | premium
    3. current_billing_period_start: none | date
    4. current_billing_period_end: none | date
    5. subscription_id: none | id of the paddle subscription
    
    ```php
    Schema::create('users', function (Blueprint $table) {
                $table->id();
                $table->string('name');
                $table->string('email')->unique();
                $table->string('role')->default('user'); // Default role for new users
                $table->timestamp('email_verified_at')->nullable();
                $table->string('password');
                $table->rememberToken();
    
                // Subscription fields
                $table->enum('status', ['active', 'cancelled', 'past_due', 'none'])->default('none');
                $table->enum('subscription_type', ['basic', 'premium'])->nullable();
                $table->timestamp('current_billing_period_start')->nullable();
                $table->timestamp('current_billing_period_end')->nullable();
                $table->string('subscription_id')->nullable(); // Paddle subscription ID
    
                $table->timestamps();
            });
    ```
    
3. Now Update user Model as well.

```php
protected $fillable = [
        'name',
        'email',
        'role', // Added role attribute
        'password',
        'status', // Subscription status
        'subscription_type', // Type of subscription
        'current_billing_period_start', // Start of current billing period
        'current_billing_period_end', // End of current billing period
        'subscription_id', // Paddle subscription ID
        
    ];
```

1. Run a fresh migration

```bash
php artisan migrate:fresh
```

1. Create a PaddleController for updating user information after subscription

```bash
php artisan make:controller PaddleController
```

1. Since this controller will handle on paddle webhook requests and will be bit complicated, so its a good practice to use it as a single-action controller. So, let's define `__invoke` function.

```php
<?php

namespace App\Http\Controllers;

use Carbon\Carbon;
use Illuminate\Http\Request;
use Illuminate\Http\Response;
use Illuminate\Support\Facades\Log;
use App\Models\User;

class PaddleController extends Controller
{
    public function __invoke(Request $request)
    {
        try {
            $eventType = $request->input('event_type');
            $data = $request->input('data');
            
            Log::info('Paddle webhook received', [
                'event_type' => $eventType,
                'subscription_id' => $data['id'] ?? null
            ]);

            switch ($eventType) {
                case 'subscription.activated':
                    $this->handleSubscriptionActivated($data);
                    break;
                    
                case 'subscription.updated':
                    $this->handleSubscriptionUpdated($data);
                    break;
                    
                case 'subscription.canceled':
                    $this->handleSubscriptionCanceled($data);
                    break;

                case 'subscription.past_due':
                    $this->handleSubscriptionPastDue($data);
                    break;
                    
                default:
                    Log::info('Unhandled webhook event type', ['event_type' => $eventType]);
                    break;
            }

            return response()->json(['status' => 'success'], Response::HTTP_OK);
            
        } catch (\Exception $e) {
            Log::error('Paddle webhook error', [
                'error' => $e->getMessage(),
                'trace' => $e->getTraceAsString(),
                'request_data' => $request->all()
            ]);
            
            return response()->json(['status' => 'error'], Response::HTTP_INTERNAL_SERVER_ERROR);
        }
    }

    private function handleSubscriptionActivated(array $data)
    {
        $email = $data['custom_data']['email'] ?? null;
        
        if (!$email) {
            Log::warning('No email found in subscription activation webhook', ['data' => $data]);
            return;
        }

        $user = User::where('email', $email)->first();
        
        if (!$user) {
            Log::warning('User not found for subscription activation', ['email' => $email]);
            return;
        }

        // Extract subscription type from price name or product name
        $subscriptionType = $this->extractSubscriptionType($data);
        
        $user->update([
            'status' => 'active',
            'subscription_type' => $subscriptionType,
            'current_billing_period_start' => Carbon::parse($data['current_billing_period']['starts_at']),
            'current_billing_period_end' => Carbon::parse($data['current_billing_period']['ends_at']),
            'subscription_id' => $data['id']
        ]);

        Log::info('User subscription activated', [
            'user_id' => $user->id,
            'subscription_id' => $data['id'],
            'subscription_type' => $subscriptionType
        ]);
    }

    private function handleSubscriptionUpdated(array $data)
    {

    }
    private function handleSubscriptionCanceled(array $data)
    {

    }
    private function handleSubscriptionPastDue(array $data)
    {

    }
    private function extractSubscriptionType(array $data): string
    {
        // Check if there are items in the subscription
        if (isset($data['items']) && count($data['items']) > 0) {
            $firstItem = $data['items'][0];
            
            if (isset($firstItem['product']['name'])) {
                $productName = strtolower($firstItem['product']['name']);
                if (str_contains($productName, 'premium')) {
                    return 'premium';
                } elseif (str_contains($productName, 'basic')) {
                    return 'basic';
                }
            }
        }
        
        return null;
    }

}

```

1. Update the webhook/paddle route in the web.php to use the newly created controller

```php
use App\Http\Controllers\PaddleController;

//Paddle webhook
Route::post('webhook/paddle', PaddleController::class)->name('webhook.paddle');

```

1. Finally, you can view webhook logs in the paddle dashboard by going to notifications and clicking on 3 dots of webhooks
