# Playwright Test Options --- Complete Practical & Interview-Ready Notes

## 1. Big Picture

Think of Playwright Test Options in layers:

``` text
playwright.config.ts
        ↓
Global test-run settings
(workers, retries, timeout, projects)

Project configuration
        ↓
Browser / device / environment settings
(Chromium, Firefox, mobile, baseURL)

test.describe()
        ↓
Group related tests

test.describe.configure()
        ↓
Group execution settings
(mode, retries, timeout)

test.use()
        ↓
Browser / Browser Context settings

test(...)
        ↓
Individual test

expect(...)
        ↓
Assertion behaviour
```

The goal is **not to memorize every option**.

Understand these five questions:

1.  Where is the option configured?
2.  What does it control?
3.  Does it affect one test, a group, a project, or the whole run?
4.  What happens when something fails?
5.  When would you use it in a real project?

### One-line memory hook

> **Config controls the run, project controls the environment, describe
> controls a group, test controls one test, and expect checks the
> result.**

------------------------------------------------------------------------

# 2. `playwright.config.ts` --- MUST KNOW

This is the main configuration file for Playwright Test.

It controls how the complete test suite runs.

``` ts
import { defineConfig } from '@playwright/test';

export default defineConfig({
  testDir: './tests',
  timeout: 30_000,
  retries: 1,
  workers: 2,
  fullyParallel: true,

  use: {
    browserName: 'chromium',
    baseURL: 'https://example.com',
    headless: true
  }
});
```

### Common configuration options

  -----------------------------------------------------------------------
  Option                              Simple meaning
  ----------------------------------- -----------------------------------
  `testDir`                           Where Playwright finds test files

  `testMatch`                         Which files should be treated as
                                      tests

  `testIgnore`                        Which files should be ignored

  `timeout`                           Maximum time for one test

  `expect.timeout`                    Default time for retrying web-first
                                      assertions

  `retries`                           How many times a failed test can be
                                      retried

  `workers`                           Maximum number of worker processes
                                      running tests at the same time

  `fullyParallel`                     Allows tests in the same file as
                                      well as across files to run in
                                      parallel

  `forbidOnly`                        Fails the run if `test.only()` is
                                      present

  `maxFailures`                       Stops the run after a specified
                                      number of test failures

  `globalTimeout`                     Maximum time allowed for the entire
                                      test run

  `projects`                          Different
                                      browser/device/environment
                                      configurations

  `reporter`                          How test results are displayed

  `outputDir`                         Where test output files are stored

  `use`                               Default browser and context options
                                      for tests
  -----------------------------------------------------------------------

### Important

Do not put every option inside `use`.

``` ts
export default defineConfig({
  workers: 4,             // Test-run option

  use: {
    browserName: 'chromium', // Browser option
    viewport: { width: 1280, height: 720 }
  }
});
```

### Mental model

``` text
playwright.config.ts
       |
       +-- Test-run settings
       |      workers, retries, timeout
       |      fullyParallel, globalTimeout
       |
       +-- Projects
       |      Chromium, Firefox, mobile
       |
       +-- use
              Browser and context settings
```

------------------------------------------------------------------------

# 3. `test.use()` --- MUST KNOW

Used to configure browser or Browser Context options for tests.

``` ts
test.use({
  browserName: 'chromium',
  viewport: { width: 1280, height: 720 },
  locale: 'en-US'
});
```

### Common options

  Option                Purpose
  --------------------- ------------------------------------------
  `browserName`         Select Chromium, Firefox, or WebKit
  `viewport`            Set browser viewport size
  `baseURL`             Base URL used by navigation
  `storageState`        Reuse saved authentication/session state
  `locale`              Set browser language and locale
  `timezoneId`          Set browser timezone
  `geolocation`         Set browser location
  `permissions`         Grant browser permissions
  `colorScheme`         Set light or dark mode
  `userAgent`           Set a custom browser user agent
  `extraHTTPHeaders`    Add extra HTTP headers to requests
  `httpCredentials`     Set HTTP authentication credentials
  `ignoreHTTPSErrors`   Ignore HTTPS certificate errors
  `serviceWorkers`      Control service-worker behaviour
  `headless`            Run browser without showing its window
  `actionTimeout`       Default timeout for each action
  `navigationTimeout`   Default timeout for navigation actions

### Example: Configure mobile viewport for a group

``` ts
test.describe('Mobile tests', () => {

  test.use({
    viewport: { width: 375, height: 667 }
  });

  test('Verify mobile layout', async ({ page }) => {
    await page.goto('/');
  });

});
```

### Important scope point

-   In `playwright.config.ts`, `use` sets defaults for the configured
    projects.
-   Inside a test file, `test.use()` can configure options for a
    `describe` group and its tests.
-   A more specific project or group configuration can override
    inherited defaults.
-   `test.use()` is configuration, not a command that changes an
    already-running page.

### Mental model

``` text
test.use()
    ↓
Sets the browser / context environment
    ↓
Viewport, locale, storageState, permissions, etc.
```

> **Memory hook:** `test.use()` = What environment should this test run
> in?

------------------------------------------------------------------------

# 4. `test.describe()` --- MUST KNOW

Groups related tests.

``` ts
test.describe('Login Tests', () => {

  test('Valid login', async ({ page }) => {
    // Test steps
  });

  test('Invalid login', async ({ page }) => {
    // Test steps
  });

});
```

It is also useful for applying common hooks and configuration to a
group.

``` ts
test.describe('Login Tests', () => {

  test.beforeEach(async ({ page }) => {
    await page.goto('/login');
  });

  test('Valid login', async ({ page }) => {});

  test('Invalid login', async ({ page }) => {});

});
```

### Remember

> `test.describe()` = Group related tests.

### Interview point

A `describe` block helps organize tests and scope hooks/configuration.
It does not automatically mean that the tests run in parallel.

------------------------------------------------------------------------

# 5. `test.describe.configure()` --- MUST KNOW

Used to configure execution behaviour for a `describe` group.

``` ts
test.describe('Checkout tests', () => {

  test.describe.configure({
    mode: 'parallel',
    retries: 2,
    timeout: 60_000
  });

  test('Test 1', async ({ page }) => {});

  test('Test 2', async ({ page }) => {});

});
```

Important properties:

``` text
mode
retries
timeout
```

### Mental model

``` text
test.describe.configure()
        ↓
Controls HOW a group of tests executes
```

### Important

-   `mode` controls the group's execution mode.
-   `retries` controls retry attempts for tests in that group.
-   `timeout` controls the test timeout for tests in that group.
-   These are group-level settings, not browser settings.

------------------------------------------------------------------------

# 6. `mode` --- MUST KNOW

Controls how tests in a `describe` group execute.

## Default mode

``` ts
test.describe.configure({
  mode: 'default'
});
```

Tests in the same file normally run in order in the same worker, unless
other configuration changes the execution behaviour.

## Parallel mode

``` ts
test.describe.configure({
  mode: 'parallel'
});
```

Tests in this group can run in parallel, subject to available workers.

Important: Parallel tests should not depend on each other's state or
shared mutable test data.

## Serial mode

``` ts
test.describe.configure({
  mode: 'serial'
});
```

Tests are treated as a dependent group and run one after another.

If a test in the serial group fails: - Subsequent tests in that group
are skipped. - The group is retried together when retries are
configured.

### Do not confuse `mode` and `workers`

``` text
workers
   ↓
How many worker processes can run at the same time?

mode: 'parallel'
   ↓
Can tests in this describe group run in parallel?
```

### Important real-project advice

Serial mode is generally discouraged. Prefer independent tests.

For example, avoid this dependency:

``` text
Test 1: Create customer
       ↓
Test 2: Update that customer
       ↓
Test 3: Delete that customer
```

A better approach is to make each test create or prepare its own
required data.

> **Memory hook:** Default = normal group behaviour, parallel =
> independent tests can overlap, serial = dependent group runs in order.

------------------------------------------------------------------------

# 7. `test.skip()` --- MUST KNOW

Skips a test. The test body does not execute.

``` ts
test.skip('Feature not ready', async ({ page }) => {
  // This test will not run
});
```

### Conditional skip

``` ts
test('Verify feature', async ({ page, browserName }) => {

  test.skip(browserName === 'webkit', 'Feature not supported in WebKit');

  await page.goto('/');

});
```

### Group-level skip

``` ts
test.describe.skip('Not ready tests', () => {

  test('Test 1', async ({ page }) => {});

  test('Test 2', async ({ page }) => {});

});
```

### When to use

Use `skip` when a test is not applicable in a particular environment or
should temporarily not run.

> **Memory hook:** `skip` = Do not execute this test.

------------------------------------------------------------------------

# 8. `test.fixme()` --- MUST KNOW

Marks a test as known to be broken and needing a fix.

Playwright does not execute the test body.

``` ts
test.fixme('Known application bug', async ({ page }) => {
  // This test will not run
});
```

### Conditional fixme

``` ts
test('Verify payment', async ({ page, browserName }) => {

  test.fixme(browserName === 'webkit', 'Known issue in WebKit');

  // Test steps
});
```

### When to use

Use it when the test is known to be broken and needs fixing.

> **Memory hook:** `fixme` = This test is broken; fix it later.

------------------------------------------------------------------------

# 9. `test.fail()` --- MUST KNOW

Marks a test as expected to fail.

**Important:** Playwright still executes the test.

``` ts
test.fail('Known application bug', async ({ page }) => {

  await page.goto('/');

  // Test is expected to fail
});
```

### What happens?

  -----------------------------------------------------------------------
  Actual result                       Playwright result
  ----------------------------------- -----------------------------------
  Test fails as expected              Expected failure

  Test passes unexpectedly            Test is reported as unexpected pass
                                      / failure of the expectation
  -----------------------------------------------------------------------

### When to use

Use it when you want to document a known bug and verify that the test
still fails until the bug is fixed.

> **Memory hook:** `fail` = Run it, but expect it to fail.

------------------------------------------------------------------------

# 10. `test.slow()` --- MUST KNOW

Marks a test as slow.

Playwright triples the test timeout for that test.

``` ts
test('Generate large report', async ({ page }) => {

  test.slow();

  // Long-running test steps

});
```

You can also use a condition and description:

``` ts
test.slow(true, 'This test needs more time');
```

### Example

If the normal test timeout is 30 seconds:

``` text
Normal test timeout = 30 seconds

test.slow()
      ↓

Slow test timeout = 90 seconds
```

### Important

`test.slow()` does not make the application or test execute faster. It
gives the test more time before timeout.

> **Memory hook:** `slow` = Triple the test timeout.

------------------------------------------------------------------------

# 11. Test Timeout --- MUST KNOW

A test timeout controls how long the test is allowed to run.

The default test timeout is **30 seconds**.

### Global test timeout

Configured in `playwright.config.ts`.

``` ts
export default defineConfig({
  timeout: 30_000
});
```

### Individual test timeout

``` ts
test('Long-running test', async ({ page }) => {

  test.setTimeout(60_000);

  // Test steps

});
```

### Describe group timeout

``` ts
test.describe('Payment tests', () => {

  test.describe.configure({
    timeout: 60_000
  });

  test('Payment test', async ({ page }) => {});

});
```

### Change timeout through `testInfo`

``` ts
test('Long-running test', async ({ page }, testInfo) => {

  testInfo.setTimeout(60_000);

});
```

### Important timeout difference

Do not confuse these timeouts:

``` text
Test timeout
    ↓
How long the complete test can take

Action timeout
    ↓
How long an action such as click() or fill() can take

Navigation timeout
    ↓
How long a navigation action can take

Assertion timeout
    ↓
How long a web-first expect() assertion retries

Global timeout
    ↓
How long the entire test run can take
```

### What is included in test timeout?

The test timeout includes: - The test function. - Fixtures set up for
the test. - `beforeEach` hooks.

`beforeAll` and `afterAll` hooks have their own hook timeout.

> **Memory hook:** Test timeout = One test's time limit.

------------------------------------------------------------------------

# 12. Action Timeout --- MUST KNOW

Controls how long an action such as `click()`, `fill()` or `check()` can
take.

By default, action timeout is `0`, which means no separate action
timeout limit is configured.

### Global action timeout

``` ts
export default defineConfig({

  use: {
    actionTimeout: 10_000
  }

});
```

### Individual action timeout

``` ts
await page.getByRole('button', { name: 'Login' }).click({
  timeout: 5_000
});
```

### Practical example

``` text
Click Login
    ↓
Playwright waits for the action to become possible
    ↓
If action does not complete within configured timeout
    ↓
Action timeout error
```

### Important

Action timeout is not the same as test timeout.

A test may have 30 seconds overall, while an individual click has a
5-second timeout.

> **Memory hook:** Action timeout = Time limit for one action.

------------------------------------------------------------------------

# 13. Navigation Timeout --- MUST KNOW

Controls how long a navigation action can take.

### Global navigation timeout

``` ts
export default defineConfig({

  use: {
    navigationTimeout: 30_000
  }

});
```

### Individual navigation timeout

``` ts
await page.goto('https://example.com', {
  timeout: 15_000
});
```

### When to use

Use it when navigation to a page takes longer than expected.

### Important

Navigation timeout is for navigation actions. It is not the timeout for
every action on the page.

> **Memory hook:** Navigation timeout = Time limit for page navigation.

------------------------------------------------------------------------

# 14. Assertion Timeout --- MUST KNOW

Controls how long a web-first assertion retries until the expected
condition is satisfied.

The default timeout for web-first assertions is 5 seconds.

### Global assertion timeout

``` ts
export default defineConfig({

  expect: {
    timeout: 10_000
  }

});
```

### Individual assertion timeout

``` ts
await expect(page.getByText('Welcome')).toBeVisible({
  timeout: 10_000
});
```

### Practical example

``` ts
await expect(page.getByText('Order placed')).toBeVisible();
```

Playwright repeatedly checks the condition until: - The element becomes
visible, or - The assertion timeout is reached.

### Important difference

``` text
locator.click()
    ↓
Actionability checks before action

expect(locator).toBeVisible()
    ↓
Retries the assertion condition until it passes or times out
```

An assertion timeout does not increase the overall test timeout.

> **Memory hook:** Assertion timeout = How long expect keeps checking.

------------------------------------------------------------------------

# 15. Global Timeout --- SHOULD KNOW

Controls the maximum time allowed for the entire test run.

By default, there is no global timeout.

``` ts
export default defineConfig({
  globalTimeout: 60 * 60 * 1000
});
```

This example sets a one-hour limit for the complete test run.

### Example

``` text
Test suite starts
      ↓
Test 1
      ↓
Test 2
      ↓
Test 3
      ↓
Global timeout reached
      ↓
Playwright stops the run
```

### Difference

  Timeout               Controls
  --------------------- -------------------------
  `timeout`             One test
  `actionTimeout`       One action
  `navigationTimeout`   One navigation
  `expect.timeout`      One web-first assertion
  `globalTimeout`       Entire test run

> **Memory hook:** Global timeout = Time limit for the whole suite.

------------------------------------------------------------------------

# 16. `retries` --- MUST KNOW

Controls how many times Playwright retries a failed test.

### Global retries

``` ts
export default defineConfig({
  retries: 2
});
```

### Group-level retries

``` ts
test.describe('Login tests', () => {

  test.describe.configure({
    retries: 2
  });

});
```

### Project-level retries

``` ts
projects: [
  {
    name: 'chromium',
    retries: 2
  }
]
```

### Example

``` text
First attempt → Failed
       ↓
Retry 1 → Failed
       ↓
Retry 2 → Passed
       ↓
Final result → Flaky
```

A test that fails initially but passes on retry is reported as flaky.

### Retry only in CI

``` ts
export default defineConfig({
  retries: process.env.CI ? 2 : 0
});
```

### Important interview points

-   `retries: 2` means up to two retry attempts after the initial
    attempt.
-   Retries do not fix the root cause of flaky tests.
-   A retry can help identify intermittent failures.
-   Use reports and traces to investigate why the first attempt failed.

> **Memory hook:** Retries = How many extra attempts after failure?

------------------------------------------------------------------------

# 17. `workers` --- MUST KNOW

Workers are separate worker processes used by Playwright Test to execute
tests.

``` ts
export default defineConfig({
  workers: 4
});
```

### Mental model

``` text
workers: 1
    ↓
One worker process

workers: 4
    ↓
Up to four worker processes can execute tests concurrently
```

### Important

Worker does not mean browser tab or browser page.

A worker process can run tests using Playwright fixtures and browser
contexts.

### Why configure workers?

-   More workers can reduce total execution time.
-   Too many workers can overload the machine or test environment.
-   Parallel tests must not conflict with shared test data.

### CI example

``` ts
export default defineConfig({
  workers: process.env.CI ? 1 : undefined
});
```

This uses one worker in CI and lets Playwright choose its default
locally.

> **Memory hook:** Workers = How many test workers can work at the same
> time?

------------------------------------------------------------------------

# 18. `fullyParallel` --- MUST KNOW

Configured in `playwright.config.ts`.

``` ts
export default defineConfig({
  fullyParallel: true
});
```

It allows all tests in all test files to run in parallel, subject to the
available workers.

### Understand the default behaviour

Normally: - Different test files can run in parallel. - Tests within the
same file run in order in the same worker.

With `fullyParallel: true`: - Tests within the same file can also run in
parallel. - Tests across files can run in parallel.

### Do not confuse

``` text
workers
    ↓
How many worker processes are available?

fullyParallel
    ↓
Can individual tests be distributed for parallel execution?
```

### Important

`fullyParallel: true` does not mean every test starts at exactly the
same time.

The number of available workers limits how many tests can execute
concurrently.

> **Memory hook:** fullyParallel = Allow individual tests to run in
> parallel, not just files.

------------------------------------------------------------------------

# 19. `forbidOnly` --- MUST KNOW

Prevents accidental focused tests from being committed to CI.

``` ts
export default defineConfig({
  forbidOnly: !!process.env.CI
});
```

### Why is it important?

During local debugging, we may write:

``` ts
test.only('Login test', async ({ page }) => {
  // Test steps
});
```

This runs only the focused test(s).

If `test.only()` is accidentally committed, other tests may not run.

With `forbidOnly: true`, Playwright fails the test run when focused
tests are found.

### Memory hook

> `forbidOnly` = Do not allow `test.only()` in this run.

------------------------------------------------------------------------

# 20. `maxFailures` --- SHOULD KNOW

Controls the maximum number of test failures allowed before Playwright
stops the run.

``` ts
export default defineConfig({
  maxFailures: 5
});
```

### Example

``` text
Test 1 → Failed
Test 2 → Failed
Test 3 → Passed
Test 4 → Failed
Test 5 → Failed
Test 6 → Failed
       ↓
5 failures reached
       ↓
Playwright stops the run
```

By default, `maxFailures` is `0`, which disables this limit.

### When to use

Useful in CI when you want to stop spending time on a suite after too
many tests have failed.

> **Memory hook:** maxFailures = Stop after this many failures.

------------------------------------------------------------------------

# 21. Project Configuration --- MUST KNOW

A project is a logical group of tests running with the same
configuration.

Projects are configured in `playwright.config.ts`.

### Example: Run tests in three browsers

``` ts
projects: [
  {
    name: 'chromium',
    use: {
      browserName: 'chromium'
    }
  },

  {
    name: 'firefox',
    use: {
      browserName: 'firefox'
    }
  },

  {
    name: 'webkit',
    use: {
      browserName: 'webkit'
    }
  }
]
```

The same test suite can run under all three project configurations.

### Projects can also represent

-   Different browsers.
-   Desktop and mobile devices.
-   Different environments such as staging and production.
-   Different authentication states.
-   Different timeout or retry settings.

### Example: Different retry settings

``` ts
projects: [
  {
    name: 'staging',
    use: {
      baseURL: 'https://staging.example.com'
    },
    retries: 2
  },

  {
    name: 'production',
    use: {
      baseURL: 'https://example.com'
    },
    retries: 0
  }
]
```

> **Memory hook:** Project = A named test environment/configuration.

------------------------------------------------------------------------

# 22. Project Dependencies --- SHOULD KNOW

A project can depend on another project.

The dependency project runs first. The dependent project starts only
after its dependency passes.

### Practical example

Imagine you need to log in once and save the authentication state before
running other tests.

``` text
Setup project
     ↓
Save authentication state
     ↓
Chromium tests
Firefox tests
WebKit tests
```

Example configuration:

``` ts
projects: [
  {
    name: 'setup',
    testMatch: /.*\.setup\.ts/
  },

  {
    name: 'chromium',
    use: {
      browserName: 'chromium'
    },
    dependencies: ['setup']
  },

  {
    name: 'firefox',
    use: {
      browserName: 'firefox'
    },
    dependencies: ['setup']
  }
]
```

### Important

If the dependency project fails, the dependent projects do not run.

Project dependencies are also useful for setup and teardown workflows.

> **Memory hook:** Dependency = Run this project before that project.

------------------------------------------------------------------------

# 23. `test.only()` --- MUST KNOW

Runs only the selected test or tests.

``` ts
test.only('Debug this test', async ({ page }) => {
  // Test steps
});
```

### When to use

Useful when debugging one test locally.

### Important

Do not leave `test.only()` in committed code.

Use `forbidOnly: true` in CI to catch accidental focused tests.

> **Memory hook:** `only` = Run only this test.

------------------------------------------------------------------------

# 24. `test.describe.only()` --- SHOULD KNOW

Runs only the selected `describe` block.

``` ts
test.describe.only('Login Tests', () => {

  test('Test 1', async ({ page }) => {});

  test('Test 2', async ({ page }) => {});

});
```

Useful when debugging a complete group of tests.

------------------------------------------------------------------------

# 25. `test.describe.skip()` --- SHOULD KNOW

Skips the complete `describe` group.

``` ts
test.describe.skip('Not ready', () => {

  test('Test 1', async ({ page }) => {});

  test('Test 2', async ({ page }) => {});

});
```

Neither test body will execute.

------------------------------------------------------------------------

# 26. `test.describe.serial()` --- SHOULD KNOW

Creates a serial group.

``` ts
test.describe.serial('Checkout', () => {

  test('Login', async ({ page }) => {});

  test('Add product', async ({ page }) => {});

  test('Checkout', async ({ page }) => {});

});
```

### Important

-   Tests run one after another.
-   If a test fails, subsequent tests are skipped.
-   The group is retried together.
-   Serial mode is discouraged for most test suites.

Prefer independent tests whenever possible.

`test.describe.configure({ mode: 'serial' })` is the preferred
configuration style when you need serial mode.

------------------------------------------------------------------------

# 27. `test.step()` --- SHOULD KNOW

Creates a named step inside a test.

``` ts
test('Login test', async ({ page }) => {

  await test.step('Enter username and password', async () => {
    await page.getByLabel('Username').fill('admin');
    await page.getByLabel('Password').fill('password');
  });

  await test.step('Click login', async () => {
    await page.getByRole('button', { name: 'Login' }).click();
  });

});
```

### Useful for

-   Readable reports.
-   Debugging.
-   Organizing large tests.
-   Understanding which step failed.

### Important

`test.step()` does not create a separate test. It creates a named step
inside the existing test.

> **Memory hook:** `test.step()` = Break one test into meaningful report
> steps.

------------------------------------------------------------------------

# 28. `test.setTimeout()` vs `testInfo.setTimeout()` --- SHOULD KNOW

Both can change the timeout of the current test.

### Using `test.setTimeout()`

``` ts
test('Long test', async ({ page }) => {

  test.setTimeout(60_000);

});
```

### Using `testInfo.setTimeout()`

``` ts
test('Long test', async ({ page }, testInfo) => {

  testInfo.setTimeout(60_000);

});
```

### Remember

-   `test.setTimeout()` is available through the `test` API.
-   `testInfo.setTimeout()` is available through the current test's
    `testInfo` object.
-   Both change the timeout for the current test.

------------------------------------------------------------------------

# 29. Practical Configuration Example --- MUST KNOW

This is a realistic configuration combining several important options.

``` ts
import { defineConfig, devices } from '@playwright/test';

export default defineConfig({

  testDir: './tests',

  timeout: 30_000,

  expect: {
    timeout: 5_000
  },

  fullyParallel: true,

  forbidOnly: !!process.env.CI,

  retries: process.env.CI ? 2 : 0,

  workers: process.env.CI ? 1 : undefined,

  globalTimeout: 60 * 60 * 1000,

  reporter: 'html',

  use: {
    baseURL: 'https://example.com',
    headless: true,
    actionTimeout: 10_000,
    navigationTimeout: 30_000,
    trace: 'on-first-retry'
  },

  projects: [
    {
      name: 'chromium',
      use: {
        ...devices['Desktop Chrome']
      }
    },

    {
      name: 'firefox',
      use: {
        ...devices['Desktop Firefox']
      }
    },

    {
      name: 'webkit',
      use: {
        ...devices['Desktop Safari']
      }
    }
  ]

});
```

### Understand this configuration

  Setting               What it does
  --------------------- -----------------------------------------------
  `timeout`             Each test gets 30 seconds
  `expect.timeout`      Web-first assertions get 5 seconds by default
  `fullyParallel`       Allows tests to run individually in parallel
  `forbidOnly`          Prevents accidental focused tests in CI
  `retries`             Two retries in CI, zero locally
  `workers`             One worker in CI; Playwright default locally
  `globalTimeout`       Stops the complete run after one hour
  `actionTimeout`       Each action gets up to 10 seconds
  `navigationTimeout`   Each navigation gets up to 30 seconds
  `projects`            Runs tests in Chromium, Firefox and WebKit

------------------------------------------------------------------------

# 30. Five Buckets to Remember

``` text
                    PLAYWRIGHT TEST OPTIONS
                             |
        +--------------------+--------------------+
        |                    |                    |
    EXECUTION             TIMEOUTS            ENVIRONMENT
        |                    |                    |
    workers              test timeout          test.use()
    fullyParallel        actionTimeout         browserName
    mode                 navigationTimeout     viewport
    retries              expect.timeout        storageState
    maxFailures          globalTimeout         baseURL
                         slow

        +--------------------+--------------------+
        |                                         |
    TEST CONTROL                           ORGANIZATION
        |                                         |
    skip                                    test.describe
    fixme                                   test.step
    fail                                    projects
    only                                    dependencies
    forbidOnly                              hooks
```

------------------------------------------------------------------------

# 31. Important Interview Differences --- MUST KNOW

## A. `workers` vs `fullyParallel`

  -----------------------------------------------------------------------
  workers                             fullyParallel
  ----------------------------------- -----------------------------------
  Controls maximum concurrent worker  Allows tests within files to run in
  processes                           parallel

  Controls execution capacity         Controls how tests can be
                                      distributed

  Example: `workers: 4`               Example: `fullyParallel: true`
  -----------------------------------------------------------------------

## B. Test timeout vs Assertion timeout

  Test timeout                 Assertion timeout
  ---------------------------- -----------------------------------
  Controls the whole test      Controls one web-first assertion
  Default 30 seconds           Default 5 seconds
  Configured using `timeout`   Configured using `expect.timeout`

## C. Action timeout vs Navigation timeout

  -----------------------------------------------------------------------
  Action timeout                      Navigation timeout
  ----------------------------------- -----------------------------------
  `click()`, `fill()`, etc.           `goto()`, navigation actions

  Configured using `actionTimeout`    Configured using
                                      `navigationTimeout`
  -----------------------------------------------------------------------

## D. `skip` vs `fixme` vs `fail`

  Method    Does test execute?   Meaning
  --------- -------------------- --------------------------------------------
  `skip`    No                   Test is not applicable / should be skipped
  `fixme`   No                   Known broken test needs fixing
  `fail`    Yes                  Test is expected to fail

## E. `test.describe()` vs `test.describe.configure()`

  Method                        Purpose
  ----------------------------- --------------------------------------
  `test.describe()`             Groups related tests
  `test.describe.configure()`   Configures group execution behaviour

## F. `test.only()` vs `forbidOnly`

  Option          Purpose
  --------------- --------------------------------------------
  `test.only()`   Runs only focused tests
  `forbidOnly`    Fails the run if focused tests are present

## G. `test.slow()` vs `test.setTimeout()`

  Method                Purpose
  --------------------- -------------------------------
  `test.slow()`         Triples the test timeout
  `test.setTimeout()`   Sets a specific timeout value

------------------------------------------------------------------------

# 32. Top Practical Interview Scenarios

You should be able to explain and implement these without looking at
notes.

1.  How do you run only Chromium tests?
2.  How do you configure different viewport sizes for desktop and
    mobile?
3.  How do you run tests sequentially in one group but in parallel in
    another?
4.  How do you allow all tests across files and within files to run in
    parallel?
5.  What happens when you configure `workers: 4` and
    `fullyParallel: true`?
6.  How do you prevent `test.only()` from accidentally reaching CI?
7.  How do you skip tests only in WebKit?
8.  What is the difference between `skip`, `fixme` and `fail`?
9.  How do you increase timeout for one particular test?
10. How do you increase timeout for one particular assertion?
11. How do you configure action timeout and navigation timeout?
12. How do you retry failed tests only in CI?
13. What happens when a test fails and retry is enabled?
14. What does Playwright report when a test fails initially but passes
    on retry?
15. How do you stop a test run after five failures?
16. How do you limit the entire test suite to one hour?
17. What happens when one test in a serial group fails?
18. Why is serial execution generally discouraged?
19. How do you configure Chromium, Firefox and WebKit using projects?
20. How do project dependencies work?
21. How would you prevent parallel tests from modifying the same test
    data?
22. What is the difference between test timeout, action timeout and
    assertion timeout?
23. Can a project have its own retry and timeout settings?
24. What is the difference between `test.describe()` and
    `test.describe.configure()`?
25. How would you investigate a test that passes on retry but fails on
    the first attempt?

------------------------------------------------------------------------

# 33. Final Mental Map

Remember this instead of memorizing every option separately.

``` text
PLAYWRIGHT TEST OPTIONS
   |
   +-- CONFIGURATION
   |      |
   |      +-- testDir
   |      +-- workers
   |      +-- fullyParallel
   |      +-- retries
   |      +-- forbidOnly
   |      +-- maxFailures
   |      +-- globalTimeout
   |      +-- projects
   |
   +-- GROUP
   |      |
   |      +-- test.describe()
   |      +-- describe.configure()
   |      +-- mode
   |      +-- group retries
   |      +-- group timeout
   |
   +-- ENVIRONMENT
   |      |
   |      +-- test.use()
   |      +-- browserName
   |      +-- viewport
   |      +-- storageState
   |      +-- baseURL
   |      +-- locale
   |
   +-- TIMEOUTS
   |      |
   |      +-- test timeout
   |      +-- action timeout
   |      +-- navigation timeout
   |      +-- assertion timeout
   |      +-- global timeout
   |
   +-- TEST CONTROL
   |      |
   |      +-- skip
   |      +-- fixme
   |      +-- fail
   |      +-- slow
   |      +-- only
   |
   +-- ORGANIZATION
          |
          +-- test.step()
          +-- projects
          +-- dependencies
          +-- hooks
```

------------------------------------------------------------------------

# 34. Final Revision Checklist

Use this checklist to check whether you can recall the topic without
reading the notes.

## Configuration

-   [ ] I can explain the purpose of `playwright.config.ts`.
-   [ ] I know which options belong at the top level and which belong
    inside `use`.
-   [ ] I can configure workers, retries and parallel execution.
-   [ ] I understand `forbidOnly`, `maxFailures` and `globalTimeout`.

## Environment

-   [ ] I can use `test.use()` to configure browser/context options.
-   [ ] I understand project configuration.
-   [ ] I can configure multiple browsers.
-   [ ] I understand project dependencies.

## Execution

-   [ ] I can explain default, parallel and serial modes.
-   [ ] I understand the difference between workers and fullyParallel.
-   [ ] I know why independent tests are preferred.

## Test control

-   [ ] I understand skip, fixme and fail.
-   [ ] I understand slow and only.
-   [ ] I can use `test.step()`.

## Timeouts

-   [ ] I can explain test timeout.
-   [ ] I can explain action timeout.
-   [ ] I can explain navigation timeout.
-   [ ] I can explain assertion timeout.
-   [ ] I can explain global timeout.
-   [ ] I understand how retries work.

## Interview readiness

-   [ ] I can answer the 25 practical scenarios.
-   [ ] I can write a configuration file without copying.
-   [ ] I can explain what happens when a test fails, retries and
    passes.
-   [ ] I can explain how parallel execution can affect shared test
    data.

------------------------------------------------------------------------

# 35. Official References

Use the official Playwright documentation when checking exact behaviour
or API details.

1.  [Playwright Test
    Configuration](https://playwright.dev/docs/test-configuration)
2.  [Test Use Options](https://playwright.dev/docs/test-use-options)
3.  [Timeouts](https://playwright.dev/docs/test-timeouts)
4.  [Annotations](https://playwright.dev/docs/test-annotations)
5.  [Parallelism](https://playwright.dev/docs/test-parallel)
6.  [Projects](https://playwright.dev/docs/test-projects)
7.  [Test API](https://playwright.dev/docs/api/class-test)
8.  [TestInfo API](https://playwright.dev/docs/api/class-testinfo)

**Final memory hook:**

> Know where an option is configured, what it controls, what happens
> when a test fails, and how you would use it in a real automation
> project.
