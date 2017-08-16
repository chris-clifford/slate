---
title: RBA Android Documentation

language_tabs: # must be one of https://git.io/vQNgJ
  - java

toc_footers:
  - <a href='#'>Sign Up for a Developer Key</a>
  - <a href='https://github.com/tripit/slate'>Documentation Powered by Slate</a>

includes:
  - errors

search: true
---

# Introduction

This document will cover how the Android RBA SDK can be implemented into a native Android app. The first section will cover some details on how Android specifically handles all of the methods which can be invoked. Subsequent sections will go over how to invoke the policy, enroll, authenticate and unenroll calls.

# Implentation Details

> Example of the AetnaNGAAuth Class

```java
AetnaNGAAuth sAetnaNGAAuth = null;
try {
  sAetnaNGAAuth = AetnaNGAAuth.getInstance();
} catch(UserNotFoundException e) {
  //user was not found
}
```

This section will cover some of the classes used across all of the SDK methods, as well as some general information about how these methods are to be set up and called.

All of the SDK’s methods are invoked from the AetnaNGAAuth class. This is a static singleton class which is responsible for coordinating all the major RBA operations, keeping references to all of the authenticators available to the device, and keeping track of the state of the user with respect to the RBA flow. A reference to this class can be set with the following code:

> Use this pattern to make asynchronous API call

```java
Subject<T, T> observer = PublishSubject.create();
observer.subscribeOn(Schedulers.newThread())
.observeOn(AndroidSchedulers.mainThread())
.subscribe((T result) -> {
  //call is done
});
```

As a note, you’ll have to save a username to shared preferences with the key “NGAUser”, otherwise the getInstance method will throw a UserNotFoundException. It can be an actual username or a random string, it just can’t be empty.

Most of the methods which can be invoked involve making an asynchronous API call. The RxJava observer-subscriber implementation is used to handle this. For each of the asynchronous method calls, an observer is created, subscribed on, and passed into the method. Specific examples will be given in the appropriate sections, but they will all use the following pattern:

```java
subscribe(new Subscriber<T>() {
  @Override
  public void onCompleted() {}

    @Override
    public void onError(Throwable e) {}

      @Override
      public void onNext(T result) {
        //call is done
      }
    });
```

You may need to import the RetroLambda library to use the (T result) -> {} syntax shown. If you’re not using RetroLambda, the following syntax can be used:


In the assets folder of your project, you’ll need to add a file called configuration.properties. This contains some values which need to be configured from the app level such that the SDK can make the appropriate API calls. The fields which need to be added are as follows:

### Configuration Properties

Parameter | Value
--------- | ----------
BASE_API_URL | https://nga.acceptto.com -- base url of the server you’re pointing to
NGA_REQUESTS_TOKEN | api/v1/user/mock/app_token -- token endpoint for the server
NGA_REQUESTS_POLICY | api/v1/policy -- policy endpoint for the server
NGA_REQUESTS_LOA | rbaorcheng/v1/scopetoloa -- policy endpoint for the server
NGA_REQUESTS_ENROLL | api/v1/requests/enroll -- enroll endpoint for the server
NGA_REQUESTS_AUTHENTICATE | api/v1/requests/authenticate -- auth endpoint for the server
NGA_REQUESTS_UNENROLL | api/v1/requests/unenroll -- unenroll endpoint for the server
CLIENT_UID | 1767d8cf-ce5d-414f-b0ba-d680aeb79920 -- client uid used to retrieve access tokens
CLIENT_SECRET | 2tW3xQ7iM0rW6qY3wQ6tF1gW8kN1nO3rV5qL1dK2iQ0kI8sT1 -- client secret used to retrieve access tokens
SDK_VERSION | 1.0 -- SDK version which is sent in all operation requests
APIC_GRANT_TYPE | client_credentials -- APIC grant type used to retrieve access tokens
APIC_SCOPE | Public%20NonPII -- APIC scope used to retrieve access tokens
RBA_APP_ID | com.acceptto.androidngasample -- AppId sent in all operation requests
RBA_APP_VERSION | 1.0 -- AppVersion sent in all operation requests


Finally, before trying to use the SDK, you will need to specify your application context. The SDK comes with an ApplicationContextProvider class which can help you achieve this:

```java
ApplicationContextProvider.setContext(getApplicationContext());
```

You will need to do this before invoking any of the SDK methods.

# Policy

```java
sAetnaNGAAuth.setTrustMode(false);
```

There are a couple steps involved in making a good policy call. Firstly, depending on whether or not you plan to use trusted enroll, you’ll want to call the AetnaNGAAuth setTrustMode method, passing in either true or false:

```java
Subject<RBAEnrollState, RBAEnrollState> stateObserver = PublishSubject.create();
stateObserver.subscribeOn(Schedulers.newThread())
	.observeOn(AndroidSchedulers.mainThread())
	.subscribe((RBAEnrollState isRBAEnrolled) -> {
		if(isRBAEnrolled == RBAEnrollState.FINAL) {
			//user is enrolled
} else if(isRBAEnrolled == RBAEnrollState.TRANSIENT) {
	//user is not enrolled
}
});
```

After that you’ll need to set up and subscribe on your observer, which will return an RBAEnrollState object once it’s finished making the call:

```java
sAetnaNGAAuth.isNGAEnrolledWithCompletion(this, stateObserver);
```

Finally, invoke the isNGAEnrolledWithCompletion method on AetnaNGAAuth, passing in the calling activity context and your observer:

# Enroll

There are two ways to handle the enrollment operation in the SDK. First, NGA provides an automated flow which guides the user through the process of setting up new authentication methods. It invokes a series of activities, in order to explain to the user what NGA is and how to use it. All the text and coloring on these activities is configurable, which is documented in the Configuration section. It should be noted that if using this method of enrollment, the current setup is such that enabling PIN authentication is mandatory, while Fingerprint is optional. It is also possible to handle the enrollment process manually, which provides more flexibility on which authenticators to allow the user to enroll with. This way of enrolling also introduces more pitfalls however, all of which will be covered in this spec.

## Automated Enrollment

```java
sAetnaNGAAuth.startEnrollFlow(this, 9001);
```

Unlike the other operations, NGA automated enrollment invokes a series of activities in order to walk the user through the enrollment process, as opposed to simply running some logic in the background. Because of this, rather than using observers to listen for the result of the operation, enroll relies on the onActivityResult lifecycle method. Once the user either completes or errors/cancels out of the enroll process, the SDK activities finish with a result code indicating whether or not the operation was successful.

To start the enrollment process, the SDK uses the built in method startEnrollFlow. It’s given a context and a request code to be used on onActivityResult:

```java
@Override
protected void onActivityResult(int requestCode, int resultCode, Intent data) {
	if(requestCode == 9001) {
		if(resultCode == Activity.RESULT_OK) {
			//enroll was successful
} else {
	//enroll was not successful
}
}
}
```

The first parameter needs to be the Activity which invokes enroll in the first place. The second parameter is a request code which the SDK will use when invoking its own activities. This request code has to be the same one used on your onActivityResult method. Here’s an example of how to implement this:

The important pieces here are the request code and result code. The request code needs to match what you passed into the startEnrollFlow method. You also need to make sure you’re filtering it, otherwise you may see some unexpected behavior from some of the authenticators. The result code is set by the SDK to the value of Activity.RESULT_OK if enroll finishes successfully, so you’ll want to filter there as well.

## Manual Enrollment

```java
RBAAuthenticationType deviceRisk = NGAUtil.getRBATypeFromFriendlyName(NGAConstants.ACCEPTTO_DEVICE_RISK);
Subject<RBAEnrollState, RBAEnrollState> deviceRiskObserver = PublishSubject.create();
        deviceRiskObserver.subscribeOn(Schedulers.newThread())
                .observeOn(AndroidSchedulers.mainThread())
                .subscribe((RBAEnrollState deviceEnrollState) -> {
                    if(deviceEnrollState == RBAEnrollState.FINAL) {


                       //Enrollment successful
                    } else if(deviceEnrollState == RBAEnrollState.ERROR) {
                       //Enrollment errored
                    } else if(deviceEnrollState == RBAEnrollState.CANCEL) {
                       //Enrollment canceled
                    }
                }, Timber::e);
        sAetnaNGAAuth.enrollNGAWithCompletion(mActivity, deviceRisk, deviceRiskObserver);
```

If you’d like to display your own screens or dictate the rules of enrollment yourself, it is possible to invoke the SDK’s enrollment operation manually. This is done by using observers, similar to the other operations:

<aside class="warning">
It should be noted that if you are using manual enrollment, there are two practices which, if not followed properly, could lead to unexpected behavior in your application.
</aside>

Firstly, you must enroll device risk before enrolling any other authenticators. If you skip this step, policy will return incorrect response codes in future operations. Secondly, you should be sure the authenticator you are trying to enroll with is supported on the device. The static method findOpFromName, belonging to the AetnaNGAAuth class will return the correct NGAOperation instance if the authenticator is supported and null if not, and as such it should be called before attempting to enroll your authenticator.

# Authenticate

```java
Subject<RBAAuthenticationState, RBAAuthenticationState> stateObserver = PublishSubject.create();
stateObserver.subscribeOn(Schedulers.newThread())
	.observerOn(AndroidSchedulers.mainThread())
	.subscribe((RBAAuthenticationState rbaAuthenticationState) -> {
		if(rbaAuthenticationState == RBAAuthenticationState.FINAL) {
	//auth successful
} else if(rbaAuthenticationState == RBAAuthenticationState.CANCEL) {
	//user canceled auth flow
} else if(rbaAuthenticationState == RBAAuthenticationState.ERROR) {
	//auth flow error
}
});
```

Both authenticate and unenroll work very similarly to policy, each with just a few minor tweaks. For authenticate specifically, there are only two differences from policy; first is the value which is being returned in the subscriber’s onNext method. This is best shown with a code sample:

```java
sAetnaNGAAuth.authenticateNGA(this, 2, stateObserver);
```

The authenticate flow can either succeed, fail, or be canceled by the user, and will return an appropriate result code to allow for the app to handle each case. When invoking the authenticate method, in addition to the state observer and activity context, an argument representing the LOA must be passed in. At the time of writing, this value is always 2:

```java
@Override
protected void onActivityResult(int requestCode, int resultCode, Intent data) {
	sAetnaNGAAuth.getCurrentOp().passIntentData(requestCode, resultCode,
data, this);
}
```
Finally, any FIDO certified authenticators (and possibly others in the future) will need to use the onActivityResult method to successfully complete the authentication flow. This can be achieved by making the following method call:

You’ll want to be sure to put this in a branch such that it will not be triggered during the enroll flow. The request codes of the various authenticators will likely change over the course of time, and hence are out of the scope of this document. However, placing this line of code in an else statement should work just fine.

# Unenroll

```java
Subject<RBAUnEnrollState, RBAUnEnrollState> stateObserver = PublishSubject.create();
stateObserver.subscribeOn(Schedulers.newThread())
	.observeOn(AndroidSchedulers.mainThread())
	.subscribe((RBAUnEnrollState rbaUnenrollState) -> {
		if(rbaUnenrollState == RBAUnEnrollState.FINAL) {
			//unenroll successful
} else if(rbaUnenrollState == RBAUnEnrollState.ERROR) {
	//unenroll failed
}
});
sAetnaNGAAuth.unenrollNGAWithCompletion(this, stateObserver);
```

The unenroll flow is pretty much the same as authenticate. The only differences are that it uses RBAUnEnrollState to indicate whether the flow was successful or not, there is no cancel state, and no LOA parameter is required to invoke the method. A proper implementation is as follows:

Some authenticators also use onActivityResult for the unenroll flow, but the implementation is exactly the same as was described in the authenticate section (4). As long as you have that same code it should work just fine.

# Certificates

RBAAuthSDK performs several sign/verify operations during the creation of server requests, and for that reason we need to add the correct keys on the app level. The name for those keys should be the following:
* Nga_core_id_rsa_pub.pem (server public key)
* App_id_rsa_priv.pem (app private key)
Keys must be added on the project for all the needed targets. Please make sure they are included in the assets folder.
