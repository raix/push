Details and changes for FCM
===========================

These changes introduce a new token type `{ fcm: <token> }`. Only tokens keyed with fcm will be handled by these changes.
You can continue using apn and gcm tokens and they will be handled as before.
These changes have bumped phonegap-plugin-push to version 2.1.3 (can't go higher until Meteor updates to cordova-android 7.1.0)
You need to migrate your GCM project to Firebase to use this version, see: https://developers.google.com/cloud-messaging/android/android-migrate-fcm

See the phonegap-plugin-push installation docs for other particulars https://github.com/phonegap/phonegap-plugin-push/blob/master/docs/INSTALLATION.md
Such as how to include your `google-services.json` for sending to Android devices and `GoogleService-Info.plist` if you wish to send to iOS with Firebase

FCM messages are sent using the firebase-admin npm package and requires you to authenticate with a service account.

## API Details

### Server api
```js
    let serviceAccountJson = JSON.parse(Assets.getText('FirebaseAdminSdkServiceAccountKey.json')); // File located in the /private directory

    Push.Configure({
        fcm: {
            serviceAccountJson: serviceAccountJson
        },
        // gcm: {},
        // apn: {},
        // production: true, // apn production flag (not needed if using fcm for iOS)
        // keepNotifications: true,
        // sendInterval: 15000, 
        // sendBatchSize: 1000, 
        appName: 'MyAppName'
    });
```

### Client api
For options to be passed to phonegap-plugin-push see [here](https://github.com/phonegap/phonegap-plugin-push/blob/master/docs/API.md#pushnotificationinitoptions)
Android 26 requires you to use notification channels, phonegap-plugin-push creates a default channel with id 'PushPluginChannel'
Changes to the default notification channel need to be made before calling Push.Configure (see below)

This plugin calls PushNotification.init for you and saves the returned instance to `Push.push`, so you can call its API directly
See [here](https://github.com/phonegap/phonegap-plugin-push/blob/master/docs/API.md) for the full PushNotification (phonegap-plugin-push) API reference
E.g. you might call `Push.push.listChannels(...)` 
```js
    // Changes to the default notification channel need to be made before calling Push.Configure
    if (Meteor.isCordova){
        PushNotification.createChannel(
            function(){
                console.log('Channel Created!');
            },
            function(){
                console.log('Channel not created :(');
            },
            {
                id: 'PushPluginChannel',
                description: 'Channel Name Shown To Users',
                importance: 3,
                vibration: true
            }
        );
    }

    Push.Configure({
        cordovaOptions: {
            // Options here are passed to phonegap-plugin-push
            android: {
                sound: true,
                vibrate: true,
                clearBadge: false,
                clearNotifications: true,
                forceShow: false
                // icon: '',
                // iconColor: ''
            },
            ios: {
                // voip: false,
                alert: true,
                badge: true,
                sound: true,
                clearBadge: false,
                // categories: {},
                // fcmSandbox: false, // Doesn't need to be set if using fcm with 'APNs Authentication Key' not 'APNs Certificates'
                // topics: []
            }
        },
        appName: 'MyAppName'
    });

    Push.id(); // Unified application id - not a token
    // Push.setBadge(count); // ios specific - ignored everywhere else
```
Badge numbers actually can be set on some Android devices, the Push.setBadge function just hasn't be updated to reflect this.
For example you could call `Push.push.setApplicationIconBadgeNumber()` directly.

### Push.send API Changes
As previously, but with 'androidChannel' and 'fcm' keys. For the structure of the fcm object, see: https://firebase.google.com/docs/cloud-messaging/admin/send-messages
If you omit the fcm object (or parts of it), the plugin will correctly poulate/complete it with the top-level fields you specify, namely:
('title','text','badge','sound', 'notId', 'contentAvailable', 'androidChannel' and 'payload')

If you plan on using the fcm override, be mindful of various gotchas with the necessary phonegap-plugin-push message structure,
as some things contradict Google's docs, see: https://github.com/phonegap/phonegap-plugin-push/blob/master/docs/PAYLOAD.md

```js
Push.send({
    from: '',
    title: '', 
    text: '',
    badge: 55,
    sound: '',
    notId: 12345678,
    contentAvailable: 0/1,
    androidChannel: '', // If not specified will go to the default channel
    apn: {}, // unchanged
    gcm: {}, // unchanged
    fcm: {
        // Optional fcm overrides
        android: {
            // Android overrides
        },
        apns: {
            // iOS overrides
            headers: {},
            payload: {}
        },
        webpush: { 
            // TODO webpush with fcm not implemented yet
        },
        // data: (Don't use: we set this from 'payload')
        // notification: Don't use this, see: https://github.com/phonegap/phonegap-plugin-push/blob/master/docs/PAYLOAD.md#notification-vs-data-payloads 
        // token: (Don't use directly: we set this with the user's token)
        // topic: (not supported as we currently only send to tokens directly)
        // condition: (not supported as we currently only send to tokens directly)
    }),
    query: {},
    token: {},
    tokens: [{}, {}]),
    payload: {}, // All payload values must be strings if sending using FCM
    delayUntil: new Date(),
);
```

### iOS Background Handling of Notifications
If you want your Push event handlers to be called when your app is in the background (before the user clicks the notification),
have a read of https://github.com/phonegap/phonegap-plugin-push/blob/master/docs/PAYLOAD.md#background-notifications-1

To call the finish function use `Push.push.finish()`

