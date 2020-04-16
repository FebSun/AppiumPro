## Android App的性能测试

传统上，Appium是用来做移动App的功能验证。我的意思是，人们通常会验证App的的状态，以确保该应用程序的功能正常运行。功能测试非常重要，但是用户体验的其他方面同样值得构建自动化方式进行验证。其中一个维度的是性能。性能是你的App如何响应用户，并且可以包括从网络请求时间到CPU和内存使用情况等各种具体因素。

移动App，比起桌面应用程序，在资源非常有限的环境中运行。移动App在运行时，不仅会给用户带来不良体验，而且还可能占用CPU或内存，从而缩短电池寿命、导致其他App运行缓慢。因此，进行性能测试是使App生态环境良好运行，而不仅仅是确保你自己的用户获得最佳的体验。

幸运的是，对于Appium用户而言，可以通过Appium API获得各种有趣的性能数据——至少在Android上是如此。在通过 ***adb dumpsys*** 获得大量信息的基础上，Appium通过[getPerformanceData命令](https://appium.io/docs/en/commands/device/performance-data/get-performance-data/)，提供了App性能各方面的简单概述。客户端调用很简单：
```
driver.getPerformanceData("<package>", "<perf type>", <timeout>);
```

在这里，***<package>*** 是您的AUT（或任何你要分析的App）软件包。***<perf type>*** 是你想要的性能数据类型。还有另一个方便的驱动程序命令（[getSupportedPerformanceDataTypes](https://appium.io/docs/en/commands/device/performance-data/performance-data-types/)），可以告诉你哪些类型有效。目前，它们是：***cpuinfo*** ，***memoryinfo*** ，***batteryinfo*** 和 ***networkinfo*** 。最后，***<timeout>*** 是一个整数，表示不能立即获得Appium性能数据的轮询秒数。

在这一章中有太多内容需要探讨，因此让我们集中讨论一个内存使用的简单示例。 许多App曾经遭遇过的一个问题是内存泄漏。尽管使用了垃圾收集机制，但Android App仍然会引起内存锁定而无法使用。 因此，测试的重点是，你的APP没有正当理由时，不会使用越来越多的内存

让我们构造一个简单的场景来说明这一点，这很容易移植到你自己的Android App中。在App中打开一个视图，获取内存使用情况的快照，然后等待。过了一会儿，再获取一次快照。然后，断言第二个快照中的内存使用量不会明显大于第一个快照中的内存使用量。这是内存泄漏的一个简单测试。

假设我们使用的是旧版的ApiDemos App，我们需要这样调用：
```
List<List<Object>> data = driver.getPerformanceData("io.appium.android.apis", "memoryinfo", 10);
```

返回的是包含两个列表的集合。一个列表是键，另一个是值。我们可以将上面的调用与一些帮助代码结合在一起，以方便查询所需的特定类型的内存信息（当然，实际应用中，我们可以创建一个类来保存数据）：
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

本质上，我们现在有了一个 ***HashMap*** ，可以用来查询检索到的特定类型的内存信息。在本例中，我们将寻找totalPss值：
```
HashMap<String, Integer> memoryInfo = getMemoryInfo(driver);
int setSize = memoryInfo.get("totalPss");
```

***totalPss*** 的作用是什么？ 根据 ***Android dumpsys*** 文档，PSS的意思是“实际使用的物理内存大小”。

> 这是对你的App RAM（主存）使用情况的一种衡量，其中考虑了跨进程共享页面的情况。进程独占的RAM页面会直接增加PSS值。而和其他进程共享的RAM页面，会与共享量成比例的增加PSS值。例如，两个进程共享的页面相对于单独占用页面，只会增加一半的。

换句话说，这是RAM影响App的有效衡量。这里还有很多更深入的研究，但是这种度量对我们的情况有用。剩下的就是把它与Appium模板合并，并在App真正执行。这是测试用例的完整代码。
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

我衷心建议你为App添加一些自动化性能测试。即使只使用上述的基础测试，即使功能测试没有发现任何错误，你也可以发现会严重影响用户体验的问题。