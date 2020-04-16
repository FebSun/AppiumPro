## Testing Android App Upgrades

Previously, we explored how to test app upgrades on iOS, and this week we're going to do the same for Android. What's the deal with app upgrades? The idea is that as testers, our responsibility goes beyond the functionality of an app in just one version. We also care about issues that might arise as users migrate from one version to another. Your customers are upgrading to the latest version, not installing it from scratch. There might be all kinds of existing user data that needs to be migrated for the app to work correctly, and simply running functional tests of the new version would not be sufficient to catch these cases.

Appium comes with some built-in commands to handle this kind of requirement. Fortunately, on Android it's even simpler than on iOS! To demonstrate, let's go back to the old version of The App, specifically v1.0.0. This version contains just one feature: a little text field which saves what you write into it and echoes it back to you. It just so happens that this data is saved internally using the key ***@TheApp:savedEcho***. (The specifics of how this data is saved have to do with React Native and aren't important---just imagine that your dev team has a way to save local user data that might need to persist between upgrades).

Now, imagine that some crazy developer has decided that this key just absolutely needs to change to ***@TheApp:savedAwesomeText***. What this means is that if a user were to simply upgrade to the new version of the app after having saved some text in the old version of the app, the app would find no saved text under the new storage key! In other words, the upgrade would break the user's experience of the app. In this case, the fix is to provide some migration code in the app itself which looks for data at the old key and moves it over to the new key if any is there.

That's all the responsibility of the app developer (in this case, me). Let's pretend that like a good developer I initially forgot to write the migration code after changing the data storage key. I then release v1.0.1. This version will unfortunately contain the bug I described above (of the missing text) due to the forgotten migration code---even though as a standalone version of the app it works fine. Eventually I realize the mistake, and write the migration code. I can't re-release v1.0.1, so I release the fix as v1.0.2.

At this point, the testers, having been burned by my incompetence as a developer, want to write an automated test to prove that upgrading to v1.0.2 will actually work (and of course, are themselves facepalming that they didn't write such an upgrade test before v1.0.1 was released!).

So what do we need in terms of Appium code? Basically, we take advantage of these two methods:
```
driver.installApp("/path/to/apk");
// 'activity' is an instance of import io.appium.java_client.android.Activity;
driver.startActivity(activity);
```

This set of commands:
1. Replaces the existing app with the one given in the path (or URL) as the argument to installApp, and in so doing stops the old version of the app from running (so there is no need to explicitly stop the app before installing the new one).
2. Starts up the new version of the app using its package name and whatever activity you choose.

The only wrinkle here is that the argument to ***startActivity*** must be of type ***Activity***. There is a separate ***Activity*** class because there are a number of options that can be passed to this method. We're only interested in the basic ones for the purposes of app upgrades, so we can simply create our ***Activity*** object as follows:
```
Activity activity = new Activity("com.mycompany.myapp", "MyActivity");
driver.startActivity(activity);
```

For our example, to create a passing app upgrade test we'll of course need two apps: our original version and the version to upgrade to. In our case, that's v1.0.0 and v1.0.2 of The App:
```
private String APP_V1_0_0 = "https://github.com/cloudgrey-io/the-app/releases/download/v1.0.0/TheApp-v1.0.0.apk";
private String APP_V1_0_2 = "https://github.com/cloudgrey-io/the-app/releases/download/v1.0.2/TheApp-v1.0.2.apk";
```

I'm happily using GitHub asset download URLs to feed into Appium here. Assuming we've started the test with ***APP_V1_0_0*** as our ***app*** capability, the duo of app upgrade commands then looks like:
```
driver.installApp(appUpgradeVersion);
Activity activity = new Activity(APP_PKG, APP_ACT);
driver.startActivity(activity);
```

For the full test flow, we want to:
1. Open up v1.0.0
2. Add our saved message and verify it is displayed
3. Upgrade to v1.0.2
4. Open the app and verify that the message is still displayed -- this proves that the old message has been migrated to the new key

Here's the code for the implementation of this full flow, including boilerplate:
```
```

(Note that I included the option of running the test with the version of the app that has the bug, namely v1.0.1, which would produce a failing test.)

As always, you can find the code inside a working repository on GitHub. That's it for testing app upgrades on Android! Remember to check out the previous tip on how to do the same thing with iOS.