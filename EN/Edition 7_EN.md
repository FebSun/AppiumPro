## Speeding Up Tests With Deep Links

One of the biggest difficulties involved in functional testing is that functional tests can be slow. Some functional testing frameworks are faster than others, and I'll be the first to admit that Appium isn't always as speedy as I would like. Speed itself is a topic for another time! This edition is all about how to make automation speed less important overall, by building shortcuts into your testing.

What do I mean by shortcuts? Let's take logging in to an app as an example. Most apps require some kind of login step. Before your tests can access all the functionality they're supposed to exercise, they must first log in. And if you are keeping your tests atomic, as you should, every test has to log in separately. This is a huge waste of time. You only need to run Appium commands on your login screen when you are actively testing the login functionality. For every other test, it'd be great to just land somewhere behind the authentication barrier and get directly to business.

We can accomplish this on iOS and Android with something called a deep link. Deep links are special URLs that can be associated with specific apps. The developer of the app registers a unique URL scheme with the OS, and from then on, the app will come alive and will be able to do whatever it wants based on the content of the URL.

What we need, then, is some kind of special URL that our app interprets as a login attempt. When it detects that it has been activated with that URL, it will perform the login behind the scenes and redirect the app to the correct logged-in view.

I recently spent some time adding deep link interpretation into The App. Exactly how I did that is beyond the scope of this edition, and is specific to React Native apps in any case. There are plenty of tutorials online for adding deep link support to your iOS or Android app. But what I ended up with in v1.2.1 is support for a deep link that looks like this:
```theapp://login/<username>/<password>
```

In other words, with The App installed on a device, we can direct the device to navigate to a URL with that form, and The App will wake up and perform authentication of the requested username/password combination behind the scenes, dumping the user to the logged-in area instantly. Great! Now how do we do the same thing with Appium?

It's easy: just use ***driver.get***. Since the Appium client libraries just extend the Selenium ones, we can use the same command that Selenium would use to navigate to a browser URL. In our case, the concept of URL is just ... bigger. So let's say we want to log in with the username ***darlene*** and the password ***testing123***. All we have to do is construct a test that opens a session with our App Under Test, and runs the following command:
```
driver.get("theapp://login/darlene/testing123");
```

Of course, it's up to the app developer to build the ability to respond to URLs and perform the appropriate action with them. And you probably want to make sure that URLs used for testing are not turned on in production versions of your app! But it's worth building them into your app for testing, or convincing your developer to. Being able to set up arbitrary test state is the best way to cut off significant portions of time off your testing. In my own experimentation with The App, I noticed a savings of 5-10s on each test just by bypassing the login screen in this way. These savings add up!

Here is a full example. It shows two ways of testing that the logged-in experience works correctly: first by navigating the login UI with Appium, and then by using the deep linking trick described in this edition. It also shows an example of what an extremely simple inline Page Object Model might look like, as a way to reuse code as much as possible. You'll also notice that because my app is cross-platform, I'm able to run these tests on both iOS and Android with minimal code difference.
```
import io.appium.java_client.AppiumDriver;
import io.appium.java_client.MobileBy;
import io.appium.java_client.android.AndroidDriver;
import io.appium.java_client.ios.IOSDriver;
import java.io.IOException;
import java.net.URL;
import org.junit.Test;
import org.junit.runner.RunWith;
import org.junit.runners.JUnit4;
import org.openqa.selenium.By;
import org.openqa.selenium.remote.DesiredCapabilities;
import org.openqa.selenium.support.ui.ExpectedConditions;
import org.openqa.selenium.support.ui.WebDriverWait;

@RunWith(JUnit4.class)
public class Edition007_Deep_Linking {

    private String APP_IOS = "https://github.com/cloudgrey-io/the-app/releases/download/v1.2.1/TheApp-v1.2.1.app.zip";
    private String APP_ANDROID = "https://github.com/cloudgrey-io/the-app/releases/download/v1.2.1/TheApp-v1.2.1.apk";

    private String AUTH_USER = "alice";
    private String AUTH_PASS = "mypassword";

    @Test
    public void testLoginSlowIOS() throws IOException {
        IOSModel model = new IOSModel();
        IOSDriver driver = new IOSDriver<>(new URL("http://localhost:4723/wd/hub"), model.caps);
        runStepByStepTest(driver, model);
    }

    @Test
    public void testLoginSlowAndroid() throws IOException {
        AndroidModel model = new AndroidModel();
        AndroidDriver driver = new AndroidDriver(new URL("http://localhost:4723/wd/hub"), model.caps);
        runStepByStepTest(driver, model);
    }

    private void runStepByStepTest(AppiumDriver driver, Model model) {
        WebDriverWait wait = new WebDriverWait(driver, 10);

        try {
            wait.until(ExpectedConditions.presenceOfElementLocated(model.loginScreen)).click();
            wait.until(ExpectedConditions.presenceOfElementLocated(model.username)).sendKeys(AUTH_USER);
            wait.until(ExpectedConditions.presenceOfElementLocated(model.password)).sendKeys(AUTH_PASS);
            wait.until(ExpectedConditions.presenceOfElementLocated(model.loginBtn)).click();
            wait.until(ExpectedConditions.presenceOfElementLocated(model.getLoggedInBy(AUTH_USER)));
        }
        finally {
            driver.quit();
        }
    }

    @Test
    public void testDeepLinkForDirectNavIOS () throws IOException {
        IOSModel model = new IOSModel();
        IOSDriver driver = new IOSDriver<>(new URL("http://localhost:4723/wd/hub"), model.caps);
        runDeepLinkTest(driver, model);
    }

    @Test
    public void testDeepLinkForDirectNavAndroid () throws IOException {
        AndroidModel model = new AndroidModel();
        AndroidDriver driver = new AndroidDriver(new URL("http://localhost:4723/wd/hub"), model.caps);
        runDeepLinkTest(driver, model);
    }

    private void runDeepLinkTest(AppiumDriver driver, Model model) {
        WebDriverWait wait = new WebDriverWait(driver, 10);

        try {
            driver.get("theapp://login/" + AUTH_USER + "/" + AUTH_PASS);
            wait.until(ExpectedConditions.presenceOfElementLocated(model.getLoggedInBy(AUTH_USER)));
        }
        finally {
            driver.quit();
        }
    }

    private abstract class Model {
        public By loginScreen = MobileBy.AccessibilityId("Login Screen");
        public By loginBtn = MobileBy.AccessibilityId("loginBtn");
        public By username;
        public By password;

        public DesiredCapabilities caps;

        abstract By getLoggedInBy(String username);
    }

    private class IOSModel extends Model {
        IOSModel() {
            username = By.xpath("//XCUIElementTypeTextField[@name=\"username\"]");
            password = By.xpath("//XCUIElementTypeSecureTextField[@name=\"password\"]");

            caps = new DesiredCapabilities();
            caps.setCapability("platformName", "iOS");
            caps.setCapability("deviceName", "iPhone 7");
            caps.setCapability("platformVersion", "11.2");
            caps.setCapability("app", APP_IOS);
        }

        public By getLoggedInBy(String username) {
            return MobileBy.AccessibilityId("You are logged in as " + username);
        }
    }

    private class AndroidModel extends Model {
        AndroidModel() {
            username = MobileBy.AccessibilityId("username");
            password = MobileBy.AccessibilityId("password");

            caps = new DesiredCapabilities();
            caps.setCapability("platformName", "Android");
            caps.setCapability("deviceName", "Android Emulator");
            caps.setCapability("app", APP_ANDROID);
            caps.setCapability("automationName", "UiAutomator2");
        }

        public By getLoggedInBy(String username) {
            return By.xpath("//android.widget.TextView[@text=\"You are logged in as " + username + "\"]");
        }
    }
}
```