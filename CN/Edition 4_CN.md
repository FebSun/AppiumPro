## 使用Appium测试移动Web应用

大多数人将Appium与原生App相关联。这是Appium最初的功能，现在仍然是Appium的核心价值主张。但是，Appium允许测试各种应用程序-不仅仅是原生。基本上有三种移动应用程序：
1. ***原生App*** (***Native***)。这些是使用操作系统提供的SDK和原生API编写和构建的应用程序。用户从官方应用商店中获取这些应用，然后在移动设备主屏幕上点击应用图标来运行它们。
2. ***Web App*** 。这些是用HTML/JS/CSS编写并通过网络服务器部署的应用程序。用户通过在自己选择的移动浏览器（eg: Safari，Chrome）中导航到一个URL来访问它们。
3. ***混生App*** 。这些是将前两种模式混合到一起的App。混生App具有原生“外壳”，它可能包含大量的原生UI, 或者可能根本不包含原生UI。在一个或多个App屏幕中嵌入一个名为“ Web视图”的UI组件，该组件本质上是无边框的嵌入式Web浏览器。这些Web视图可以访问本地或远程的URL. 一些混生App使用HTML/JS/CSS构建完整的App功能集，这些功能与原生应用程序包捆绑在一起，并通过Webview在本地检索。一些混生App访问远程URL。混生App体系结构有很多可能。无论如何设置，混生App都有两种模式-原生和Web。Appium可以同时访问！ 

Appium使你可以跨iOS和Android平台，自动化测试任何类型的App。唯一的区别在于你如何设置所需的capabilities，以及在会话启动后，访问所用的命令。这正是Appium对WebDriver协议的依赖的真正体现：基于Appium的移动Web测试与Selenium测试一样！实际上，您甚至可以使用标准的Selenium客户端与Appium服务器对话，并自动化移动Web App。关键是使用 ***browserName*** capability而不是 ***app*** capability。然后，Appium将负责启动指定的浏览器并自动进入Web上下文（***context***），以便可以使用的所有常规Selenium方法（例如，通过CSS查找元素，导航至URL等）。

测试移动App的好处是，在不同的平台上编写测试用例没有任何不同。就像你希望无论在测试Firefox还是Chrome时，Selenium测试的代码保持不变一样。无论是在iOS的Safari上，还是在Android的Chrome上，Appium的Web测试代码都保持不变。让我们看一下在这两个移动浏览器上启动会话的capabilities是什么样的：
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

从这里开始，在启动适当的 ***RemoteWebDriver*** 会话（如果您具有Appium客户端，则启动 ***IOSDriver*** 或 ***AndroidDriver*** ）之后，我们可以做任何我们想做的事情。例如，我们可以导航到网站并获取其标题！
```
driver.get("https://appiumpro.com");
String title = driver.getTitle();
// here we might verify that the title contains "Appium Pro"
```

iOS真机的特别说明

iOS真机唯一需要注意的警告是。由于Appium与Safari对话的方式（通过浏览器公开的远程调试器），需要额外的步骤将WebKit远程调试协议转换为usbmuxd公开的Apple的iOS Web检查器协议（Apple's iOS web inspector protocol）。听起来复杂吗？值得庆幸的是，Google的小伙伴们创建了一个启用此转译的工具，称为[ios-webkit-debug-proxy](https://github.com/google/ios-webkit-debug-proxy)（IWDP）。要在真机上运行Safari测试或混生测试，必须在系统上安装IWDP。有关如何操作的更多信息，您可以查看[Appium IWDP文档](https://appium.io/docs/en/writing-running-appium/web/ios-webkit-debug-proxy/)，安装完IWDP之后，您只需要在上述iOS集合中添加一个或两个capability，***udid*** 和 ***startIWDP*** ：
```
// extra capabilities for Safari on a real iOS device
capabilities.setCapability("udid", "<the id of your device>");
capabilities.setCapability("startIWDP", true);
```

如果没有包括 ***startIWDP*** capability，则必须自己运行IWDP，而Appium只会假定它在那里侦听代理请求。

***请注意，在iOS真机上运行测试本身就是一个完整的主题，我将在以后的文章中对其进行详细介绍。同时，您可以参考Appium文档站点上的[real device documentation](https://appium.io/docs/en/drivers/ios-xcuitest-real-devices/)***

Android的特别说明

幸运的是，在Android中事情变得更加简单，因为对于模拟器和真机，Appium都可以利用[Chromedriver](https://sites.google.com/a/chromium.org/chromedriver/)。当你想自动化任何基于Chrome的浏览器或Webview时，Appium只需在后台管理一个新的Chromedriver进程，即可获得WebDriver服务的全部功能，而无需自己进行任何设置。但是，这意味着你需要确保Android系统上的Chrome版本与Appium使用的Chromedriver版本兼容。如果不兼容，则会收到一条很明显的错误消息，提示你需要升级Chrome。

如果不想升级Chrome，实际上可以在安装Appium时使用 ***--chromedriver_version*** 标志告诉Appium你要安装的Chromedriver版本。例如：
```
npm install -g appium --chromedriver_version="2.35"
```

您如何知道要安装哪个版本的Chromedriver？Appium团队在[Chromedriver guide](https://appium.io/docs/en/writing-running-appium/web/chromedriver/)中维护了一个有用的列表，其中列出了哪些版本的Chromedriver支持哪些版本的Chrome。

一个完整的例子
我们来挑战一下：在iOS和Android上运行Web测试，而无需进行任何代码更改，并构建一个场景来测试Appium Pro的联系人表单提交功能。我们要确保，如果用户没有通过验证码校验进行提交，会显示正确的错误消息。这是我们关心的实际测试逻辑的代码：
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

您可以看到，为了便于阅读，我将一些选择器放入类变量中。这基本上是进行一个有用的测试，我们需要做的全部事情！ 当然，尽管我们确实需要一些样板才能为iOS和Android设置所需的capabilities。 完整的测试文件如下所示：
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

这是开始在Appium中进行Web测试所需的全部内容。
