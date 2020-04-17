## 测试Android App升级

之前，我们探讨了如何测试iOS App升级，而本周我们将在Android上进行相同的操作。 App升级如何处理？作为测试人员，我们的责任不只是测试一个版本的App功能。我们还应该关注用户从一个版本迁移到另一个版本时可能出现的问题。你的客户将升级到最新版本，而不是从头开始安装。为了使App正常运行，可能需要迁移各种现有的用户数据，仅仅运行新版本的功能测试不足以覆盖这些情况。

Appium有一些内置命令来处理这种需求。幸运的是，在Android上比在iOS上更简单！为了演示，让我们回到旧版本的"The App"，v1.0.0。此版本仅包含一个功能：一个小的文本字段，用于保存你在其中编写的内容并回显。碰巧的是，这些数据保存在 ***@TheApp:savedEcho*** key内部。（如何保存这些数据的细节与React Native有关，这并不重要——想象一下你的开发团队有一种方法可以保存在升级之间需要保留的本地用户数据）。

现在，想象一下一个疯狂的开发人员已经决定绝对需要将此键更改为 ***@TheApp:savedAwesomeText*** 。这意味着，如果用户在旧版本的App中保存了一些文本后，简单地升级到App的新版本，则该App在新的存储键下将找不到任何已保存的文本！换句话说，升级将破坏用户对App的体验。在这种情况下，解决方法是在App自身中提供一些迁移代码，该代码将在旧key上查找数据，如果存在数据就将数据移至新key。

这是App开发人员（在本例中为我）的全部责任。假设我像一个优秀的开发人员一样，最初在更改数据存储key后忘记编写迁移代码。然后，我发布v1.0.1。不幸的是，由于忘记了编写迁移代码，该版本包含我上面描述的（缺少文本）错误，尽管作为一个独立版本，它也可以正常工作。最终，我意识到了这个错误，并编写了迁移代码。我无法重新发布v1.0.1，所以我发布了修订版本v1.0.2。

这时，测试人员对我这个开发人员的无能感到恼火，他们希望编写一个自动化测试来证明升级到v1.0.2确实可以工作（当然，他们自己也对没有在发布v1.0.1之前编写这样的升级测试感到无语！）。

那么，就Appium代码而言，我们需要什么呢？ 基本上，我们利用以下两种方法：
```
driver.installApp("/path/to/apk");
// 'activity' is an instance of import io.appium.java_client.android.Activity;
driver.startActivity(activity);
```

这套命令：
1. 用路径（或URL）中给定的App作为installAp参数，替换现有App，这样做会停止运行旧版本App（因此，在安装新应用之前无需显式停止该应用 ）。
2. 使用package名称和任何activity来启动新版本的App。

唯一的缺点是 ***startActivity*** 的参数必须是 ***Activity*** 类型。这里需要一个单独的 ***Activity*** 类，因为有许多类型可以传递给此方法。我们仅对基类感兴趣，以便进行App升级。我们可以按如下所示简单地创建 ***Activity*** 对象：
```
Activity activity = new Activity("com.mycompany.myapp", "MyActivity");
driver.startActivity(activity);
```

对于我们的示例，要创建一个有效的测试，我们当然需要两个应用：我们的原始版本和要升级到的版本。在我们的例子中，是"The App"的v1.0.0和v1.0.2： 
```
private String APP_V1_0_0 = "https://github.com/cloudgrey-io/the-app/releases/download/v1.0.0/TheApp-v1.0.0.apk";
private String APP_V1_0_2 = "https://github.com/cloudgrey-io/the-app/releases/download/v1.0.2/TheApp-v1.0.2.apk";
```

我很乐意在这里把GitHub的下载链接反馈给Appium。假设我们已经把 ***APP_V1_0_0*** 作为 ***app*** capability开始测试了，那么App升级命令的组合如下：
```
driver.installApp(appUpgradeVersion);
Activity activity = new Activity(APP_PKG, APP_ACT);
driver.startActivity(activity);
```

对于完整的测试流程，我们要做的是：
1. 打开v1.0.0
2. 添加我们需要保存的消息并验证它是否存在
3. 升级到v1.0.2
4. 打开应用程序并验证消息是否仍然存在——这证明旧消息已迁移到新key

这是实现完整流程（包括Appium样板）的代码：
```
import io.appium.java_client.MobileBy;
import io.appium.java_client.android.Activity;
import io.appium.java_client.android.AndroidDriver;
import java.io.IOException;
import java.net.URL;
import org.junit.Assert;
import org.junit.Test;
import org.junit.runner.RunWith;
import org.junit.runners.JUnit4;
import org.openqa.selenium.By;
import org.openqa.selenium.remote.DesiredCapabilities;
import org.openqa.selenium.support.ui.ExpectedConditions;
import org.openqa.selenium.support.ui.WebDriverWait;

@RunWith(JUnit4.class)
public class Edition009_Android_Upgrade {

    private String APP_PKG = "io.cloudgrey.the_app";
    private String APP_ACT = "com.theapp.MainActivity";

    private String APP_V1_0_0 = "https://github.com/cloudgrey-io/the-app/releases/download/v1.0.0/TheApp-v1.0.0.apk";
    private String APP_V1_0_1 = "https://github.com/cloudgrey-io/the-app/releases/download/v1.0.1/TheApp-v1.0.1.apk";
    private String APP_V1_0_2 = "https://github.com/cloudgrey-io/the-app/releases/download/v1.0.2/TheApp-v1.0.2.apk";


    private String TEST_MESSAGE = "Hello World";

    private By msgInput = MobileBy.AccessibilityId("messageInput");
    private By savedMsg = MobileBy.AccessibilityId(TEST_MESSAGE);
    private By saveMsgBtn = MobileBy.AccessibilityId("messageSaveBtn");
    private By echoBox = MobileBy.AccessibilityId("Echo Box");

    @Test
    public void testSavedTextAfterUpgrade () throws IOException {
        DesiredCapabilities capabilities = new DesiredCapabilities();

        capabilities.setCapability("platformName", "Android");
        capabilities.setCapability("deviceName", "Android Emulator");
        capabilities.setCapability("automationName", "UiAutomator2");
        capabilities.setCapability("app", APP_V1_0_0);

// change this to APP_V1_0_1 to experience a failing scenario
        String appUpgradeVersion = APP_V1_0_2;

// Open the app.
        AndroidDriver driver = new AndroidDriver(new URL("http://localhost:4723/wd/hub"), capabilities);

        WebDriverWait wait = new WebDriverWait(driver, 10);

        try {
            wait.until(ExpectedConditions.presenceOfElementLocated(echoBox)).click();
            wait.until(ExpectedConditions.presenceOfElementLocated(msgInput)).sendKeys(TEST_MESSAGE);
            wait.until(ExpectedConditions.presenceOfElementLocated(saveMsgBtn)).click();
            String savedText = wait.until(ExpectedConditions.presenceOfElementLocated(savedMsg)).getText();
            Assert.assertEquals(savedText, TEST_MESSAGE);

            driver.installApp(appUpgradeVersion);
            Activity activity = new Activity(APP_PKG, APP_ACT);
            driver.startActivity(activity);

            wait.until(ExpectedConditions.presenceOfElementLocated(echoBox)).click();
            savedText = wait.until(ExpectedConditions.presenceOfElementLocated(savedMsg)).getText();
            Assert.assertEquals(savedText, TEST_MESSAGE);
        } finally {
            driver.quit();
        }
    }
}
```

（请注意，我包含了在带有Bug的App版本中运行测试的选项，这会导致测试失败）