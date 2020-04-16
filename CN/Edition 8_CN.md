## How to Find Elements in iOS (Not) By XPath

Finding elements for use in your Appium tests is done by means of one of a number of locator strategies. Appium inherited a number of locator strategies from Selenium, and different Appium drivers have also added some new locator strategies to make finding elements faster and more effective. Let's take a look at all the locator strategies available, and which ones are supported in the current flagship iOS driver (the XCUITest driver):
<table class="table"> 
    <style>
        .class={
            margin: 0px auto;
        }
    </style>>
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

