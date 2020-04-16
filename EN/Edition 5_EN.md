## Performance Testing of Android Apps

Traditionally, Appium is used to make functional verifications of mobile apps. What I mean is that people usually check the state of an app's UI to ensure that the app's features are working correctly. Functional testing is very important, but there are other dimensions of the user experience that are equally worth checking via automated means in your build. One of these other dimensions is performance. Performance is simply how responsive your app is to the user, and can include a variety of specific factors, from network request time to CPU an d memory usage.

Mobile apps, more than desktop apps, run in very resource-constrained environments. Mobile apps also have the potential not only to create a bad experience for the user while the app is open, but by hogging CPU or memory, could shorten the battery life or cause other applications to run slowly. Engaging in performance testing is therefore a way to be a good citizen of the app ecosystem, not just to ensure the snappiest experience for your own users.

Luckily for Appium users, all kinds of interesting performance data is available to you via the Appium API---well, on Android at least. Building on the wealth of information that comes via ***adb dumpsys***, Appium provides a succinct overview of all aspects of your app's performance via the getPerformanceData command. The client call is simple:
```
driver.getPerformanceData("<package>", "<perf type>", <timeout>);
```

Here, ***<package>*** is the package of your AUT (or any other app you wish to profile). ***<perf type>*** is what kind of performance data you want. There is another handy driver command (getSupportedPerformanceDataTypes) which tells you which types are valid. For the time being, they are: ***cpuinfo***, ***memoryinfo***, ***batteryinfo***, and ***networkinfo***. Finally, ***<timeout>*** is an integer denoting the number of seconds Appium will poll for performance data if it is not immediately available.

There's too much to dig into in one edition, so let's focus on a simple example involving memory usage. One problem encountered by many apps at some point in their history is a memory leak. Despite living in a garbage-collected environment, Android apps can still cause memory to be locked up in an unusable state. It's therefore important to test that your app is not using increasing amounts of memory over time, without good reason.

Let's construct a simple scenario to illustrate this, which is easily portable to your own Android apps. Basically, we want to open up a view in our app, take a snapshot of the memory usage, and then wait. After a while, we take another snapshot. Then we make an assertion that the memory usage outlined in the second snapshot is not significantly more than what we found in the first snapshot. This is a simple test that we don't have a memory leak.

Assuming we're using the good old ApiDemos app, our call now looks like:
```
List<List<Object>> data = driver.getPerformanceData("io.appium.android.apis", "memoryinfo", 10);
```

What is returned is a set of two lists; one list is the keys and the other is the values. We can bundle up the call above along with some helper code that makes it easier to query the specific kind of memory info we're looking for (of course, in the real world we might make a class to hold the data):
```
private HashMap<String, Integer> getMemoryInfo(AndroidDriver driver) throws Exception {
    List<List<Object>> data = driver.getPerformanceData("io.appium.android.apis", "memoryinfo", 10);
    HashMap<String, Integer> readableData = new HashMap<>();
    for (int i = 0; i < data.get(0).size(); i++) {
        int val;
        if (data.get(1).get(i) == null) {
            val = 0;
        } else {
            val = Integer.parseInt((String) data.get(1).get(i));
        }
        readableData.put((String) data.get(0).get(i), val);
    }
    return readableData;
}
```

Essentially, we now have a ***HashMap*** we can use to query the particular kinds of memory info we retrieved. In our case, we're going to look for the ***totalPss*** value:
```
HashMap<String, Integer> memoryInfo = getMemoryInfo(driver);
int setSize = memoryInfo.get("totalPss");
```

What is this ***totalPss*** business? PSS means 'Proportional Set Size'. According to the Android ***dumpsys*** docs:

> This is a measurement of your app’s RAM use that takes into account sharing pages across processes. Any RAM pages that are unique to your process directly contribute to its PSS value, while pages that are shared with other processes contribute to the PSS value only in proportion to the amount of sharing. For example, a page that is shared between two processes will contribute half of its size to the PSS of each process.

In other words, it's a pretty good measurement of the RAM impact of our app. There's a lot more to dig into here, but this measurement will do for our case. All that remains is for us to tie this into the Appium boilerplate and make a real assertion on our app. Here's the full code for the test case we outlined above (also available on GitHub):
```
import io.appium.java_client.android.AndroidDriver;
import java.io.File;
import java.net.URL;
import java.util.HashMap;
import java.util.List;
import org.hamcrest.Matchers;
import org.junit.Assert;
import org.junit.Test;
import org.openqa.selenium.remote.DesiredCapabilities;

public class Edition005_Android_Memory {

    private static int MEMORY_USAGE_WAIT = 30000;
    private static int MEMORY_CAPTURE_WAIT = 10;
    private static String PKG = "io.appium.android.apis";
    private static String PERF_TYPE = "memoryinfo";
    private static String PSS_TYPE = "totalPss";

    @Test
    public void testMemoryUsage() throws Exception {
        File classpathRoot = new File(System.getProperty("user.dir"));
        File appDir = new File(classpathRoot, "../apps/");
        File app = new File(appDir.getCanonicalPath(), "ApiDemos.apk");

        DesiredCapabilities capabilities = new DesiredCapabilities();
        capabilities.setCapability("platformName", "Android");
        capabilities.setCapability("deviceName", "Android Emulator");
        capabilities.setCapability("automationName", "UiAutomator2");
        capabilities.setCapability("app", app);

        AndroidDriver driver = new AndroidDriver(new URL("http://localhost:4723/wd/hub"), capabilities);
        try {
// get the usage at one point in time
            int totalPss1 = getMemoryInfo(driver).get(PSS_TYPE);

// then get it again after waiting a while
            try { Thread.sleep(MEMORY_USAGE_WAIT); }
            catch (InterruptedException ign) {}
            int totalPss2 = getMemoryInfo(driver).get(PSS_TYPE);

// finally, verify that we haven't increased usage more than 5%
            Assert.assertThat((double) totalPss2, Matchers.lessThan(totalPss1 * 1.05));
        } finally {
            driver.quit();
        }
    }

    private HashMap<String, Integer> getMemoryInfo(AndroidDriver driver) throws Exception {
        List<List<Object>> data = driver.getPerformanceData(PKG, PERF_TYPE, MEMORY_CAPTURE_WAIT);
        HashMap<String, Integer> readableData = new HashMap<>();
        for (int i = 0; i < data.get(0).size(); i++) {
            int val;
            if (data.get(1).get(i) == null) {
                val = 0;
            } else {
                val = Integer.parseInt((String) data.get(1).get(i));
            }
            readableData.put((String) data.get(0).get(i), val);
        }
        return readableData;
    }
}
``` 

I heartily recommend throwing some automated performance testing in the mix for your app. With even a basic test like the one above, you could catch issues that greatly affect your user's experiences, even if a functional test wouldn't have noticed anything wrong. Stay tuned for more discussion on performance testing or other kinds of testing made possible by Appium, in future editions.
