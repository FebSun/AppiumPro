## 在iOS模拟器中植入测试照片

关于移动App最好的事情之一是，他们对移动设备的所有出色功能拥有多少访问权限。APP可以出于自己的目的从设备中提取照片。但是在运行这些APP的自动化测试时，设备始终从干净的状态启动，设备中没有照片（或者它应该有）。

Appium测试人员面临的一项挑战是：如何在被测设备中插入图片，不是任意照片，而是在APP中其行为具有已知结果的确切照片，以使该行为的验证成为可能。 

诀窍是使用Appium的 ***pushFile*** 命令。这是一个鲜为人知的命令，可用于将文件推送到iOS容器中。在特殊模式下，它也可以利用 ***simctl*** 的 ***addmedia*** 命令在后台删除设备上任何你想删除的图片。***pushFile*** 命令的工作方式如下：
```
driver.pushFile("myFileName.jpg", "/path/to/file/on/disk.jpg");
```

换句话说，给图像起一个名字（这很重要，因为iOS使用图像名使其成为sim卡上正确的图像类型），然后为驱动程序提供了指向本地系统上实际文件的路径。就是这样！查看下面的完整示例：
```
import io.appium.java_client.MobileBy;
import io.appium.java_client.ios.IOSDriver;
import java.io.File;
import java.io.IOException;
import java.net.URL;
import java.util.List;
import org.junit.Test;
import org.junit.runner.RunWith;
import org.junit.runners.JUnit4;
import org.openqa.selenium.WebElement;
import org.openqa.selenium.remote.DesiredCapabilities;
import org.openqa.selenium.support.ui.ExpectedConditions;
import org.openqa.selenium.support.ui.WebDriverWait;

@RunWith(JUnit4.class)
public class Edition001_iOS_Photos {
    @Test
    public void testSeedPhotoPicker () throws IOException, InterruptedException {
        DesiredCapabilities capabilities = new DesiredCapabilities();

        File classpathRoot = new File(System.getProperty("user.dir"));
        File appDir = new File(classpathRoot, "../apps/");
        File app = new File(appDir.getCanonicalPath(), "SamplePhotosApp.app.zip");

        capabilities.setCapability("platformName", "iOS");
        capabilities.setCapability("deviceName", "iPhone 8 Plus");
        capabilities.setCapability("platformVersion", "11.2");
        capabilities.setCapability("app", app);

        // Open the app.
        IOSDriver driver = new IOSDriver<>(new URL("http://localhost:4723/wd/hub"), capabilities);

        try {
            // first allow the app to access photos on the phone
            driver.switchTo().alert().accept();

            // navigate to the photo view and count how many there are
            WebDriverWait wait = new WebDriverWait(driver, 10);
            WebElement el = wait.until(ExpectedConditions.presenceOfElementLocated(MobileBy.AccessibilityId("All Photos")));
            el.click();

            List<WebElement> photos = driver.findElements(MobileBy.className("XCUIElementTypeImage"));
            int numPhotos = photos.size();

            // set up the file we want to push to the phone's library
            File assetDir = new File(classpathRoot, "../assets");
            File img = new File(assetDir.getCanonicalPath(), "cloudgrey.png");

            // push the file -- note that it's important it's just the bare basename of the file
            driver.pushFile("pano.jpg", img);

            // in lieu of a formal verification, simply print out the new number of photos, which
            // should have increased by one
            photos = driver.findElements(MobileBy.className("XCUIElementTypeImage"));
            System.out.println("There were " + numPhotos + " photos before, and now there are " +
            photos.size() + "!");

        } finally {
            driver.quit();
        }
    }
}
```

请注意，不要在图像名称前添加任何斜杠，否则Appium可能会认为您要将其复制到APP容器中，而不是将其添加到设备上的媒体库中，这一点很重要。


**simctl是iOS模拟器命令行管理工具，simctl与安卓的adb命令非常相似**

