## 在Android设备中植入测试照片

我们正在尝试解决的问题是如何将具有已知内容的图片下载到设备上以供我们的待测应用程序使用。这是任意App利用或处理图像的要求。

你如何测试你的App对用户提供的图片做了正确的事情呢？
> 1. 拿随便一张你的照片，将它手动上传到你的应用中。
> 2. 使用你的图片手动运行应用功能。例如：功能可能使用了某种类型的过滤器。
> 3. 仍然手动，用任何你可以使用的方式从你的应用程序中提取修改后的图片（例如：发短信给自己）。

这些步骤的目的是提供可供使用的黄金标准的前后对照的测试固件。现在，我们可以以自动化的方式为APP提供相同的初始图片，运行所需的功能，然后检索修改后的图像。我们可以逐字节地验证此修改后的图像是否与黄金标准等效，以确保应用程序功能仍能按预期运行。

在这片文章，我们将关注于如何将初始图片粘贴到设备上的问题。在Android上如何做到这一点呢？幸运的是，我们使用与iOS相同的功能：***pushFile***。在后台，***pushFile***使用一系列***ADB***命令将图片移动到设备上，然后“广播”系统以刷新媒体库。由于不同的设备制造商在不同的地方存放图片，因此你需要知道设备上存储媒体的路径。对于仿真器和某些实际设备，设备上的路径为：***/mnt/sdcard/Pictures***，所以这是一个你需要记住的重要常数。举个例子：
```
driver.pushFile("/mnt/sdcard/Pictures/myPhoto.jpg", "/path/to/photo/locally/myPhoto.jpg");
```

如你所见，第一个参数是我们希望图片最终在设备上的远程路径。这个路径，你可能需要参考特定设备或Android版本的文档，以确保你拥有正确的路径。第二个参数是运行测试的本地计算机上文件的路径。执行测试时，Appium客户端会将这个文件编码为字符串，然后将其发送到Appium服务器，然后由服务器执行工作，将图片获取到设备上。这种架构很不错，因为它意味着无论是在本地运行还是在Sauce Labs等云供应商上运行，***pushFile***都可以工作。

让我们看一个完整的工作示例。在此示例中，我们将自动执行Google内置的图片App。首先，我们设置所需的capabilities以使用内置App：
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

打开App后，我们必须浏览一些UI样板，以及Google希望我们使用它的云进行的其他操作。当然，我们是机器人，对他们的云不感兴趣！我已将App的状态设置放在*setupAppState*方法中，并设置了一些定位器常量，逻辑删除所有现有图片，以便之后进行验证。
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

最后，我们可以进行我们的测试了，这取决于恰当的设备媒体路径常数：
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

如你所见，我们实际关心的点就像调用***pushFile***一样简单。为了进行此测试，我们只在最后验证是否存在我们的图片。在实际情景中，我们将在图片放到App上运行我们功能，然后从设备中检索它以对它执行一些本地验证。
将所有内容放在一起，整个测试类如下所示：
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