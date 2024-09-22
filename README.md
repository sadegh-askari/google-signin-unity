# Google Sign-In Unity Plugin
_Copyright (c) 2017 Google Inc. All rights reserved._

> [!IMPORTANT]  
> Forked to preserve git url in manifest.json

>[!WARNING]
>Some information might be outdated or incorrect.

## Main contributors

- @CodeMasterYi. Update Android version to 20+ & iOS to 6.0.2. Refractor iOS adapter.
- @Thaina. Upgraded base library to newer version.
- @DulgiKim. IOS fix from [pillsgood fork](https://github.com/pillsgood/google-signin-unity ).  [Issue comment](https://github.com/googlesamples/google-signin-unity/pull/205#issuecomment-1724733615)

## Overview

Google Sign-In API plugin for Unity game engine.  Works with Android and iOS.
This plugin exposes the Google Sign-In API within Unity.  This is specifically
intended to be used by Unity projects that require OAuth ID tokens or server
auth codes.

It is cross-platform, supporting both Android and iOS.

See [Google Sign-In for Android](https://developers.google.com/identity/android-credential-manager) for more information.
Tested in unity 2021.3.21 and unity 6000.0.5

## Get started

Add UPM dependency

```json
{
  "dependencies": {
    "com.google.external-dependency-manager": "https://github.com/googlesamples/unity-jar-resolver.git?path=upm",
    "com.google.signin": "https://github.com/Scrag0/google-signin-unity.git",
    ...
  }
}
```

## Configuring the application on the API Console

To authenticate you need to create credentials on the [API console](https://console.cloud.google.com) for your
application. 
1. Create project specific for your application.
2. Open Navigation Menu/Api & Services/OAuth consent screen
	1. Setup OAuth consent screen information, which will be displayed in application, and scopes, which you need to get information about user from.
3. Open Navigation Menu/Api & Services/Credentials
	1. Create credentials/OAuth client ID per platform
	2. Use specific client IDs per platform in configuration of google sign in

### OAuth client ID setup

- Web Client ID. Simply creates. No specific settings tuning
- Android Client ID. To setup android client ID you need:
	- Package name, which is written in Unity player settings;
	- SHA1 fingerprint from keystore used to sign your application. Used keystore is configured in the publishing settings of the Android Player properties in the Unity editor. Keystore must be the same one used to generate the SHA1 fingerprint when creating the application on the console.
- IOS Client ID.

### Configure Google Sign in for IOS 

Also, [New version of iOS recommend](https://developers.google.com/identity/sign-in/ios/quick-migration-guide#google_sign-in_sdk_v700) that we should set `GIDClientID` and `GIDServerClientID` into Info.plist

So Thaina have add an editor tool `PListProcessor` that look for plist files in the project, extract `CLIENT_ID` and `WEB_CLIENT_ID` property of the plist which contain the `BUNDLE_ID` with the same name as bundle identifier of the project.

The plist file in the project should be downloaded from Google Cloud Console credential page.

Select iOS credential and download at ⬇ button

```xml
<!-- This plist was the default format downloaded from your google cloud console -->
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
	<key>CLIENT_ID</key> 
	<string>{YourCloudProjectID}-yyyyyYYYYyyyyYYYYYYYYYYYYYYyyyyyy.apps.googleusercontent.com</string>
	<key>REVERSED_CLIENT_ID</key>
	<string>com.googleusercontent.apps.{YourCloudProjectID}-yyyyyYYYYyyyyYYYYYYYYYYYYYYyyyyyy</string>
	<key>PLIST_VERSION</key>
	<string>1</string>
	<key>BUNDLE_ID</key>
	<string>com.{YourCompany}.{YourProductName}</string>
<!-- Optional, These 2 lines below should be added manually if you need ServerAuthCode -->
  <key>WEB_CLIENT_ID</key>
  <string>{YourCloudProjectID}-zzzZZZZZZZZZZZZZZzzzzzzzzzzZZZzzz.apps.googleusercontent.com</string>
</dict>
</plist>
```

## Thaina's remarks

https://developer.android.com/identity/sign-in/legacy-gsi-migration
https://developers.google.com/identity/sign-in/ios/quick-migration-guide

Android was migrated to use `CredentialManager` and `AuthorizationClient` since [GoogleSignInAccount was deprecated](https://developers.google.com/android/reference/com/google/android/gms/auth/api/signin/GoogleSignInAccount).

However, `GoogleIdTokenCredential` actually not provide numeric unique ID anymore and set email as userId instead, so Thaina have to extract jwt `sub` value from idToken (which seem like the same id as userId from GoogleSignIn of other platform).

Also, this new system seem like it did not support email hint. And now require WebClientId in addition to Android Client ID. Which need to provided at configuration initialization

```C#
        GoogleSignIn.Configuration = new GoogleSignInConfiguration() {
            RequestEmail = true,
            RequestProfile = true,
            RequestIdToken = true,
            RequestAuthCode = true,
            // must be web client ID, not android client ID
            WebClientId = "XXXXXXXXX-xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx.apps.googleusercontent.com",
#if UNITY_EDITOR || UNITY_STANDALONE
            ClientSecret = "XXXXXX-xxxXXXxxxXXXxxx-xxxxXXXXX" // optional for windows/macos and test in editor
#endif
        };
```

## Using this plugin with Firebase Auth
Follow the instructions to use Firebase Auth with Credentials on the [Firebase developer website]( https://firebase.google.com/docs/unity/setup).

Make sure to copy the google-services.json and/or GoogleService-Info.plist to your Unity project.

Then to use Google SignIn with Firebase Auth, you need to request an ID token when authenticating.
The steps are:
1. Configure Google SignIn to request an id token and set the web client id as described above.
2. Call __SignIn()__ (or __SignInSilently()__).
3. When handling the response, use the ID token to create a Firebase Credential.
4. Call Firebase Auth method  __SignInWithCredential()__.

```C#
    GoogleSignIn.Configuration = new GoogleSignInConfiguration {
      RequestIdToken = true,
      // Copy this value from the google-service.json file.
      // oauth_client with type == 3
      WebClientId = "1072123000000-iacvb7489h55760s3o2nf1xxxxxxxx.apps.googleusercontent.com"
    };

    Task<GoogleSignInUser> signIn = GoogleSignIn.DefaultInstance.SignIn ();

    TaskCompletionSource<FirebaseUser> signInCompleted = new TaskCompletionSource<FirebaseUser> ();
    signIn.ContinueWith (task => {
      if (task.IsCanceled) {
        signInCompleted.SetCanceled ();
      } else if (task.IsFaulted) {
        signInCompleted.SetException (task.Exception);
      } else {

        Credential credential = Firebase.Auth.GoogleAuthProvider.GetCredential (((Task<GoogleSignInUser>)task).Result.IdToken, null);
        auth.SignInWithCredentialAsync (credential).ContinueWith (authTask => {
          if (authTask.IsCanceled) {
            signInCompleted.SetCanceled();
          } else if (authTask.IsFaulted) {
            signInCompleted.SetException(authTask.Exception);
          } else {
            signInCompleted.SetResult(((Task<FirebaseUser>)authTask).Result);
          }
        });
      }
    });
```
