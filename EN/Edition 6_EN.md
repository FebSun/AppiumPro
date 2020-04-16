## Testing iOS App Upgrades

This week we wander back to the fun world of iOS to talk about testing app upgrades. One common requirement for many testers goes beyond the functionality of an app in a frozen state. Hopefully, your team is shipping new versions of your app to customers on a frequent basis. This means your customers are upgrading to the latest version, not installing it from scratch. There might all kinds of existing user data that needs to be migrated for the app to work correctly, and simply running functional tests of the new version would not be sufficient to catch these cases.

Luckily, the latest version of the Appium 1.8 beta now supports commands that make it easy to reinstall and relaunch your app from within a single Appium session. This opens up the possibility of testing an in-place upgrade.

To demonstrate this, I'd like to introduce The App, a cross-platform React Native demo app that I will be building out to support the various examples we'll encounter here in Appium Pro. If the name sounds vaguely familiar, it might be because I'm infringing on my friend Dave Haeffner's trademark use of the definite article! (Dave is the author of Elemental Selenium, and hosts a demo web app called The Internet). Luckily for me, Dave has been gracious in donating the name idea to Appium Pro, so we can proceed without worry.

I recently published v1.0.0 of The App, and it contains just one feature: a little text field which saves what you write into it and echoes it back to you. It just so happens that this data is saved internally using the key ***@TheApp:savedEcho***.

Now, in our toy app world, imagine that some crazy developer has decided that this key just absolutely needs to change to ***@TheApp:savedAwesomeText***. What this means is that if a user were to simply upgrade to the new version of the app after having saved some text in the old version of the app, the app would find no saved text under the new storage key! In other words, the upgrade would break the user's experience of the app. In this case, the fix is to provide some migration code in the app itself which looks for data at the old key and moves it over to the new key if any is there.

That's all the responsibility of the app developer (i.e., me). Let's pretend that instead of writing this migration code, I forgot, and went ahead and changed the key. I then released v1.0.1. This version contains the bug of the missing text due to the forgotten migration code, even though as a standalone version of the app it works fine. Let's keep pretending, and say that after much facepalming I wrote the migration code and released it as v1.0.2. Now the intrepid testers who are rightly suspicious of me want to write an automated test to prove that upgrading to v1.0.2 will actually work (and of course, are themselves facepalming that they didn't write such an upgrade test before v1.0.1 was released!).

So what do we need in terms of Appium code? Basically, we take advantage of these three ***mobile:*** methods:
```
driver.executeScript("mobile: terminateApp", args);
driver.executeScript("mobile: installApp", args);
driver.executeScript("mobile: launchApp", args);
```

This set of commands stops the current app (which we need to do explicitly so that the underlying WebDriverAgent instance doesn't think the app has crashed on us), installs the new app over top of it, and then launches the new app.

What do the ***args*** look like? In each case they should be a ***HashMap*** that is used to generate simple JSON structures. The ***terminateApp*** and ***launchApp*** commands both have one key, ***bundleId*** (which is of course the bundle ID of your app). ***installApp*** takes an ***app*** key, which is the path or URL to the new version of your app.

For our example, to create a passing test we'll need two apps: v1.0.0 and v1.0.2:
```
private String APP_V1_0_0 = "https://github.com/cloudgrey-io/the-app/releases/download/v1.0.0/TheApp-v1.0.0.app.zip";
private String APP_V1_0_2 = "https://github.com/cloudgrey-io/the-app/releases/download/v1.0.2/TheApp-v1.0.2.app.zip";
```

I'm happily using GitHub asset download URLs to feed into Appium here. Assuming we've started the test with ***APP_V1_0_0*** as our ***app*** capability, the trio of app upgrade commands then looks like:
```
HashMap<String, String> bundleArgs = new HashMap<>();
bundleArgs.put("bundleId", BUNDLE_ID);
driver.executeScript("mobile: terminateApp", bundleArgs);

HashMap<String, String> installArgs = new HashMap<>();
installArgs.put("app", APP_V1_0_2);
driver.executeScript("mobile: installApp", installArgs);

// can just reuse args for terminateApp
driver.executeScript("mobile: launchApp", bundleArgs);
```

For the full test flow, we want to:
1. Open up v1.0.0
2. Add our saved message and verify it is displayed
3. Upgrade to v1.0.2
4. Open the app and verify that the message is still displayed -- this proves that the old message has been migrated to the new key

Here's the code (note that I included the option of running the test with the version of the app that has the bug, which would produce a failing test):
```
import io.appium.java_client.MobileBy;
import io.appium.java_client.ios.IOSDriver;
import java.io.IOException;
import java.net.URL;
import java.util.HashMap;
import org.junit.Assert;
import org.junit.Test;
import org.junit.runner.RunWith;
import org.junit.runners.JUnit4;
import org.openqa.selenium.By;
import org.openqa.selenium.remote.DesiredCapabilities;
import org.openqa.selenium.support.ui.ExpectedConditions;
import org.openqa.selenium.support.ui.WebDriverWait;

@RunWith(JUnit4.class)
public class Edition006_iOS_Upgrade {

    private String BUNDLE_ID = "io.cloudgrey.the-app";

    private String APP_V1_0_0 = "https://github.com/cloudgrey-io/the-app/releases/download/v1.0.0/TheApp-v1.0.0.app.zip";
    private String APP_V1_0_1 = "https://github.com/cloudgrey-io/the-app/releases/download/v1.0.1/TheApp-v1.0.1.app.zip";
    private String APP_V1_0_2 = "https://github.com/cloudgrey-io/the-app/releases/download/v1.0.2/TheApp-v1.0.2.app.zip";

    private By msgInput = By.xpath("//XCUIElementTypeTextField[@name=\"messageInput\"]");
    private By savedMsg = MobileBy.AccessibilityId("savedMessage");
    private By saveMsgBtn = MobileBy.AccessibilityId("messageSaveBtn");
    private By echoBox = MobileBy.AccessibilityId("Echo Box");

    private String TEST_MESSAGE = "Hello World";

    @Test
    public void testSavedTextAfterUpgrade () throws IOException {
        DesiredCapabilities capabilities = new DesiredCapabilities();

        capabilities.setCapability("platformName", "iOS");
        capabilities.setCapability("deviceName", "iPhone 7");
        capabilities.setCapability("platformVersion", "11.2");
        capabilities.setCapability("app", APP_V1_0_0);

// change this to APP_V1_0_1 to experience a failing scenario
        String appUpgradeVersion = APP_V1_0_2;

// Open the app.
        IOSDriver driver = new IOSDriver<>(new URL("http://localhost:4723/wd/hub"), capabilities);

        WebDriverWait wait = new WebDriverWait(driver, 10);

        try {
            wait.until(ExpectedConditions.presenceOfElementLocated(echoBox)).click();
            wait.until(ExpectedConditions.presenceOfElementLocated(msgInput)).sendKeys(TEST_MESSAGE);
            wait.until(ExpectedConditions.presenceOfElementLocated(saveMsgBtn)).click();
            String savedText = wait.until(ExpectedConditions.presenceOfElementLocated(savedMsg)).getText();
            Assert.assertEquals(savedText, TEST_MESSAGE);

            HashMap<String, String> bundleArgs = new HashMap<>();
            bundleArgs.put("bundleId", BUNDLE_ID);
            driver.executeScript("mobile: terminateApp", bundleArgs);

            HashMap<String, String> installArgs = new HashMap<>();
            installArgs.put("app", appUpgradeVersion);
            driver.executeScript("mobile: installApp", installArgs);

            driver.executeScript("mobile: launchApp", bundleArgs);

            wait.until(ExpectedConditions.presenceOfElementLocated(echoBox)).click();
            savedText = wait.until(ExpectedConditions.presenceOfElementLocated(savedMsg)).getText();
            Assert.assertEquals(savedText, TEST_MESSAGE);
        } finally {
            driver.quit();
        }
    }
}
```

You can also find the code inside a working repository on GitHub. That's it for testing app upgrades on iOS! Stay tuned for a future edition where we discuss how to achieve the same results with Android.