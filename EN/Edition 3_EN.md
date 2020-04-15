## Running arbitrary ADB commands via Appium

Last week, we discussed how to get photos into the Android media library. That was all well and good, but there was a bit of clunkiness in how we set up the conditions for the test. We actually automated the UI in order to remove all the existing pictures at the beginning of the test. Wouldn't it be great if there were a way to do this instantly, as part of test setup? Luckily, there is!

If you're not a big Android person, you might not know about ADB, the "Android Debug Bridge". ADB is a powerful tool provided as part of the Android SDK by Google, that allows running all sorts of interesting commands on a connected emulator or device. One of these commands is ***adb shell***, which gives you shell access to the device filesystem (including root access on emulators or rooted devices). ***adb shell*** is the perfect tool for solving the problem above, because with it we can run the ***rm*** command to remove any existing images from the SD card prior to our test.

For a long time, Appium did not allow running of arbitrary ADB commands. This is because Appium was designed to run in a remote environment, possibly sharing an OS with other services or Appium servers, and potentially many connected Android devices. It would be a huge security hole to give any Appium client the full power of ADB in this context. Recently, the Appium team decided to unlock this functionality behind a special server flag, so that someone running an Appium server could intentionally open up this security hole (in situations where they know they're the only one using the server, for example).

As of Appium 1.7.2, the ***--relaxed-security*** flag is available, so you can now start up Appium like this:
```
appium --relaxed-security
```

With Appium running in this mode, you have access to a new "mobile:" command called "mobile: shell". The Appium "mobile:" commands are special commands that can be accessed using ***executeScript*** (at least until client libraries make a nicer interface for taking advantage of them). Here's how a call to "mobile: shell" looks in Java:
```
driver.executeScript("mobile: shell", <arg>);
```

What is ***arg*** here? It needs to be a JSONifiable object with two keys:

1. command: a String, the command to be run under adb shell
2. args: an array of Strings, the arguments passed to the shell command.

For the purposes of our example, let's say we want to clear out the pictures on the SD card, and that on our device, these are located at ***/mnt/sdcard/Pictures***. If we were running ADB on our own without Appium, we'd accomplish our goal by running:
```
adb shell rm -rf /mnt/sdcard/Pictures/*.*
```

To translate this to Appium's "mobile: shell" command, we simply strip off ***adb shell*** from the beginning, and we are left with:
```
rm -rf /mnt/sdcard/Pictures/*.*
```

The first word here is the "command", and the rest constitute the "args". So we can construct our object as follows:
```
List<String> removePicsArgs = Arrays.asList(
                                  "-rf",
                                  "/mnt/sdcard/Pictures/*.*"
                              );
Map<String, Object> removePicsCmd = ImmutableMap.of(
                                        "command", "rm",
                                        "args", removePicsArgs
                                    );
driver.executeScript("mobile: shell", removePicsCmd);
```

Essentially, we construct an Object that in JSON would look like:
```
{"command": "rm", "args": ["-rf", "/mnt/sdcard/Pictures/*.*"]}
```

We can also retrieve the result of the ADB call, for example if we wish to verify that the directory is now indeed empty:
```
List<String> lsArgs = Arrays.asList("/mnt/sdcard/Pictures/*.*");
Map<String, Object> lsCmd = ImmutableMap.of(
                                "command", "ls",
                                "args", lsArgs
                            );
String lsOutput = (String) driver.executeScript("mobile: shell", lsCmd);
```

The output of our command is returned to us as a String which we can do whatever we want with, including making assertions on it. Putting it all together now, here's a full test that uses Appium to combine the two actions above. Of course, it would be most useful in the context of some other actions, like the ones in the previous edition.
```
import com.google.common.collect.ImmutableMap;
import io.appium.java_client.android.AndroidDriver;
import java.net.MalformedURLException;
import java.net.URL;
import java.util.Arrays;
import java.util.List;
import java.util.Map;
import org.junit.Assert;
import org.junit.Test;
import org.openqa.selenium.remote.DesiredCapabilities;

public class Edition003_Arbitrary_ADB {

    private static String ANDROID_PHOTO_PATH = "/mnt/sdcard/Pictures";

    @Test
    public void testArbitraryADBCommands() throws MalformedURLException {
        DesiredCapabilities capabilities = new DesiredCapabilities();
        capabilities.setCapability("platformName", "Android");
        capabilities.setCapability("deviceName", "Android Emulator");
        capabilities.setCapability("automationName", "UiAutomator2");
        capabilities.setCapability("appPackage", "com.google.android.apps.photos");
        capabilities.setCapability("appActivity", ".home.HomeActivity");

// Open the app.
        AndroidDriver driver = new AndroidDriver<>(new URL("http://localhost:4723/wd/hub"), capabilities);

        try {
            List<String> removePicsArgs = Arrays.asList("-rf", ANDROID_PHOTO_PATH + "/*.*");
            Map<String, Object> removePicsCmd = ImmutableMap
            .of("command", "rm", "args", removePicsArgs);
            driver.executeScript("mobile: shell", removePicsCmd);

            List<String> lsArgs = Arrays.asList("/mnt/sdcard");
            Map<String, Object> lsCmd = ImmutableMap.of("command", "ls", "args", lsArgs);
            String lsOutput = (String) driver.executeScript("mobile: shell", lsCmd);
            Assert.assertEquals("", lsOutput);
        } finally {
            driver.quit();
        }
    }

}
```

In this edition we've explored the power of ADB through a few simple filesystem commands. You can actually do many more useful things than delete files with ADB, so go out there and have fun with it. Let me know if you come up with any ingenious uses! As always, you can check out the code in the context of all its dependencies on GitHub.