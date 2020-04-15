## 通过Appium运行任意ADB命令

上周，我们讨论了如何将照片放入Android媒体库。一切都很好，但是我们设置测试条件的方式有些笨拙。实际上，我们自动化UI，以便在测试开始时删除所有现存图片。如果有一种方法，可以将它作为作为测试设置的一部分立即执行，那不是很好吗？幸运的是，有！

如果您不是Android的大人物，那么您可能不了解ADB，即“ Android调试桥”。ADB是Google提供的功能强大的工具，它是Android SDK的一部分，允许在连接的仿真器或设备上运行各种有趣的命令。这些命令之一是***adb shell***，它使您可以通过Shell对设备的文件系统进行访问（包括在仿真器或有root权限的设备上访问）。***adb shell***是解决上述问题的完美工具，因为在测试之前，我们可以使用***rm***命令SD卡中删除所有图片。

很长时间以来，Appium不允许运行任意ADB命令。这是因为Appium设计为在远程环境中运行，可能与其他服务或Appium服务器以及可能连接的许多Android设备共享操作系统。在这种情况下，让任何Appium客户端都拥有ADB的全部功能将是一个巨大的安全漏洞。最近，Appium团队决定通过一个特殊服务标志来解锁此功能，以便运行Appium服务器的人可以有意地打开此安全漏洞（例如，在他们知道自己是唯一使用该服务器的情况下）。

从Appium 1.7.2开始，***--relaxed-security***标志可用，因此您现在可以像这样启动Appium：
```
appium --relaxed-security
```

当Appium以这种模式运行时，你可以访问名为"mobile:shell"的新"mobile:"命令。 Appium mobile:命令是特殊命令，可以使用***executeScript***的特殊命令。（至少在客户端库创建一个更好的接口以利用它们之前）。在Java中，对"mobile:shell"的调用如下所示：
```
driver.executeScript("mobile: shell", <arg>);
```

这里的arg是什么？ 它必须可JSON序列化的两个键：
1. command：String，要在adb shell下运行的命令
2. args：String数组，传递给shell命令的参数

就我们的示例而言，假设我们要清除SD卡上的图片，而我们设备上的图片位于***/mnt/sdcard/Pictures***。如果我们在没有Appium的情况下运行ADB，则可以通过运行以下命令来实现目标：
```
adb shell rm -rf /mnt/sdcard/Pictures/*.*
```

要将其转换为Appium的"mobile:shell"命令，我们只需去掉开头的***adb shell***，然后剩下：
```
rm -rf /mnt/sdcard/Pictures/*.*
```

这里的第一个单词是"command"，其余的组成"args"。因此，我们可以如下构造我们的对象：
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

本质上，我们使用JSON格式构造一个对象，如下所示：
```
{"command": "rm", "args": ["-rf", "/mnt/sdcard/Pictures/*.*"]}
```

我们还可以校验ADB执行的结果，例如，如果我们希望验证目录现在确实为空：
```
List<String> lsArgs = Arrays.asList("/mnt/sdcard/Pictures/*.*");
Map<String, Object> lsCmd = ImmutableMap.of(
                                "command", "ls",
                                "args", lsArgs
                            );
String lsOutput = (String) driver.executeScript("mobile: shell", lsCmd);
```

命令的输出以String的形式返回给我们，我们可以使用它做任何事情，包括对其进行断言。

现在，使用Appium组合上述两个动作，组成一个完整的测试。当然，在某些其他场景（例如上一章）下，它将是最有用的。
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

在这篇文章中，我们通过简单的文件系统命令探索了ADB的功能。与ADB删除文件相比，实际上你可以做更多有用的事情，因此，去玩一玩ADB吧。
