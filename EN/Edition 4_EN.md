## Using Appium for Testing Mobile Web Apps

Most people associate Appium with native mobile apps. That's how Appium started, and it's still Appium's core value proposition. However, Appium allows you to test a variety of apps---not just native. There are basically three kinds of mobile apps:
1. ***Native apps***. These are apps which are written and built using the OS-provided SDK and native APIs. Users get these apps from an official app store and run them by tapping on the app icon from the home screen of their mobile device.
2. ***Web apps***. These are apps which are written in HTML/JS/CSS and deployed via a web server. Users access them by navigating to a URL in the mobile browser of their choice (e.g., Safari, Chrome).
3. ***Hybrid apps***. These are apps which mix the previous two modes into the same app. Hybrid apps have a native "shell", which might involve a fair amount of native UI, or it might involve no native UI at all. Stuck into one or more of the app screens is a UI component called a "web view", which is essentially a chromeless embedded web browser. These web views can access local or remote URLs. Some hybrid apps build the entire set of app functionality in HTML/JS/CSS which are bundled with the native app package, and retrieved locally by the webview. Other hybrid apps access remote URLs. There are a lot of possibilities for hybrid app architecture. Regardless of how they're set up, hybrid apps have two modes---native and web. Appium can access both! (More on hybrid apps in a future edition).

Appium enables you to automate any kind of app, across both iOS and Android platforms. The only difference is in how you set up the desired capabilities, and then in the commands you have access to once the session is started. This is where Appium's dependence on the WebDriver protocol really shines: an Appium-based mobile web test is just the same thing as a Selenium test! In fact, you can even use a standard Selenium client to speak to the Appium server and automate a mobile web app. The key is to use the ***browserName*** capability instead of the ***app*** capability. Appium will then take care of launching the specified browser and getting you automatically into the web context so you have access to all the regular Selenium methods you're used to (like finding elements by CSS, navigating to URLs, etc...).

The nice thing about testing mobile apps is that there is no difference between platforms in how your test is written. In the same way that you would expect the code for your Selenium tests to remain the same, regardless of whether you're testing Firefox or Chrome, your Appium web tests remain the same regardless of whether you're testing Safari on iOS or Chrome on Android. Let's take a look at what the capabilities would look like to start a session on these two mobile browsers:
```
// test Safari on iOS
DesiredCapabilities capabilities = new DesiredCapabilities();
capabilities.setCapability("platformName", "iOS");
capabilities.setCapability("platformVersion", "11.2");
capabilities.setCapability("deviceName", "iPhone 7");
capabilities.setCapability("browserName", "Safari");

// test Chrome on Android
DesiredCapabilities capabilities = new DesiredCapabilities();
capabilities.setCapability("platformName", "Android");
capabilities.setCapability("deviceName", "Android Emulator");
capabilities.setCapability("browserName", "Chrome");
```

From here on out, after we've launched the appropriate ***RemoteWebDriver*** session (or ***IOSDriver*** or ***AndroidDriver*** if you have the Appium client), we can do whatever we want. For example, we could navigate to a website and get its title!
```
driver.get("https://appiumpro.com");
String title = driver.getTitle();
// here we might verify that the title contains "Appium Pro"
```

Special notes for iOS real devices
The only caveat to be aware of is with iOS real devices. Because of the way Appium talks to Safari (via the remote debugger exposed by the browser) an extra step is required to translate the WebKit remote debug protocol to Apple's iOS web inspector protocol exposed by usbmuxd. Sound complicated? Thankfully, the good folks at Google have created a tool that enables this translation, called ios-webkit-debug-proxy (IWDP). For running Safari tests on a real device, or hybrid tests on a real device, IWDP must be installed on your system. For more info on how to do that, you can check out the Appium IWDP doc. Once you've got IWDP installed, you simply need to add one two more capabilities to the set for iOS above, ***udid*** and ***startIWDP***:
```
// extra capabilities for Safari on a real iOS device
capabilities.setCapability("udid", "<the id of your device>");
capabilities.setCapability("startIWDP", true);
```

If you dont include the ***startIWDP*** capability, you must run IWDP on your own and Appium will just assume it's there listening for proxy requests.

***Note that running tests on real iOS devices is a whole topic in and of itself, which I will cover in more detail in a future edition of Appium Pro. Meanwhile you can refer to the real device documentation available at the Appium docs site.***

Special notes for Android
Thankfully, things are simpler in the Android world because for both emulators and real devices Appium can take advantage of Chromedriver. When you want to automate any Chrome-based browser or webview, Appium simply manages a new Chromedriver process under the hood, so you get the full power of a first-class WebDriver server without having to set anything up yourself. This does mean, however, that you may need to ensure that the version of Chrome on your Android system is compatible with the version of Chromedriver used by Appium. If it's not, you'll get a pretty obvious error message saying you need to upgrade Chrome.

If you don't want to upgrade Chrome, you can actually tell Appium which version of Chromedriver you want installed, when you install Appium, using the ***--chromedriver_version*** flag. For example:
```
npm install -g appium --chromedriver_version="2.35"
```

How do you know which version of Chromedriver to install? The Appium team maintains a helpful list of which versions of Chromedriver support which versions of Chrome at our Chromedriver guide in the docs.

A full example
Let's take advantage of the fact that we can run web tests on both iOS and Android without any code changes, and construct a scenario for testing the Appium Pro contact submission form. We want to make sure that if a user tries to submit it without going through the Captcha challenge, an appropriate error message is presented. Here's the codefor the actual test logic that we care about:
```
driver.get("http://appiumpro.com/contact");
wait.until(ExpectedConditions.visibilityOfElementLocated(EMAIL))
    .sendKeys("foo@foo.com");
driver.findElement(MESSAGE).sendKeys("Hello!");
driver.findElement(SEND).click();
String response = wait.until(ExpectedConditions.visibilityOfElementLocated(ERROR)).getText();

// validate that we get an error message involving a captcha, which we didn't fill out
Assert.assertThat(response, CoreMatchers.containsString("Captcha"));
```

You can see that I've put some selectors into class variables for readability. This is basically all we need to have a useful test! Though of course we do need some boilerplate to set up the desired capabilities for iOS and Android. The full test file looks like:
```
import io.appium.java_client.AppiumDriver;
import io.appium.java_client.android.AndroidDriver;
import io.appium.java_client.ios.IOSDriver;
import java.net.MalformedURLException;
import java.net.URL;
import org.hamcrest.CoreMatchers;
import org.junit.Assert;
import org.junit.Test;
import org.openqa.selenium.By;
import org.openqa.selenium.remote.DesiredCapabilities;
import org.openqa.selenium.support.ui.ExpectedConditions;
import org.openqa.selenium.support.ui.WebDriverWait;

public class Edition004_Web_Testing {

    private static By EMAIL = By.id("contactEmail");
    private static By MESSAGE = By.id("contactText");
    private static By SEND = By.cssSelector("input[type=submit]");
    private static By ERROR = By.cssSelector(".contactResponse");

    @Test
    public void testAppiumProSite_iOS() throws MalformedURLException {
        DesiredCapabilities capabilities = new DesiredCapabilities();
        capabilities.setCapability("platformName", "iOS");
        capabilities.setCapability("platformVersion", "11.2");
        capabilities.setCapability("deviceName", "iPhone 7");
        capabilities.setCapability("browserName", "Safari");

// Open up Safari
        IOSDriver driver = new IOSDriver<>(new URL("http://localhost:4723/wd/hub"), capabilities);
        actualTest(driver);
    }

    @Test
    public void testAppiumProSite_Android() throws MalformedURLException {
        DesiredCapabilities capabilities = new DesiredCapabilities();
        capabilities.setCapability("platformName", "Android");
        capabilities.setCapability("deviceName", "Android Emulator");
        capabilities.setCapability("browserName", "Chrome");

// Open up Safari
        AndroidDriver driver = new AndroidDriver(new URL("http://localhost:4723/wd/hub"), capabilities);
        actualTest(driver);
    }

    public void actualTest(AppiumDriver driver) {
// Set up default wait
        WebDriverWait wait = new WebDriverWait(driver, 10);

        try {
            driver.get("http://appiumpro.com/contact");
            wait.until(ExpectedConditions.visibilityOfElementLocated(EMAIL))
            .sendKeys("foo@foo.com");
            driver.findElement(MESSAGE).sendKeys("Hello!");
            driver.findElement(SEND).click();
            String response = wait.until(ExpectedConditions.visibilityOfElementLocated(ERROR)).getText();

// validate that we get an error message involving a captcha, which we didn't fill out
            Assert.assertThat(response, CoreMatchers.containsString("Captcha"));
        }
        finally {
            driver.quit();
        }

    }
}
```

This is all you need to get started with web testing in Appium. As always, you can find the source for this edition in a working repo on GitHub.
