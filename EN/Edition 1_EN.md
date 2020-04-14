## Seeding the iOS simulator with test photos

One of the best things about mobile apps is how much access they have to all the wonderful functionality of mobile devices. Apps can pull in photos taken from the device for their own purposes. But when running automated tests of these applications, the device is always starting from a clean slate, with no photo fixtures on it (or it should be!).

One challenge for the Appium tester is to figure out how to seed the device under test with photos, and not just any photos, but the exact ones whose behavior within the app will have known results, to make verification of that behavior possible.

The trick is to use Appium's *pushFile* command. This is a little-known command that enables pushing of files to the iOS app container. In a special mode, it can also leverage *simctl*'s *addmedia* command under the hood to drop whatever images you want on the device. The *pushFile* command works like this:

```
driver.pushFile("myFileName.jpg", "/path/to/file/on/disk.jpg");
```

In other words, you give the image a name (this is important because iOS uses it to make it the correct image type on the sim), and then you give the driver a path to the actual file on your local system. That's it! Check out the full example below (or view it on GitHub):

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

Note that it's important not to put any slashes in front of the image name, otherwise Appium might think you want to copy it into the app container rather than add it to the media library on the device.
