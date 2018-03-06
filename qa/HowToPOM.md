# Page Object Model (POM)
POM is a design pattern that promotes clean, reusable and readable test code. It follows DRY (Do Not Repeat Yourself) and Single Responsibility principles. Examples in this guide are written in JavaScript and taken from [test-automation-java](https://github.com/opantsjoha/test-automation-java).

Here's an illustration of what our framework structure looks like with POM:

![PageObjectModel](https://github.com/opantsjoha/docs/blob/master/assets/pom.png)

## Clean, Reusable and Readable test code
Before we dive into detail about how to create objects and write clean tests I want to demonstrate the difference between using POM and not using POM.

Here's an example of a test that uses POM:
```js
it('should successfully log in', async function () {
    browser.homePage.clickOnLoginLink();
    browser.loginPage.setEmailFieldValueTo('test@afullstackqa.com');
    browser.loginPage.setPasswordFieldValueTo('secure_password');
    browser.loginPage.clickOnLoginButton();
    expect(browser.accountPage.getUrl()).to.contain('account');
});
```

And this is without POM:
```js
it('should successfully log in', async function () {
    driver.findElement(By.xpath('//a[@href="/login"]')).click();
    driver.findElement(By.id('loginEmailInput'])).sendKeys('test@afullstackqa.com');
    driver.findElement(By.id('loginPasswordInput'])).sendKeys('secure_password');
    driver.findElement(By.xpath('//input[@vale="login"]')).click();
    expect(driver.getUrl()).to.contain('account');
});
```
Yes we could pull out the [locators](#locators) on to separate lines but that would make the code bulkier. The real challenge starts when we have multiple tests like this and someone has decided to rename `loginEmailInput` id to e.g. `emailId` then we'd need to update all the tests where as with POM we'd only need to do this in one place (DRY principle).

## Base Page Factory (Page Factory)
Please note that I use words Base Page Factory and Page Factory interchangeably, they are the same thing.
This contains most common selenium webdriver functions that will be used by different Page Objects:

```js
function pageFactory(driver) {
  const page = {};

  page.click = async function (locator) {
    const element = await driver.findElement(locator);
    element.click();
    return;
  };

  page.sendKeys = async function (locator, value) {
    const element = await driver.findElement(locator);
    element.sendKeys(value);
    return;
  }

  return page;
}
module.exports = pageFactory;
```
By [passing in driver](#instantiating-page-objects) to our page factory we can use it's methods and create low level functions like the above which can then be re-used by our page objects. Moreover, because we're also passing in locator parameter the searching of the element is done in a single place.

```js
page.click = async function (locator) {
    const element = await driver.findElement(locator);
    element.click();
    return;
};
```
Take the click function as an example; this takes a [locator](#locators) as a parameter, then `driver.findElement()` function finds the element on the page we called it from. We then assign it to element variable and execute 'click()' function on that element.

## Creating Page Objects (factories)
We can take home page as an example. We need to import base page object `const pageFactory = require('../page-factory');` to be able to use our global page functions such as `click()`.

```js
const {By, Key} = require('selenium-webdriver');
const pageFactory = require('../page-factory');

// selectors for elements on this page
const LOGIN_BUTTON = '//a[@href="/login"]';
const ADDRESS_FIELD = '//input[contains(@class, "address-field")]';
const CLEAR_ADDRESS_ICON = '//input[contains(@class,"form-control address-field")]/following-sibling::div[1]/a[@href="#"]';
const SEARCH_BUTTON = '//a[contains(@class, "search-button")]';

function homePageFactory(driver) {
  const homePage = pageFactory(driver);

  // locators for elements on this page
  homePage.loginButton = By.xpath(LOGIN_BUTTON);
  homePage.addressField = By.xpath(ADDRESS_FIELD);
  homePage.clearAddressIcon = By.xpath(CLEAR_ADDRESS_ICON);
  homePage.searchButton = By.xpath(SEARCH_BUTTON);

  homePage.clickOnLoginButton = async function () {
    await homePage.click(homePage.loginButton);
  }

  homePage.setAddressFieldValueTo = async function (value) {
    await homePage.click(homePage.addressField);
    await homePage.sendKeys(homePage.addressField, value);
    await homePage.wait(1000);
    await homePage.sendKeys(homePage.addressField, Key.DOWN);
    await homePage.sendKeys(homePage.addressField, Key.ENTER);
  }

  homePage.clickOnSearchAddressButton = async function () {
    await homePage.click(homePage.searchButton);
  }

  return homePage;
}
module.exports = homePageFactory;
```

### selectors
These are like addresses of the elements on the specific page. Each page has it's own set of elements. As an example the two below are elements of home page.
```js
const LOGIN_BUTTON = '//a[@href="/login"]';
const ADDRESS_FIELD = '//input[contains(@class, "address-field")]';
``` 

### locators
There are several different ways to specify how an element should be located on a page. This the current list from By class:
```js
.className()
.css()
.id()
.js()
.linkText()
.name()
.partialLinkText()
.tagName()
.xpath()
```
Below is an example from home page.
```js
homePage.loginButton = By.xpath(LOGIN_BUTTON);
homePage.addressField = By.xpath(ADDRESS_FIELD);
```
### Page specific actions
Each page will have it's own specific actions such as clicking on a login button. If the action applies to all pages then it should be in the base page.

In order for us to use base page functions in e.g. home page, we need to instantiate a page factory instance like so `  const homePage = pageFactory(driver);`. You will also notice we are [passing in the driver](#instantiating-page-objects) to the page factory.

We can then create our high level functions that will be used in tests:
```js
homePage.clickOnLoginButton = async function () {
    await homePage.click(homePage.loginButton);
}
```

## Browser (Factory)
Browser factory is responsible for [instantiation of page objects](#instantiating-page-objects) as well as browser itself. The instantiation of the browser occurs in the test file itself, only those pages that are used in the test file will be instantiated.

```js
const sel = require('selenium-webdriver');
const browserCapabilitiesFactory = require('./browser-capabilities-factory');
const browserCaps = browserCapabilitiesFactory();
const homePageFactory = require('../pages/home-page/home-page-factory');
const BASE_URL = process.env.BASE_URL || 'http://www.afullstackqa.com';
const SELENIUM_REMOTE_URL = process.env.SELENIUM_REMOTE_URL || 'http://localhost:4444/wd/hub';
// You can change the target browser at runtime through the SELENIUM_BROWSER environment variable.
const SELENIUM_BROWSER = process.env.SELENIUM_BROWSER || 'chrome';

function browserFactory() {
  const browser = {};

  let caps;
  switch (SELENIUM_BROWSER) {
    case 'chrome':
      caps = browserCaps.getChromeCapabilities();
      break;
    case 'firefox':
      break;
    case 'safari':
      break;
    case 'ie':
      break;
    default:
      console.error(`Unhandled switch condition for ${SELENIUM_BROWSER}`);
  }

  browser.driver = new sel.Builder()
    .withCapabilities(caps)
    .usingServer(SELENIUM_REMOTE_URL)
    .build();

  browser.navigate = async function () {
    await browser.driver.get(BASE_URL);
  }

  browser.quit = async function () {
    await browser.driver.quit();
  }

  let _homePage = null;

  browser.homePage = function () {
    if (!_homePage) {
      _homePage = homePageFactory(browser.driver);
    }
    return _homePage;
  };

  return browser;
}
module.exports = browserFactory;
```
### Instantiating page objects
We first need to import our home page factory like so `const homePageFactory = require('../pages/home-page/home-page-factory');` before it can be instantiated.

Home page object will not be created again if it already exists. Also notice that we pass the driver in to home page factory which in turn is passed in to page factory.
```js
  let _homePage = null;

  browser.homePage = function () {
    if (!_homePage) {
      _homePage = homePageFactory(browser.driver);
    }
    return _homePage;
  };
```
### Selenium WebDriver
It's in the name, the driver allows us to drive through application under test and perform different actions.
```js
browser.driver = new sel.Builder()
    .withCapabilities(caps)
    .usingServer(SELENIUM_REMOTE_URL)
    .build();
```

## Tests
Example of an end-to-end test:
```js
const browserFactory = require('../libs/browser-factory');

  describe('Booking flow tests', function () {
    let browser;

    before(async function () {
      browser = browserFactory();
    });

    beforeEach(async function () {
      await browser.navigate();
    });

    afterEach(async function () {
      await browser.quit();
    });

    it('Book massage starting from home page and not logged in', async function () {
      let address = 'postcode';
      await browser.homePage().clickOnClearAddressIcon();
      await browser.homePage().setAddressFieldValueTo(address);
      await browser.homePage().clickOnSearchAddressButton();
    });

  });
}
```