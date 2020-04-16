## 不通过XPath如何在iOS上定位元素

通过多种定位策略中的一种来定位要在Appium测试中使用的元素。Appium继承了Selenium的许多定位策略，不同的是，Appium driver还添加了一些新的定位策略，以便更快，更有效地定位元素。让我们探究一下所有可用的定位策略，以及当前iOS驱动程序（[XCUITest driver](https://appium.io/docs/en/drivers/ios-xcuitest/)）支持哪些定位策略：
<table> 
    <thead> 
        <tr> 
             <th>Locator Strategy</th> 
             <th>From</th> 
             <th>Available?</th> 
        </tr> 
    </thead> 
    <tbody>
        <tr> 
             <td><code>class name</code></td> 
             <td>Selenium</td> 
             <td>Yes</td> 
        </tr> 
        <tr> 
             <td><code>id</code></td> 
             <td>Selenium</td> 
             <td>Yes</td> 
        </tr> 
        <tr> 
             <td><code>name</code></td> 
             <td>Selenium</td> 
             <td>Yes</td> 
        </tr> 
        <tr> 
             <td><code>xpath</code></td> 
             <td>Selenium</td> 
             <td>Yes</td> 
        </tr> 
        <tr> 
             <td><code>accessibility id</code></td> 
             <td>Appium</td> 
             <td>Yes</td> 
        </tr> 
        <tr> 
             <td><code>-ios predicate string</code></td> 
             <td>Appium</td> 
             <td>Yes</td> 
        </tr> 
        <tr> 
             <td><code>-ios class chain</code></td> 
             <td>Appium</td> 
             <td>Yes</td> 
        </tr> 
        <tr> 
             <td><code>css selector</code></td> 
             <td>Selenium</td> 
             <td>No</td> 
        </tr> 
        <tr> 
             <td><code>link text</code></td> 
             <td>Selenium</td> 
             <td>No</td> 
        </tr> 
        <tr> 
             <td><code>partial link text</code></td> 
             <td>Selenium</td> 
             <td>No</td> 
        </tr> 
        <tr> 
             <td><code>tag name</code></td> 
             <td>Selenium</td> 
             <td>No</td> 
        </tr> 
        <tr> 
             <td><code>-ios uiautomation</code></td> 
             <td>Appium</td> 
             <td>No</td> 
        </tr> 
        <tr> 
             <td><code>-android uiautomator</code></td> 
             <td>Appium</td> 
             <td>No</td> 
        </tr> 
    </tbody>
</table>

因此，对于iOS测试，我们可以使用7种定位策略，包括臭名昭著的[XPath](https://en.wikipedia.org/wiki/XPath)。 在研究替代方案之前，让我们探讨为什么使用XPath定位策略很有吸引力，以及为什么它经常是一个坏主意。

### 为什么是（不是）XPath
XPath之所以吸引人，是因为它是基于层次结构的定位策略，因此可以在DOM（Selenium）或app层次结构（对于Appium）中找到任何元素。通常，App以这样的形式出现：没有对元素进行注释，出于可测试性目的，通常应该对元素进行注释。依靠XPath，使用类似 ***`//*[1]/*[1]/*[3]/*[2]/*[1]/*[1]`*** 的选择器很容易找到元素，尤其是在看起来没有其他方法可以找到该元素的情况下。

当然，依赖这样的选择器是一个可怕的主意，因为对App层次结构所做的任何改动，几乎都会使选择器无效。依赖元素的层次结构会带来麻烦，因为App层次结构远远称不上稳定。它可能在运行时（例如，因为显示了不同的数据）或在App版本之间（例如：App开发人员添加了新组件，或者只是更改了基础UI库）变动。

XPath和XCUITest driver一起使用，这也是一个可怕的想法，因为XPath可能会非常慢。XPath及其表亲XML不是iOS开发的本机语言。为了通过XPath访问元素，Appium团队不得不执行一系列技术魔术。本质上，每次执行XPath查询时，都必须递归遍历整个App层次结构并将其序列化为XML。这本身会花费很多时间，尤其是在你的App包含许多元素的情况下。然后，在序列化的XML上执行XPath查询后，必须将匹配到的元素列表反序列化为实际的元素对象，然后才能将其WebDriver表示形式返回给客户端调用。

总之，XPath是一个大麻烦。 即便使用更好，更严格的选择器（例如： ***`//XCUIElementTypeButton[@name="Foo"]`***）避免了典型的XPath陷阱，你仍将承担生成XML文档的开销。 如果你在使用XPath和XCUITest driver时遇到了速度慢的问题，那可能是因为你的App包含太多元素，就像我描述的那样，是效率低下的主要原因。有什么解决办法？不要使用XPath！

当然，如果您可以使用 ***id***，***name***，或者 ***accessibility id*** 直接获得元素的句柄，那么这将是首选。但是，如果没有与元素关联的唯一id或标签，并且XPath太慢了，请考虑使用 ***-ios predicate string*** 或 ***-ios class chain*** 定位策略。

### iOS Predicate String 策略
[Predicate Format Strings](https://developer.apple.com/library/archive/documentation/Cocoa/Conceptual/Predicates/Articles/pSyntax.html)（字符串断言？）
是Apple开发工具，它们也可在iOS中使用。Predicate format strings可进行基本比较和匹配。对我们而言，它允许根据简单的标准对元素进行基本匹配。Predicate strings真正有用的是，你可以组合简单的条件以形成更复杂的匹配项。在XCUITest driver中，Predicate strings可用于匹配各种元素属性，包括 ***name***, ***value***, ***label***, ***type***, ***visible*** 等。

[WebDriverAgent predicate string guide](https://github.com/facebookarchive/WebDriverAgent/wiki/Predicate-Queries-Construction-Rules)中的一个例子展示了一个有趣的组合断言： 
```
type == 'XCUIElementTypeButton' AND value BEGINSWITH[c] 'bla' AND visible == 1
```

该predicate string将匹配任何以'bla'开头的可见按钮。我们如何在Java代码中编写这些内容？ 只需使用适当的 ***MobileBy*** 方法，如下所示：
```
String selector = "type == 'XCUIElementTypeButton' AND value BEGINSWITH[c] 'bla' AND visible == 1";
driver.findElement(MobileBy.iOSNsPredicateString(selector));
```

由于predicate匹配内置于[XCUITest](https://developer.apple.com/documentation/xctest/xcuielementquery/1500768-element)中，因此它有比Appium的XPath策略快得多的潜力。

### iOS Class Chain 策略
最后一个选项是XPath和predicate strings之间的一种混合形式：***-ios class chain*** （iOS类链）定位策略。这是由Appium团队开发的，旨在以更高效的方式满足层次查询的需求。通过class chain策略定位可能不如XPath那样强大，但是这种限制意味着更好的性能保证（这是因为可以通过一系列XCUITest直接调用来匹配class chain，而不是必须递归地构建整个UI树）。class chain查询看起来非常类似于XPath查询，但是唯一允许的过滤器是基本的子/后代索引或predicate string匹配。 这值得查阅[class chain docs](https://github.com/facebookarchive/WebDriverAgent/wiki/Class-Chain-Queries-Construction-Rules)文档以找到更多示例。 让我们来看几个：
- ```XCUIElementTypeWindow[2]``` 选择层次结构中的第二个窗口。
- ```XCUIElementTypeWindow[`label BEGINSWITH "foo"`][-1]``` 选择标签以"foo"开头的最后一个窗口。
- ```**/XCUIElementTypeCell[`name BEGINSWITH "C"`]/XCUIElementTypeButton[10]``` 选择名称以"C"开头而且至少有10个直接XCUIElementTypeButton子类的XCUIElementTypeCell的第十个XCUIElementTypeButton

和上面一样，有一个特殊的 ***MobileBy*** 方法可以在Java中使用class chain查询：
```
driver.findElement(MobileBy.iOSClassChain(selector));
```

从这些示例可以看出，class chain策略允许某些相当强大的搜索过滤组合。下次你使用XPath遇到速度慢的时候，请改用此策略，看看是否可以用它来表示你的查询！

### 注意事项
当然，没有天下没有免费的午餐（即使你在初创公司工作！）。这些定位策略的主要缺点是它们不是跨平台的。你必须知道特定元素属性在iOS中如何显示，以使用predicate strings（例如，***label*** 仅适用于iOS）。 然而，在PO或等效模型中适当地分离关注点，你会体会到切换平台特定定位器所带来的性能提升。这由你决定！  