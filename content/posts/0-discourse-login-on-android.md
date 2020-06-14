---
title: Discourse Login on Android
date: 2018-04-10
author: riyaz-ali
description: |
    This how-to aims to serve as a documentation on how
    one can implement Discourse's OAuth-based login flow
    on Android.
tags:
- Android
- Discourse
- HowTo
keywords:
- Android
- Discourse
- Discourse login
---

A few days back, I was looking for a hobby side-project to work on. I wanted to develop an Android application (open source) and to collaborate with my peers on [Google Udacity Android scholarship program](https://in.udacity.com/google-india-scholarships). On the Udacity discussion forum, there were many threads filled with exciting ideas but they all were either *too* ambitious or required quite a bit of time.

Then, one day while checking into the forum an idea struck me! Why not make an app for the discussion forum itself! I knew the forum was powered by [Discourse](https://discourse.org) and quickly started looking up for ways in which in could create an app for it...

And voila! I found the official [Discourse API](https://docs.discourse.org). I quickly went through the API and found that for *most* of the calls the user needed to be authenticated (due to obivious reasons). But the *only* way to *authenticate* was to use an *Admin generated API key*. Obviously, I couldn't generate the API keys myself and didn't want to bother the forum moderators or admins. Besides that the Admin API key would grant complete read/write access to the forum to anyone who have it, not something a forum admin would want. So I thought to scrape the idea...

But then something struck me! I knew a user could log into the forum from a browser (very obvious) and the Discourse frontend (written in Ember) communicated to the backend (written in Ruby) using an API. I suspected if it was the *same* API? Turns out the API used by Discourse frontend is a *superset* of the [public API](https://docs.discourse.org/) and it uses the `_t` cookie as the Session Cookie.

And so I started looking for ways I could use this to create a login for the user of my app. Turns out it's not that easy.

First I thought to use [Chrome Custom Tabs](https://developer.chrome.com/multidevice/android/customtabs) but it turns out that you *cannot pass cookies* between an app and a chrome custom tab for [security reasons](https://bugs.chromium.org/p/chromium/issues/detail?id=539240)

Then I tried using the [`WebView`](https://developer.android.com/reference/android/webkit/WebView.html) and succeeded to some extent. But the solution was very fragile as there is no well-defined `/login` page on Discourse and the user could easily navigate away from the `WebView`. I know I could've prevented the navigation but Discourse allows 3<sup>rd</sup> party login integration and then I would have to account for all possible domains a user could have navigated too! And besides these there are no autofill and *password remembering* features in `WebView` which means the user would have to login *manually* then.

{{< image src="https://media.giphy.com/media/YoGX4OZFjUwRq/giphy.gif" alt="Lo and Behold" position="center" >}}

And then I found what I was *actually* looking for! The [Discourse User API key specification](https://meta.discourse.org/t/user-api-keys-specification/48536)!! First created in Aug'16, this feature *facilitates* “application” access to Discourse instances without needing to involve moderators. The spec provides a flow *similar* to OAuth but with a few tweaks so that the Discourse instance doesn't have to *know* about the app prior to authentication. The token generation flow is documented [here](https://meta.discourse.org/t/user-api-keys-specification/48536)

The problem with User API key spec is that *it's documentation is outdated* and the canonical way is to *follow* the source of the [Discourse Mobile App](https://github.com/discourse/DiscourseMobile) which is a companion app and provide features like Push notificaton, etc. (not browsing!) The app is written in React Native.

After a lot of trial-and-error I finally go it to work natively (in Java) on Android. And following is the Gist of what I did.
  
  1 . Create a `DiscourseAuthAgent` class.
  
  ```java
  public class DiscourseAuthAgent {
    // Randomly generated or static client id for your app
    private String clientId;
    
    // Random string to verify that the 
    // reply indeed comes from Discourse server
    private String nonce;
    
    // RSA Public key
    private String publicKey;
    
    // RSA Private key
    private String privateKey;

    // API key we get from the server
    private String apiKey;
  }
  ```
  2 . To generate the RSA [`KeyPair`](https://developer.android.com/reference/java/security/KeyPair.html)

  ```java
  String[] generateKeyPair(){
    String[] result = new String[2];
    
    // generate KeyPair
    KeyPairGenerator kpg = KeyPairGenerator.getInstance("RSA");
    kpg.initialize(2048);
    KeyPair keyPair = kpg.generateKeyPair();

    // get and base64 encode public key
    byte[] publicKey  = keyPair.getPublic().getEncoded();
    String b64PublicKey = Base64.encodeToString(publicKey, Base64.DEFAULT);
    result[0] = b64PublicKey;

    // get and base63 encode private key
    byte[] privateKey = keyPair.getPrivate().getEncoded();
    String b64PrivateKey = Base64.encodeToString(privateKey, Base64.DEFAULT);
    b64PublicKey[1] = b64PrivateKey;

    return result;
  }
  ```

  3 . For the `clientId` and `nonce` you *should* generate a random string of length 32 and 16 respectively. I tried with other length options but failed so I guess it's good to go with these values.

  ```java
  static String randomString(final int sizeOfRandomString) {
    // taken from https://stackoverflow.com/a/12116194/6611700
    String ALLOWED_CHARACTERS = 
      "0123456789qwertyuiopasdfghjklzxcvbnm";
    
    final Random random= new Random();
    final StringBuilder sb= new StringBuilder(sizeOfRandomString);
    
    for(int i=0;i<sizeOfRandomString;++i)
      sb.append(
        ALLOWED_CHARACTERS.charAt(
          random.nextInt(ALLOWED_CHARACTERS.length())
        )
      );
    return sb.toString();
  }
  ```

  4 . Now to generate a [`Uri`](https://developer.android.com/reference/android/net/Uri.html),

  ```java
  /**
   * @param appName Your Application name that will be displayed to the user
   * @param scopes Permission scopes that you want (e.g. read, write)
   * @param redirectUri Uri to redirect to after authentication
   */
  Uri generateAuthUri(String appName, String[] scopes, String redirectUri) {
    clientId = randomString(32 /* size */);
    nonce = randomString(16 /* size */);
    String[] rsaKeyPair = generateKeyPair();
    publicKey = rsaKeyPair[0];
    privateKey = rsaKeyPair[1];

    return Uri.parse(BASE_URL).buildUpon()
      .appendEncodedPath("user-api-key/new")
      .appendQueryParameter("scopes", TextUtils.join(",", scopes))
      .appendQueryParameter("client_id", clientId)
      .appendQueryParameter("nonce", nonce)
      .appendQueryParameter("auth_redirect", redirectUri)
      .appendQueryParameter("push_url", "https://api.discourse.org/api/publish_android")  // placeholder
      .appendQueryParameter("application_name", appName)
      .appendQueryParameter("public_key", getFormattedPublicKey(Base64.decode(rsaPublicKey, Base64.DEFAULT)))
      .build();
  }

  String getFormattedPublicKey(byte[] unformatted){
    String pkcs1pem = "-----BEGIN PUBLIC KEY-----\n";
    pkcs1pem += Base64.encodeToString(unformatted, Base64.DEFAULT);
    pkcs1pem += "-----END PUBLIC KEY-----";
    return pkcs1pem;
  }
  ```

  5 . Finally to validate the result of the authentication and to get the API key from the redirect, we need to decrypt the `payload` query parameter that was sent with the redirection. The payload is encrypted using our `publicKey` and can only be decrypted by our `privateKey`

  ```java
  boolean handleAuthIntent(Intent data){
    if(null == data.getData())
      return false;
    
    try {
      String payload = java.net.URLDecoder.decode(
        redirection.getData().getQueryParameter("payload").replace("+", "%2B"),
        "UTF-8"
      ).replace("%2B", "+");

      // get back the private key
      byte[] privateKeyBytes = Base64.decode(privateKey, Base64.DEFAULT);
      PKCS8EncodedKeySpec spec = new PKCS8EncodedKeySpec(privateKeyBytes);
      PrivateKey privateKey = KeyFactory.getInstance("RSA").generatePrivate(spec);

      // initialize the Cipher
      Cipher cipher = Cipher.getInstance("RSA/NONE/PKCS1Padding");
      cipher.init(Cipher.DECRYPT_MODE, privateKey);

      // decrypt the payload into json
      byte[] decryptedBytes = cipher.doFinal(Base64.decode(payload, Base64.DEFAULT));
      String decrypted = new String(decryptedBytes);

      // create json object, verify nonce and let the fun begin!!!
      JSONObject payloadJson = new JSONObject(decrypted);

      // compare nonce
      if(!nonce.equals(payloadJson.getString("nonce")))
        throw new IllegalAccessException("invalid nonce");

      apiKey = payloadJson.getString("key");

      // and we are all done...
      return true;
    } catch (Exception e) {
      // failed to decrypt!
      // app must die :(
      throw new RuntimeException(e);
    }
  }
  ```

  6 . And that's it!! Now you just need to persist the `clientId` and `apiKey` (preferably in `SharedPreferences`) so that you can use them later to authenticate your requests.


Now to use the `DiscourseAuthAgent` class in your (say) `LoginActivity`, add this to your *manifest* :

  ```xml
  <activity android:name=".LoginActivity" android:launchMode="singleTask">
    <!-- deep link redirection handler after authentication -->
    <intent-filter>
      <action android:name="android.intent.action.VIEW" />
      <category android:name="android.intent.category.DEFAULT" />
      <category android:name="android.intent.category.BROWSABLE" />
      <data android:scheme="discourse" android:host="auth_redirect" />
    </intent-filter>
  </activity>
  ```

  This will make sure that the *same instance* of the Activity gets the [deep linking](https://developer.android.com/training/app-links/deep-linking.html) intent so that you can use the same `DiscourseAuthAgent` instance.

In the example `LoginActivity`, start authentication by getting Uri and launching the *default* browser or a Chrome Cutom Tab (preferred).

  ```java
  // on some button click
  // redirect uri must be exactly the same, I tried with other but failed
  Uri rediectTo = discourseAgent.generateAuthUri("My Amazing App", new String[]{ "read", "write" }, "discourse://auth_redirect");
  CustomTabsIntent customTabsIntent = new CustomTabsIntent.Builder()
    .setToolbarColor(getResources().getColor(R.color.colorPrimary))
    .build();
  customTabsIntent.launchUrl(this, redirectTo);
  ```

Then, when Chrome redirects the user back to your Activity:

```java
@Override protected void onNewIntent(Intent intent) {
  super.onNewIntent(intent);
  if(null != intent.getData() && intent.getData().toString().startsWith(REDIRECT_URI)){
    // redirection intent
    Timber.d("reply: %s", intent.getData());
    if(discourseAgent.handleAuthIntent(intent)){
      // authentication completed!
      Timber.d("Session authenticated!");
    } else {
      Timber.d("Failed to authenticate session :(");
    }
  }
}
```

And well that's it! Now that you have an `api key` you can make *autheticated* calls to the [Discourse Public API](http://docs.discourse.org) using the key. Just send the key in the `User-Api-Key` HTTP header and your `client id` in the `User-Api-Client-Id` HTTP header.