## How to Find Elements in iOS (Not) By XPath

Finding elements for use in your Appium tests is done by means of one of a number of locator strategies. Appium inherited a number of locator strategies from Selenium, and different Appium drivers have also added some new locator strategies to make finding elements faster and more effective. Let's take a look at all the locator strategies available, and which ones are supported in the current flagship iOS driver (the XCUITest driver):
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

So, for our iOS tests we have access to 7 locator strategies, including the infamous XPath. Before we look at alternatives, let's explore why using the XPath locator strategy is attractive, and why it is often a bad idea regardless.

Why (Not) XPath?
XPath is attractive because it is a hierarchy-based locator strategy, and can thus find any element in the DOM (for Selenium) or app hierarchy (for Appium). Often, apps come to us in such a form that elements are not annotated as they should be for the sake of testability. It's easy to rely on XPath to find an element with a selector like ***`//*[1]/*[1]/*[3]/*[2]/*[1]/*[1]`***, especially when there's seemingly no other way to find that element.

Of course, relying on a selector like this is a horrible idea, because it will be invalidated by just about any change to your app hierarchy. That's the trouble with relying on a hierarchical description of your element---the app hierarchy is typically far from stable. It could change between runs (because different data is displayed, for example), or between versions of your app (because the app devs add a new component, or simply change an underlying UI library).

With the XCUITest driver, it can also be a horrible idea because XPath might be very slow. XPath and its cousin, XML, are not native languages for iOS development. In order to provide access to elements via XPath, the Appium team has had to perform a bunch of technical magic. Essentially, every time you run an XPath query, the entire app hierarchy must be recursively walked and serialized into XML. This by itself can take a lot of time, especially if your app has many elements. Then, once the XPath query has been run on this XML serialization, the list of matching elements must be deserialized into actual element objects before their WebDriver representations are returned to your client call.

In sum, XPath is a mixed bag. Even if you avoid typical XPath pitfalls by using better, more restrictive, selectors (like ***`//XCUIElementTypeButton[@name="Foo"]`***), you will still incur the cost of generating the XML document to begin with. If you've encountered slowness with XPath and the XCUITest driver, it's probably because your app has so many elements it's revealing this core inefficiency I just described. What's the solution? Don't use XPath!

Of course, if you can get a direct handle on your element using the ***id***, ***name***, or ***accessibility id*** strategies, that is to be preferred above all else. Stop reading---you're done! But if you are in a position where there is no unique id or label associated with your element, and XPath has turned out to be too slow, consider using the ***-ios predicate string*** or ***-ios class chain*** locator strategies.

### iOS Predicate String Strategy
Predicate Format Strings are a typical Apple dev thing, and they also work in iOS. Predicate format strings enable basic comparisons and matching. In our case, they allow basic matching of elements according to simple criteria. What's really useful about predicate strings is that you can combine simple criteria to form more complex matches. In the XCUITest driver, predicate strings can be used to match various element attributes, including ***name***, ***value***, ***label***, ***type***, ***visible***, etc...

One example from the WebDriverAgent predicate string guide shows a fun compound predicate:
```
type == 'XCUIElementTypeButton' AND value BEGINSWITH[c] 'bla' AND visible == 1
```

This predicate string would match any visible button whose value begins with 'bla'. How would we write this up in our Java client code? Simply by using the appropriate ***MobileBy*** method, as follows:
```
String selector = "type == 'XCUIElementTypeButton' AND value BEGINSWITH[c] 'bla' AND visible == 1";
driver.findElement(MobileBy.iOSNsPredicateString(selector));
```

Because predicate matching is built into XCUITest, it has the potential to be much faster than Appium's XPath strategy.

### iOS Class Chain Strategy
The final option is a sort of hybrid between XPath and predicate strings: the ***-ios class chain*** locator strategy. This was developed by the Appium team to meet the need of hierarchical queries in a more performant way. The types of queries possible via the class chain strategy are not as powerful as those enabled by XPath, but this restriction means a better performance guarantee (this is because it is possible to map class chain queries into a series of direct XCUITest calls, rather than having to recursively build an entire UI tree). Class chain queries look very much like XPath queries, however the only allowed filters are basic child/descendant indexing or predicate string matching. It's worth checking out the class chain docs to find a number of examples. Let's take a look at just a couple:

- ```XCUIElementTypeWindow[2]``` selects the second window in the hierarchy.
- ```XCUIElementTypeWindow[`label BEGINSWITH "foo"`][-1]``` selects the last window whose label begins with foo.
- ```**/XCUIElementTypeCell[`name BEGINSWITH "C"`]/XCUIElementTypeButton[10]``` selects the 10th child button of the first cell in the tree whose name starts with C and which has at least ten direct children of type XCUIElementTypeButton.
Just as before, there is a special ***MobileBy*** method to hook up your class chain queries in the Java client:
```
driver.findElement(MobileBy.iOSClassChain(selector));
```

As you can tell from these examples, the class chain strategy allows for some quite powerful combinations of search filters. Next time you're experiencing slowness with XPath, reach for this strategy instead and see if you can express your query in its terms!

Caveats
There's no such thing as a free lunch, of course (even if you work in startups!). The main downside to these locator strategies is that they are not cross-platform. You have to know how the particular element attributes show up in iOS-land to use predicate strings (for example, ***label*** is an iOS-only concept). However, with appropriate separation of concerns in a page object model or equivalent, you might appreciate the performance gain of switching to one of these platform-specific locators. It's up to you!
