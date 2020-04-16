## 测试iOS App升级

本周，我们回到iOS的有趣的iOS世界，讨论App升级测试。许多测试人员的一个共同要求超出了常规的App功能测试。有幸，你的团队能够经常将新版本的App发布给客户。这意味着你的客户正在升级到最新版本，而不是从头开始安装。为了使App正常运行，可能需要迁移各种现有的用户数据，仅仅运行新版本的功能测试不足以覆盖这些情况。

幸运的是，最新版本的Appium 1.8 Beta版本现在支持一些命令，可以在单个Appium session中轻松重新安装和重新启动App。这提供了测试本地升级的可能性。

为了演示这一点，我想介绍"The App"，它是一个跨平台的React Native演示App，我构建这个App来支持我们在AppiumPro中遇到的各种示例

我最近发布了The App v1.0.0，它仅包含一个功能：一个小的文本字段，用于保存你在其中编写的内容并回显。碰巧的是，使用这些数据通过 ***@TheApp:savedEcho*** 键保存了在内部。

现在，在这个玩具App中，想象一下一个疯狂的开发人员已经决定绝对需要将此键更改为 ***@TheApp:savedAwesomeText*** 。这意味着，如果用户在旧版本的App中保存了一些文本后，简单地升级到App的新版本，则该App在新的存储键下将找不到任何已保存的文本！换句话说，升级将破坏用户对App的体验。在这种情况下，解决方法是在App自身中提供一些迁移代码，该代码将在旧key上查找数据，如果存在数据就将数据移至新key。

这是App开发人员（即我）的全部责任。让假设我忘记编写此迁移代码，继续进行并更改了key值。然后，我发布了v1.0.1。此版本包含由于遗忘了迁移代码而导致文本丢失的Bug，尽管作为一个独立版本，它也可以正常工作。让我们继续设，我慌忙编写了迁移代码并将其发布为v1.0.2。现在，对我质疑的勇敢的测试人员想要编写一个自动化测试来证明升级到v1.0.2确实可以工作（当然，他们自己也非常慌张，他们没有在发布v1.0.1之前编写这样的升级测试）。

那么，就Appium代码而言，我们需要什么呢？ 基本上，我们利用这三种 ***mobile:*** 方法：
```
driver.executeScript("mobile: terminateApp", args);
driver.executeScript("mobile: installApp", args);
driver.executeScript("mobile: launchApp", args);
```

这组命令将停止当前App（我们需要明确执行此操作，以便底层WebDriverAgent实例不会认为该应用程序已崩溃），在其之上安装新App，然后启动该新App。

***args*** 是什么样的？ 在任何情况下，它们都应该是 ***HashMap***，可以用于生成简单的JSON结构。***TerminateApp*** 和 ***launchApp*** 命令都具有一个键 ***bundleId*** （当然，这是App的包ID）。 ***installApp*** 使用 ***app*** key，该key是新版本App的路径或URL

对于我们的示例，要创建一个通过的测试，我们需要两个应用程序：v1.0.0和v1.0.2：
```
private String APP_V1_0_0 = "https://github.com/cloudgrey-io/the-app/releases/download/v1.0.0/TheApp-v1.0.0.app.zip";
private String APP_V1_0_2 = "https://github.com/cloudgrey-io/the-app/releases/download/v1.0.2/TheApp-v1.0.2.app.zip";
```

我很乐意在这里使用GitHub下载URL反馈给Appium。 假设我们已经把 ***APP_V1_0_0*** 作为 ***app*** capability开始测试了，那么App升级的三个命令的如下：
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

对于完整的测试流程，我们希望：
1. 打开v1.0.0
2. 添加我们保存的消息并验证它是否存在
3. 升级到v1.0.2
4. 打开应用程序并验证消息是否仍然存在——这证明旧消息已迁移到新key

这是代码（请注意，我包含了在带有Bug的App版本中运行测试的选项，这会导致测试失败）：
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

仅此而已，即可在iOS上测试App升级！ 