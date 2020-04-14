## Seeding an Android device with test photos

As promised, this week's edition is an Android-flavored follow-up to last week's tip on seeding the iOS simulator with test photos. The problem we're trying to solve is how to get pictures with known content onto the device for use in our App Under Test. This is a requirement for any type of app that utilizes or processes images.

How do you test that your app does the right thing to a user-provided image?
> 1. Take some image you have lying around, and input it manually to your application.
> 2. Manually run the app function on your image. Maybe this is applying a certain type of filter, for example.
> 3. Still manually, extract the modified image from your app any way you can (texting it to yourself, for example!)

What these steps do is provide a gold-standard before-and-after which you can use as a test fixture. In an automated fashion now, we can provide the app with the same initial picture, run the desired function, and then retrieve the modified image. We can verify this modified image is byte-for-byte equivalent to our gold standard to ensure the app functionality still works as expected.

In this edition we focus on the problem of getting our initial picture onto the device. How does one do this for Android? Happily, we use the same function as for iOS: ***pushFile***. Under the hood, ***pushFile*** uses a series of ***ADB*** commands to shuffle the image to the device and then broadcast a system intent to refresh the media library. Since different device manufacturers put pictures in different places, you do need to know the path on your device that stores media. For emulators and at least some real devices, the path on the device is ***/mnt/sdcard/Pictures***, so this is an important constant to your remember. As an example:
```
driver.pushFile("/mnt/sdcard/Pictures/myPhoto.jpg", "/path/to/photo/locally/myPhoto.jpg");
```

As you can see, the first argument is the remote path on the device where we want the picture to end up. This is where you may need to refer to documentation on your particular device or Android OS flavor to ensure you have the right path. The second argument is the path to the file on your local machine, where your test is running. When the test executes, the Appium client will encode this file as a string and send it over to the Appium server, which will then do the job of getting it on the device. This architecture is nice because it means that ***pushFile*** works whether you're running locally, or on a cloud provider like Sauce Labs.

Let's take a look at a complete working example. In this example, we automate the built-in Google Photos app. First of all, we set up our desired capabilities to just use a built-in app:
```
DesiredCapabilities capabilities = new DesiredCapabilities();
capabilities.setCapability("platformName", "Android");
capabilities.setCapability("deviceName", "Android Emulator");
capabilities.setCapability("automationName", "UiAutomator2");
capabilities.setCapability("appPackage", "com.google.android.apps.photos");
capabilities.setCapability("appActivity", ".home.HomeActivity");

// Open the app.
AndroidDriver driver = new AndroidDriver<>(new URL("http://localhost:4723/wd/hub"), capabilities);
```

Once the app is open, we have to navigate through some UI boilerplate and other things Google wants us to do to use their cloud. Of course we are robots and not interested in their cloud! I've put this app state setup into its own method, along with setting up some locator constants, and logic that removes any existing pictures to make our verification easier later on:
```
private static By backupSwitch = By.id("com.google.android.apps.photos:id/auto_backup_switch");
private static By touchOutside = By.id("com.google.android.apps.photos:id/touch_outside");
private static By keepOff = By.xpath("//*[@text='KEEP OFF']");
private static By photo = By.xpath("//android.view.ViewGroup[contains(@content-desc, 'Photo taken')]");
private static By trash = By.id("com.google.android.apps.photos:id/trash");
private static By moveToTrash = By.xpath("//*[@text='MOVE TO TRASH']");

public void setupAppState(AndroidDriver driver) {
    // navigate through the google junk to get to the app
    WebDriverWait wait = new WebDriverWait(driver, 10);
    WebDriverWait shortWait = new WebDriverWait(driver, 3);
    wait.until(ExpectedConditions.presenceOfElementLocated(backupSwitch)).click();
    wait.until(ExpectedConditions.presenceOfElementLocated(touchOutside)).click();
    wait.until(ExpectedConditions.presenceOfElementLocated(keepOff)).click();

    // delete any existing pictures using an infinite loop broken when we can't find any
    // more pictures
    try {
        while (true) {
            shortWait.until(ExpectedConditions.presenceOfElementLocated(photo)).click();
            shortWait.until(ExpectedConditions.presenceOfElementLocated(trash)).click();
            shortWait.until(ExpectedConditions.presenceOfElementLocated(moveToTrash)).click();
        }
    } catch (TimeoutException ignore) {}
}
```

Finally we can get to the meat of our test, which relies upon the appropriate device media path constant:
```
setupAppState(driver);

// set up the file we want to push to the phone's library
File assetDir = new File(classpathRoot, "../assets");
File img = new File(assetDir.getCanonicalPath(), "cloudgrey.png");

// actually push the file
driver.pushFile(ANDROID_PHOTO_PATH + "/" + img.getName(), img);

// wait for the system to acknowledge the new photo, and use the WebDriverWait to verify
// that the new photo is there
WebDriverWait wait = new WebDriverWait(driver, 10);
ExpectedCondition condition = ExpectedConditions.numberOfElementsToBe(photo,1);
wait.until(condition);
```

As you can see, the actual bit we care about is as simple as calling ***pushFile***. For the sake of this test, we are simply verifying that our picture exists at the end. In a real-world scenario, we'd then run our app functionality on the picture, and retrieve it from the device to perform some local verification on it.

Bringing it all together, the entire test class looks like the following:
```
import io.appium.java_client.android.AndroidDriver;
import java.io.File;
import java.io.IOException;
import java.net.URL;
import org.junit.Test;
import org.openqa.selenium.By;
import org.openqa.selenium.TimeoutException;
import org.openqa.selenium.remote.DesiredCapabilities;
import org.openqa.selenium.support.ui.ExpectedCondition;
import org.openqa.selenium.support.ui.ExpectedConditions;
import org.openqa.selenium.support.ui.WebDriverWait;

// Note that to function correctly, this test must be run against a version of Appium which includes
// the appium-android-driver package at version 1.38 or higher, since it contains relevant bugfixes

public class Edition002_Android_Photos {

    private static String ANDROID_PHOTO_PATH = "/mnt/sdcard/Pictures";

    private static By backupSwitch = By.id("com.google.android.apps.photos:id/auto_backup_switch");
    private static By touchOutside = By.id("com.google.android.apps.photos:id/touch_outside");
    private static By keepOff = By.xpath("//*[@text='KEEP OFF']");
    private static By photo = By.xpath("//android.view.ViewGroup[contains(@content-desc, 'Photo taken')]");
    private static By trash = By.id("com.google.android.apps.photos:id/trash");
    private static By moveToTrash = By.xpath("//*[@text='MOVE TO TRASH']");

    @Test
    public void testSeedPhotoPicker() throws IOException {
        DesiredCapabilities capabilities = new DesiredCapabilities();

        File classpathRoot = new File(System.getProperty("user.dir"));

        capabilities.setCapability("platformName", "Android");
        capabilities.setCapability("deviceName", "Android Emulator");
        capabilities.setCapability("automationName", "UiAutomator2");
        capabilities.setCapability("appPackage", "com.google.android.apps.photos");
        capabilities.setCapability("appActivity", ".home.HomeActivity");

// Open the app.
        AndroidDriver driver = new AndroidDriver<>(new URL("http://localhost:4723/wd/hub"), capabilities);

        try {
// there's some screens we need to navigate through and ensure there are no existing photos
            setupAppState(driver);

// set up the file we want to push to the phone's library
            File assetDir = new File(classpathRoot, "../assets");
            File img = new File(assetDir.getCanonicalPath(), "cloudgrey.png");

// actually push the file
            driver.pushFile(ANDROID_PHOTO_PATH + "/" + img.getName(), img);

// wait for the system to acknowledge the new photo, and use the WebDriverWait to verify
// that the new photo is there
            WebDriverWait wait = new WebDriverWait(driver, 10);
            ExpectedCondition condition = ExpectedConditions.numberOfElementsToBe(photo,1);
            wait.until(condition);
        } finally {
            driver.quit();
        }
    }

    public void setupAppState(AndroidDriver driver) {
// navigate through the google junk to get to the app
        WebDriverWait wait = new WebDriverWait(driver, 10);
        WebDriverWait shortWait = new WebDriverWait(driver, 3);
        wait.until(ExpectedConditions.presenceOfElementLocated(backupSwitch)).click();
        wait.until(ExpectedConditions.presenceOfElementLocated(touchOutside)).click();
        wait.until(ExpectedConditions.presenceOfElementLocated(keepOff)).click();

// delete any existing pictures using an infinite loop broken when we can't find any
// more pictures
        try {
            while (true) {
                shortWait.until(ExpectedConditions.presenceOfElementLocated(photo)).click();
                shortWait.until(ExpectedConditions.presenceOfElementLocated(trash)).click();
                shortWait.until(ExpectedConditions.presenceOfElementLocated(moveToTrash)).click();
            }
        } catch (TimeoutException ignore) {}
    }
}
```
