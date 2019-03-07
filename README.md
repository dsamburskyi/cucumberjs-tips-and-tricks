# CucumberJS tips and tricks
Here are some solutions for challenges I've been facing working with CucumberJS (Potractor-Cucumber setup).
I don't claim this to be the perfect solutions, but it suites well my needs. Constructive feedback is welcome :)

## Make CucumberJS continue running scenario after failed step
By default, Cucumber stops Scenario on first failed step. This suites Unit test philosophy, but isn't handy for longer UI tests.
To handle this, I've written a simple script to monkey-patch cucumber npm module and make it continue test execution on step failure.
The script is located in `scripts/patchCucumber.js` in my project folder. If you want to put it into different location, make sure to change the path to `test_case_runner.js` file.
The content of `scripts/patchCucumber.js` is:
```javascript {.line-numbers}
const fs = require('fs');
const cucumberLib = __dirname + '/../node_modules/cucumber/lib/runtime/test_case_runner.js';
let cucumberFile = fs.readFileSync(cucumberLib, { encoding: 'UTF-8' });
if (
  !cucumberFile.match('this.result.status !== _status.default.PASSED && this.result.status !== _status.default.FAILED')
) {
  cucumberFile = cucumberFile.replace(
    'this.result.status !== _status.default.PASSED',
    'this.result.status !== _status.default.PASSED && this.result.status !== _status.default.FAILED',
  );
  fs.writeFile(cucumberLib, cucumberFile, () => console.log('Patched CucumberJs'));
} else {
  console.log('CucumberJs has been already patched');
}
```
I've also added following scripts to my `package.json`. 'postinstall' will run the patch after `npm i`. To run it on demand - `npm run patch-cucumber`
```javascript
"scripts": {
  "postinstall": "npm run patch-cucumber",
  "patch-cucumber": "node scripts/patchCucumber.js"
}
  ```

## Add screenshot to CucumberJS json report after assertion
You can attach screenshots to Cucumber json report calling `world.attach()`.
The simplest way is to add this to `After` hook, but that will be running after Scenario, not after Step (if you have several validation steps in a scenario, the screenshot will be taken only once after all steps within given Scenario). And currently CucumberJS doesn't have built-in AfterStep hook.
To have the screenshots for every assertion, I wrapped `expect` into a function. For Protractor this looks like the following
```javascript
export async function expectEqlWithScreenshot(world: World, actual, expected) {
  try {
    expect(actual).to.eql(expected);
    world.attach(await browser.takeScreenshot(), 'image/png');
  } catch (e) {
    world.attach(await browser.takeScreenshot(), 'image/png');
    throw e;
  }
}
```
This captures the screenshot on passed assertion too (in case I want to see html report with more details).

Then in the step definition I pass `this` as the first argument to the function. Make sure to use anonymous function `function(){}` in Then(), not arrow function, because in `() => {}`, `this` will refer to outer scope.
```javascript
Then('something is {string}', function(expectedText) {
    await expectEqlWithScreenshot(this, await page.getText(), expectedText);
});
```
