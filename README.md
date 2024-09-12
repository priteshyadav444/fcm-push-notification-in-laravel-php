
# Firebase Push Notifications in Laravel

This guide explains how to integrate Firebase Cloud Messaging (FCM) to send push notifications in a Laravel project. We'll use the Firebase API to send push notifications to mobile devices, generate access tokens, and cache them for efficiency.

---

## Prerequisites

- **Laravel 8+**
- **Firebase account**
- **Google Cloud Project with Firebase Cloud Messaging enabled**

---

## Step 1: Install Required Packages

First, install the necessary package to interact with Firebase:

```bash
composer require google/apiclient
```
If you're facing a timeout error then either increase the timeout for composer by adding the env flag as COMPOSER_PROCESS_TIMEOUT=600 composer install or you can put this in the config section of the composer schema: https://packagist.org/packages/google/apiclient
---

## Step 2: Generate Service Account JSON File

1. Go to [Firebase Console](https://console.firebase.google.com/).
2. Select your project and navigate to **Project Settings > Service Accounts**.
3. Click on **Generate New Private Key** and download the JSON file.
4. Move the `service_account.json` file to a secure location in your Laravel project, e.g., `storage/app/private/service_account.json`.

---

## Step 3: Enable Firebase Cloud Messaging API

1. Go to [Google Cloud Console](https://console.cloud.google.com/).
2. Navigate to **APIs & Services > Library**.
3. Search for **Firebase Cloud Messaging API** and enable it.

---

## Step 4: Add FCM Endpoint to Configuration

In your `config/custom.php` (create this file if it doesn't exist), add the following configuration:

```php
<?php
return [
    'fcm_endpoint' => 'https://fcm.googleapis.com/v1/projects/YOUR_PROJECT_ID/messages:send'
];
```

Replace `YOUR_PROJECT_ID` with the actual Project ID from your Firebase project.

---

## Step 5: Create a FirebaseNotificationService Class

This class will handle generating an access token and sending push notifications.

```php
<?php

use Google\Auth\Credentials\ServiceAccountCredentials;
use Illuminate\Support\Facades\Log;
use Illuminate\Support\Facades\Http;
use Illuminate\Support\Facades\Cache;

class FirebaseNotificationService
{
    /**
     * Generate access token for Firebase Cloud Messaging.
     *
     * @return string|null
     */
    private function generateAccessToken()
    {
        // Check if the token exists in cache
        if (Cache::has('firebase_access_token')) {
            return Cache::get('firebase_access_token');
        }

        try {
            // Path to the service_account.json file
            $credentialsFilePath = storage_path('app/private/service_account.json');

            // Create credentials object
            $credentials = new ServiceAccountCredentials(
                ['https://www.googleapis.com/auth/firebase.messaging'],
                $credentialsFilePath
            );

            // Fetch the token
            $token = $credentials->fetchAuthToken();
            $accessToken = $token['access_token'];

            // Cache the token for 55 minutes
            Cache::put('firebase_access_token', $accessToken, now()->addMinutes(55));

            return $accessToken;
        } catch (\Exception $e) {
            Log::error('Error generating access token: ' . $e->getMessage());
            return null;
        }
    }

    /**
     * Send push notifications via Firebase Cloud Messaging.
     *
     * @param $to
     * @param string $title
     * @param string $body
     */
    public function sendPushNotificationSync($to, $title, $body)
    {
        // Generate access token for Firebase
        $access_token = $this->generateAccessToken();

        // Retrieve the user's device details
        $devices = DeviceDetail::where('user_id', $to->id)
            ->orderBy('created_at', 'DESC')
            ->get();

        // Define the FCM endpoint
        $fcmEndpoint = config('custom.fcm_endpoint');

        foreach ($devices as $device) {
            if (!empty($device)) {
                info($device->device_token);
                try {
                    // Prepare the message payload (title and body only)
                    $message = [
                        'message' => [
                            'token' => $device->device_token,
                            'notification' => [
                                'title' => $title,
                                'body'  => $body
                            ]
                        ]
                    ];

                    // Send the notification via HTTP POST request
                    $response = Http::withHeaders([
                        'Authorization' => 'Bearer ' . $access_token,
                        'Content-Type' => 'application/json',
                    ])->post($fcmEndpoint, json_encode($message)); // Ensure payload is a JSON string

                    // Log the result of the notification
                    if ($response->status() == 200) {
                        Log::info('Notification sent successfully: ' . $response->body());
                    } else {
                        Log::error('Error sending FCM notification: ' . $response->body());
                    }
                } catch (\Exception $e) {
                    Log::error('Error sending FCM notification: ' . $e->getMessage());
                }
            }
        }
    }
}
```

### Code Breakdown:
- **generateAccessToken()**: This function retrieves the access token from Google. It checks if the token is already cached and if not, it generates a new one and stores it in the cache for 55 minutes.
- **sendPushNotificationSync()**: This function sends push notifications to the user’s devices using the Firebase token and notification payload (title and body).

---

## Step 6: Implement Push Notification Logic in Controller

You can use the `FirebaseNotificationService` in any controller to send notifications:

```php
use App\Services\FirebaseNotificationService;

class NotificationController extends Controller
{
    public function sendNotification(Request $request)
    {
        $user = User::find($request->input('user_id')); // Target user
        $title = $request->input('title'); // Notification title
        $body = $request->input('body'); // Notification body

        // Send push notification
        $firebaseService = new FirebaseNotificationService();
        $firebaseService->sendPushNotificationSync($user, $title, $body);

        return response()->json(['message' => 'Notification sent successfully.']);
    }
}
```

---

## Step 7: Handling Device Tokens

In your database, you should have a table to store device tokens for each user. Here’s an example of how you can set up a `DeviceDetail` model to store the tokens:

```php
// Example of DeviceDetail model
use Illuminate\Database\Eloquent\Model;

class DeviceDetail extends Model
{
    protected $fillable = ['user_id', 'device_token', 'device_type'];

    public function user()
    {
        return $this->belongsTo(User::class);
    }
}
```

---

## Step 8: Caching Firebase Access Token

To avoid generating the access token for every request, we cache it for 55 minutes (as the token is valid for 1 hour). You can use Laravel’s cache to store this token.

### Cache Token Example

```php
if (Cache::has('firebase_access_token')) {
    $accessToken = Cache::get('firebase_access_token');
} else {
    // Generate new token and cache it
    $accessToken = $this->generateAccessToken();
    Cache::put('firebase_access_token', $accessToken, now()->addMinutes(55));
}
```

This ensures that API calls to Google are minimized.

---

## Conclusion

With these steps, you can now successfully integrate Firebase Cloud Messaging into your Laravel project. The cached access token ensures optimized performance, and the push notifications are sent efficiently to user devices.

Feel free to adapt and extend this setup based on your project’s needs!

--- 
