## 使用深层链接加快测试速度

功能测试涉及的最大困难之一是功能测试可能很慢。某些功能测试框架比其他的要快一些，我强烈认为Appium并不是像我所希望的那样快速。速度本身又是另一个话题！本章主要涉及如何在测试中构建“捷径”，来降低自动化速度的整体重要性。

“捷径”是什么意思？让我们以登录App为例。大多数App都需要某种登录步骤。在你的测试脚本可以使用它们的所有功能之前，必须先登录。并且，如果你希望保持测试的原子性，那个每个测试都必须分别登录。这是巨大的时间浪费。你只需要在测试登录功能时，在登录屏幕上运行Appium命令。对于其他所有测试，最好将入口放置在身份验证后面的某个地方并直接开展业务。

我们可以通过称为“深层链接”的东西在iOS和Android上完成此操作。“深层链接”是可以与特定App关联的特殊URL。该App的开发人员通过OS注册一个独特的URL方案，该App将开始运行，并且将能够根据URL内容执行想要的任何操作。

那么，我们需要的是某种特殊的URL，App会将其解释为登录尝试。当App检测到已被该URL激活时，它将在后台执行登录，并重定向到登录成功界面。

我最近花了一些时间在"The App"中添加了深层链接。我实际做的事情已经完全超出了本章的范围，而且全部特定于React Native App。在网上有很多教程可为你的iOS或Android应用添加深层链接支持。但是我最终在v1.2.1上获得了对如下所示的深层链接的支持：
```
theapp://login/<username>/<password>
```

换句话说，在设备上安装了"The App"之后，我们可以使用表单直接导航到该URL，然后"The App"会被唤醒并在后台执行用户名/密码组合的身份验证请求，立即跳转到已登录界面。棒极了！现在我们如何用Appium做同样的事情？

这很简单：只需使用 ***driver.get*** 。 由于Appium Client库仅扩展了Appium Client库，因此我们可以使用与Selenium导航到浏览器URL相同的命令。在我们的例子中，URL的概念只是……更大。假设我们使用用户名 ***darlene*** 和密码 ***test123*** 登录。我们要做的就是构造一个测试，通过被测App打开一个会话，并运行以下命令：
```
driver.get("theapp://login/darlene/testing123");
```

当然，这取决于App开发者需要开发响应这些URL并执行正确操作的功能。你还要确保用于测试的URL在生产版本中不能打开！但是值得将它们构建到你的App中辅助测试，或者说服开发人员进行。能够设置任意的测试状态是缩短测试时间的最佳方式。在我自己对"The App"进行的实验中，我发现通过这种方式绕过登录，每次测试可节省5-10秒。这些节省加起来！

下面是一个完整的例子。它展示了测试登录是否正常工作的两种方法：首先通过使用Appium导航登录UI，然后使用此文章中介绍的深度链接技巧。它还展示了一个非常简单的PO模型，以尽可能多地复用代码。你还会注意到，由于我的App是跨平台的，因此我能够以最小的代码差异在iOS和Android上运行测试。
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