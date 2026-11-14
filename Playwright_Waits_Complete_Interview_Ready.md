# Playwright Waits & Synchronization --- Complete Practical & Interview-Ready Notes

> **Core idea:** Do not ask, "Where should I add a wait?" Ask, **"What
> condition am I waiting for?"**

------------------------------------------------------------------------

## 1. What Problem Do Waits Solve?

Web applications do not finish every operation immediately.

``` text
Click Search
    ↓
Request sent to server
    ↓
Server processes request
    ↓
Response comes back
    ↓
Application updates the page
    ↓
Results appear
```

If automation checks the result before the application has updated it,
the test may fail.

Playwright helps by: - Automatically waiting before supported Locator
actions. - Retrying web-first assertions. - Providing specific waits for
URL, Locator state, requests, responses and events.

> **Memory hook:** Wait for the condition you need, not an arbitrary
> number of seconds.

------------------------------------------------------------------------

## 2. The Big Picture

  -----------------------------------------------------------------------
  What do I need?                     Use
  ----------------------------------- -----------------------------------
  Perform a UI action                 Locator action + auto-waiting

  Verify a UI result                  `expect()` web-first assertion

  Wait for a Locator state            `locator.waitFor()`

  Wait for a URL                      `page.waitForURL()`

  Wait for an outgoing request        `page.waitForRequest()`

  Wait for a response                 `page.waitForResponse()`

  Wait for a page load milestone      `page.waitForLoadState()`

  Wait for a popup/new page           `page.waitForEvent('popup')` or
                                      `context.waitForEvent('page')`

  Pause for a fixed time              `page.waitForTimeout()` --- usually
                                      avoid
  -----------------------------------------------------------------------

------------------------------------------------------------------------

## 3. Auto-Waiting --- MUST KNOW

For supported Locator actions, Playwright checks whether the element is
ready for the requested action. It waits for the relevant checks to pass
and then performs the action.

``` ts
await page.getByRole('button', { name: 'Submit' }).click();
```

Mental model:

``` text
click()
   ↓
Is the target ready for this click?
   ↓
Not yet → wait and check again
   ↓
Ready
   ↓
Perform click
```

If the required checks do not pass within the action timeout, Playwright
throws a timeout error.

### Important

Auto-waiting does **not** mean Playwright waits until every API call or
every part of the application is finished. It waits for conditions
relevant to the requested action.

> **Memory hook:** Auto-waiting = Playwright waits until it can perform
> the action.

------------------------------------------------------------------------

## 4. Actionability --- What Does It Mean?

**Actionability** means checking whether an element is in a suitable
state for a particular action.

For `locator.click()`, Playwright checks that:

1.  The Locator resolves to exactly one element.
2.  The element is visible.
3.  The element is stable.
4.  The element receives pointer events.
5.  The element is enabled.
6.  The element is not inert.

### Simple meanings

  -----------------------------------------------------------------------
  Check                               Simple meaning
  ----------------------------------- -----------------------------------
  Visible                             The element has a visible,
                                      non-empty rendered area and is not
                                      `visibility:hidden`

  Stable                              Its bounding box is not changing;
                                      it is not moving/animating

  Receives Events                     The pointer can reach it; another
                                      element is not covering the click
                                      target

  Enabled                             It is not disabled

  Editable                            It can be edited; it is enabled and
                                      not readonly

  Not Inert                           It is not in an inert state that
                                      prevents user interaction
  -----------------------------------------------------------------------

### Important accuracy point

Different actions have different checks. Do not assume every action
checks all six conditions.

------------------------------------------------------------------------

## 5. Different Actions Have Different Checks --- MUST KNOW

Some useful examples from the official actionability table:

  -----------------------------------------------------------------------
  Locator action                      Checks Playwright performs
  ----------------------------------- -----------------------------------
  `click()`                           Visible, stable, receives events,
                                      enabled, not inert

  `check()` / `uncheck()`             Visible, stable, receives events,
                                      enabled, not inert

  `hover()`                           Visible, stable, receives events

  `fill()`                            Visible, enabled, editable, not
                                      inert

  `clear()`                           Visible, enabled, editable, not
                                      inert

  `selectOption()`                    Visible, enabled, not inert

  `scrollIntoViewIfNeeded()`          Stable
  -----------------------------------------------------------------------

### Remember

-   `click()` needs the target to be clickable.
-   `fill()` needs the target to be editable.
-   `hover()` needs the target to be reachable by the pointer.

> **Memory hook:** The action determines the checks.

Official reference:
[Actionability](https://playwright.dev/docs/actionability).

------------------------------------------------------------------------

## 6. Locator Uniqueness --- MUST KNOW

For actions such as `click()`, the Locator must resolve to exactly one
element.

``` ts
const submitButton = page.getByRole('button', { name: 'Submit' });

await submitButton.click();
```

If two buttons match, Playwright does not simply choose one. The action
fails with a strictness/uniqueness error.

``` text
Locator
   ↓
Exactly one matching element?
   ├── No → Fix the Locator
   └── Yes
         ↓
      Actionability checks
         ↓
      Action
```

Waiting longer does not fix an ambiguous Locator.

> **Memory hook:** A wait problem and a Locator problem are different
> problems.

------------------------------------------------------------------------

## 7. Important Visibility Detail

Playwright's visibility definition is specific.

-   `display: none` → not visible.
-   `visibility: hidden` → not visible.
-   Zero-size element → not visible.
-   `opacity: 0` → Playwright still considers it visible.

So, "I cannot see it with my eyes" is not always enough to predict
Playwright's visibility result.

------------------------------------------------------------------------

## 8. Stable --- Simple Explanation

An element is stable when its bounding box stays the same for at least
two consecutive animation frames.

``` text
Button is moving
     ↓
Playwright waits
     ↓
Button stops moving
     ↓
Click can continue
```

You do not need to calculate animation frames yourself.

------------------------------------------------------------------------

## 9. Receives Events --- Simple Explanation

Imagine a button with a loading overlay covering it.

``` text
Button is visible
      ↓
Overlay covers the button
      ↓
Mouse click reaches overlay, not button
```

The button is visible, but it cannot receive the pointer event.

This check helps catch overlays, modal layers, loading screens and
elements covering the target.

------------------------------------------------------------------------

## 10. `click()` Example

``` ts
await page.getByRole('button', { name: 'Submit' }).click();
```

Conceptually:

``` text
Find exactly one element
       ↓
Visible?
       ↓
Stable?
       ↓
Receives events?
       ↓
Enabled and not inert?
       ↓
CLICK
```

If a required check is not satisfied, Playwright waits until it passes
or the timeout is reached.

------------------------------------------------------------------------

## 11. `fill()` Example

``` ts
await page.getByLabel('Username').fill('admin');
```

The relevant checks include:

``` text
Find the input
    ↓
Visible?
    ↓
Enabled?
    ↓
Editable?
    ↓
Fill the value
```

A readonly input is not editable, so `fill()` cannot fill it.

------------------------------------------------------------------------

## 12. Auto-Waiting vs Web-First Assertions --- MUST KNOW

These solve different problems.

### Action

``` ts
await page.getByRole('button', { name: 'Login' }).click();
```

Question:

> **Can Playwright perform this action?**

### Assertion

``` ts
await expect(page.getByText('Welcome Admin')).toBeVisible();
```

Question:

> **Has the expected result happened?**

Web-first assertions retry the condition until it passes or the
assertion timeout is reached.

``` text
expect(...).toBeVisible()
        ↓
Is it visible?
   ├── No → wait and check again
   └── Yes → PASS
```

### Important

Do not replace an assertion with a one-time check such as `isVisible()`
when you need Playwright to retry until the expected result appears.

``` ts
// One-time check: does not auto-retry as an assertion
const visible = await message.isVisible();

// Web-first assertion: retries until it passes or times out
await expect(message).toBeVisible();
```

> **Memory hook:** Action auto-waiting = Can I do it? Assertion retry =
> Did the expected result happen?

------------------------------------------------------------------------

## 13. Real Login Example

``` ts
await page.getByLabel('Username').fill('admin');

await page.getByLabel('Password').fill('secret');

await page.getByRole('button', { name: 'Login' }).click();

await expect(page.getByText('Welcome Admin')).toBeVisible();
```

No fixed sleep is needed between every step.

``` text
fill()  → relevant actionability checks
fill()  → relevant actionability checks
click() → relevant actionability checks
expect() → retries until the welcome message is visible
```

------------------------------------------------------------------------

## 14. `page.waitForTimeout()` --- Fixed Wait

``` ts
await page.waitForTimeout(3000);
```

Meaning:

> Pause execution for 3 seconds.

### Why it is usually a poor synchronization choice

**Application ready in 500 ms:**

``` text
Application ready
      ↓
Test still waits
      ↓
3 seconds finish
      ↓
Test continues
```

**Application needs 5 seconds:**

``` text
Application still processing
      ↓
3 seconds finish
      ↓
Test continues too early
      ↓
Possible failure
```

Fixed waits can be too long or too short. They do not check whether the
required condition is ready.

### When is it useful?

Occasionally for debugging or a temporary local investigation. Avoid
using it as the normal synchronization method in production tests.

> **Memory hook:** `waitForTimeout()` = Wait exactly this long, whether
> the app is ready or not.

------------------------------------------------------------------------

## 15. `locator.waitFor()` --- MUST KNOW

Use it when you specifically need a Locator to reach a state.

``` ts
await page.locator('#spinner').waitFor({
  state: 'hidden'
});
```

Supported states:

  State        Meaning
  ------------ ---------------------------------
  `attached`   Element is present in the DOM
  `detached`   Element is removed from the DOM
  `visible`    Element is visible
  `hidden`     Element is hidden or detached

If no state is provided, the default is `visible`.

### `locator.waitFor()` vs `expect()`

``` ts
// Wait for a Locator state
await spinner.waitFor({ state: 'hidden' });

// Assert an expected UI result
await expect(resultsTable).toBeVisible();
```

Use `expect()` when you want to verify a test requirement. Use
`locator.waitFor()` when waiting for a specific Locator state is the
clearest expression.

------------------------------------------------------------------------

## 16. `page.waitForURL()` --- MUST KNOW

Use it when a particular URL change is the condition you care about.

``` ts
await page.getByRole('button', { name: 'Login' }).click();

await page.waitForURL('**/dashboard');
```

It can also be registered before an action:

``` ts
const urlPromise = page.waitForURL('**/dashboard');

await page.getByRole('button', { name: 'Login' }).click();

await urlPromise;
```

### Memory hook

> `waitForURL()` = Wait until the page reaches the expected URL.

------------------------------------------------------------------------

## 17. `page.waitForLoadState()` --- SHOULD KNOW

Waits for a page load milestone.

``` ts
await page.waitForLoadState('domcontentloaded');
```

Available states:

  -----------------------------------------------------------------------
  State                               Simple meaning
  ----------------------------------- -----------------------------------
  `commit`                            Navigation response has been
                                      received and the document starts
                                      loading; this is used with
                                      navigation waiting APIs, not as a
                                      `waitForLoadState` value

  `domcontentloaded`                  HTML has been parsed and the
                                      DOMContentLoaded event fired

  `load`                              The page's load event fired

  `networkidle`                       No network connections for at least
                                      500 ms; discouraged for test
                                      readiness
  -----------------------------------------------------------------------

For `page.waitForLoadState()`, the supported states are `load`,
`domcontentloaded`, and `networkidle`. The default is `load`.

### Important

A page load milestone does not automatically mean the business UI is
ready.

``` text
HTML parsed
    ↓
Page load event
    ↓
JavaScript runs
    ↓
API request
    ↓
Data arrives
    ↓
Table renders
```

If the test needs the table, assert the table:

``` ts
await expect(page.getByRole('table')).toBeVisible();
```

Most of the time, `waitForLoadState()` is not needed because Playwright
already auto-waits before supported actions.

------------------------------------------------------------------------

## 18. `networkidle` --- Important Limitation

You may see:

``` ts
await page.waitForLoadState('networkidle');
```

It waits until there are no network connections for at least 500 ms.

Modern applications may continuously make background requests for: -
Analytics - Polling - Notifications - Background APIs

Therefore:

> **Network idle does not automatically mean the business UI is ready.**

Playwright discourages using `networkidle` as a general testing
readiness signal. Prefer a web assertion for the UI condition you need.

------------------------------------------------------------------------

## 19. `page.waitForRequest()` --- MUST KNOW

Use it when you need to wait for a specific outgoing request.

``` ts
const requestPromise = page.waitForRequest(
  request =>
    request.url().includes('/api/customers') &&
    request.method() === 'GET'
);

await page.getByRole('button', { name: 'Search' }).click();

const request = await requestPromise;

console.log(request.url());
```

### Memory hook

> `waitForRequest()` = Wait for the browser to send this request.

------------------------------------------------------------------------

## 20. `page.waitForResponse()` --- MUST KNOW

Use it when you need to wait for a specific response.

``` ts
const responsePromise = page.waitForResponse(
  response =>
    response.url().includes('/api/customers') &&
    response.request().method() === 'GET' &&
    response.status() === 200
);

await page.getByRole('button', { name: 'Search' }).click();

const response = await responsePromise;
```

### Important

A successful API response does not automatically prove that the UI has
finished rendering.

If the test needs both API and UI verification, do both:

``` ts
const responsePromise = page.waitForResponse(
  response =>
    response.url().includes('/api/customers') &&
    response.status() === 200
);

await page.getByRole('button', { name: 'Search' }).click();

const response = await responsePromise;

await expect(page.getByRole('table')).toBeVisible();
```

### Request vs Response

``` text
Browser
   |
   | ---- REQUEST ----> Server
   |
   | <---- RESPONSE ---- Server
```

-   Request = outgoing call.
-   Response = server's reply.

------------------------------------------------------------------------

## 21. Why Register Event/Network Waits Before the Action?

The request or event can happen immediately after the action.

Correct pattern:

``` ts
const responsePromise = page.waitForResponse(
  response => response.url().includes('/api/customers')
);

await page.getByRole('button', { name: 'Search' }).click();

const response = await responsePromise;
```

Mental model:

``` text
Register the wait
       ↓
Perform the action
       ↓
Request/event happens
       ↓
Wait resolves
```

Do not wait until after the click to register the listener.

------------------------------------------------------------------------

## 22. `Promise.all()` Pattern --- SHOULD KNOW

You may also see this pattern:

``` ts
const [response] = await Promise.all([
  page.waitForResponse(response =>
    response.url().includes('/api/customers')
  ),
  page.getByRole('button', { name: 'Search' }).click()
]);
```

The wait and action start together. This is useful when coordinating an
action with an event or response.

You do not need `Promise.all()` for every wait. The important idea is to
register the wait before triggering the action.

------------------------------------------------------------------------

## 23. `page.waitForSelector()` --- KNOW IT, PREFER LOCATORS

You may encounter:

``` ts
await page.waitForSelector('#username');
```

The API still exists, but Playwright discourages using it as the normal
approach. Prefer Locator actions, Locator `waitFor()`, or web-first
assertions.

Instead of:

``` ts
await page.waitForSelector('#username');
await page.locator('#username').fill('admin');
```

Normally use:

``` ts
await page.locator('#username').fill('admin');
```

The Locator action already performs relevant auto-waiting.

> **Memory hook:** Do not add a separate wait when the action already
> provides the synchronization you need.

------------------------------------------------------------------------

## 24. Locators Are Central to Auto-Waiting

A Locator describes how to find an element.

``` ts
const submitButton = page.getByRole('button', { name: 'Submit' });
```

A Locator is not simply a permanently stored DOM element. When used,
Playwright resolves the matching element and performs the relevant
checks.

This is useful when the DOM changes or the page re-renders.

------------------------------------------------------------------------

## 25. `force: true` --- KNOW IT, DON'T ABUSE IT

Some actions support:

``` ts
await button.click({ force: true });
```

`force: true` disables non-essential actionability checks. For a click,
this can bypass the check that the target receives pointer events.

It does not mean every possible check is skipped, and it does not
guarantee that the action will succeed.

Before using it, investigate: - Is an overlay covering the element? - Is
the Locator correct? - Is the control disabled? - Is an animation still
running? - Is there an application defect?

> **Memory hook:** `force: true` = Bypass some non-essential checks. Do
> not use it to hide a real problem.

------------------------------------------------------------------------

## 26. Real Project Example: Loading Spinner

Application flow:

``` text
Click Search
    ↓
Spinner appears
    ↓
API call
    ↓
Spinner disappears
    ↓
Customer table appears
```

### If the requirement is spinner disappearance

``` ts
await page.locator('.spinner').waitFor({
  state: 'hidden'
});
```

### If the requirement is that the table appears

``` ts
await expect(page.getByRole('table')).toBeVisible();
```

### If the API response itself matters

``` ts
const responsePromise = page.waitForResponse(
  response =>
    response.url().includes('/api/customers') &&
    response.status() === 200
);

await page.getByRole('button', { name: 'Search' }).click();

const response = await responsePromise;
```

The correct wait depends on what the test must prove.

------------------------------------------------------------------------

## 27. Real Project Example: Healthcare Claim Submission

``` text
Create Claim
    ↓
Click Submit
    ↓
POST /claims
    ↓
Server processes claim
    ↓
Claim ID / success message appears
```

### Poor approach

``` ts
await page.getByRole('button', { name: 'Submit' }).click();

await page.waitForTimeout(5000);

await expect(page.getByText('Claim created')).toBeVisible();
```

The five seconds are arbitrary.

### Better UI-based synchronization

``` ts
await page.getByRole('button', { name: 'Submit' }).click();

await expect(page.getByText('Claim created')).toBeVisible();
```

### If the API response also matters

``` ts
const responsePromise = page.waitForResponse(
  response =>
    response.url().includes('/claims') &&
    response.request().method() === 'POST' &&
    response.status() === 201
);

await page.getByRole('button', { name: 'Submit' }).click();

const response = await responsePromise;

await expect(page.getByText('Claim created')).toBeVisible();
```

This verifies both the API response and the user-visible result.

------------------------------------------------------------------------

## 28. Common Mistakes

### Mistake 1 --- Add `waitForTimeout()` everywhere

Instead ask: What condition am I actually waiting for?

### Mistake 2 --- Wait for page load when you need a UI condition

`load` does not necessarily mean the table or business data is ready.
Assert the actual UI requirement.

### Mistake 3 --- Add `waitFor()` before every action

If `click()` already provides the relevant auto-waiting, an extra wait
may add no value.

### Mistake 4 --- Assume every action has the same checks

The checks depend on the action.

### Mistake 5 --- Assume `networkidle` means application ready

Network quiet does not prove that the required UI has rendered.

### Mistake 6 --- Use `force: true` to hide a problem

Investigate why the normal action cannot be performed.

### Mistake 7 --- Use a one-time check when you need a retrying assertion

Prefer `await expect(locator).toBeVisible()` when the UI may become
visible after a short delay.

### Mistake 8 --- Register a response wait after clicking

Register the wait first, then trigger the action.

------------------------------------------------------------------------

## 29. Synchronization Decision Tree

``` text
I need synchronization
       |
       v
What am I waiting for?
       |
       +-- UI action can be performed?
       |       → Locator action + auto-waiting
       |
       +-- Expected UI result?
       |       → expect()
       |
       +-- Specific Locator state?
       |       → locator.waitFor()
       |
       +-- URL?
       |       → page.waitForURL()
       |
       +-- Outgoing request?
       |       → page.waitForRequest()
       |
       +-- Response?
       |       → page.waitForResponse()
       |
       +-- Page load milestone?
       |       → page.waitForLoadState()
       |
       +-- Fixed delay?
               → page.waitForTimeout()
                  Usually avoid
```

------------------------------------------------------------------------

## 30. Important Interview Differences

### A. Auto-waiting vs Assertion retry

  -----------------------------------------------------------------------
  Auto-waiting                        Web-first assertion
  ----------------------------------- -----------------------------------
  Used by supported Locator actions   Used by `expect()` web assertions

  Checks whether action can be        Checks whether expected condition
  performed                           becomes true

  Example: `click()`                  Example: `toBeVisible()`
  -----------------------------------------------------------------------

### B. `locator.waitFor()` vs `expect()`

  `locator.waitFor()`         `expect()`
  --------------------------- ----------------------------------
  Waits for a Locator state   Verifies an expected result
  Example: spinner hidden     Example: success message visible

### C. `waitForRequest()` vs `waitForResponse()`

  -----------------------------------------------------------------------
  Request                             Response
  ----------------------------------- -----------------------------------
  Waits for outgoing request          Waits for incoming response

  Useful to confirm request was sent  Useful to inspect server
                                      response/status
  -----------------------------------------------------------------------

### D. `waitForLoadState()` vs UI assertion

  -----------------------------------------------------------------------
  Load state                          UI assertion
  ----------------------------------- -----------------------------------
  Waits for a browser document load   Waits for the application condition
  milestone                           

  Does not prove business UI is ready Verifies the required UI state
  -----------------------------------------------------------------------

### E. Fixed wait vs condition wait

  -----------------------------------------------------------------------
  Fixed wait                          Condition wait
  ----------------------------------- -----------------------------------
  Waits for a chosen duration         Waits until a condition is met

  May be too long or too short        Continues when the required
                                      condition is ready
  -----------------------------------------------------------------------

------------------------------------------------------------------------

## 31. Top Practical Interview Scenarios

You should be able to explain and implement these without looking at
notes.

1.  A button appears after an API call. How do you click it without
    `waitForTimeout()`?
2.  What checks happen before a Locator `click()`?
3.  Does `fill()` use the same checks as `click()`?
4.  What happens if a Locator matches two buttons?
5.  A button is visible but an overlay covers it. Why can the click
    fail?
6.  What is the difference between action auto-waiting and assertion
    retry?
7.  Why is `isVisible()` not a replacement for
    `expect(locator).toBeVisible()` when waiting for a future state?
8.  How do you wait for a spinner to disappear?
9.  How do you wait for a URL to change after login?
10. How do you wait for a specific API request?
11. How do you wait for a response with status 201?
12. Why must the response wait be registered before clicking Submit?
13. Does a 200 API response prove that the table is rendered?
14. What is the difference between `locator.waitFor()` and `expect()`?
15. What are the supported states of `locator.waitFor()`?
16. What is the difference between `load` and `domcontentloaded`?
17. Why is `networkidle` discouraged as a general readiness check?
18. When would you use `waitForLoadState()`?
19. Why is `waitForSelector()` not the preferred modern approach?
20. What does `force: true` bypass, and why should it be used carefully?
21. How would you test a claim submission that triggers an API request
    and then displays a success message?
22. How do you investigate a click timeout rather than blindly
    increasing the timeout?

------------------------------------------------------------------------

## 32. One-Line Memory Tricks

  Concept                Remember it as
  ---------------------- ---------------------------------------------------
  Auto-waiting           "Can I perform this action?"
  Actionability          "Is the element ready for THIS action?"
  `expect()`             "Has the expected result happened?"
  `locator.waitFor()`    "Wait for this Locator state."
  `waitForURL()`         "Wait for this URL."
  `waitForRequest()`     "Wait for this outgoing request."
  `waitForResponse()`    "Wait for this incoming response."
  `waitForLoadState()`   "Wait for this document load milestone."
  `waitForTimeout()`     "Wait exactly this long."
  `networkidle`          "Network is quiet --- not necessarily UI ready."
  `force: true`          "Bypass some non-essential actionability checks."

------------------------------------------------------------------------

## 33. Final Mental Map

``` text
                 PLAYWRIGHT SYNCHRONIZATION
                           |
             +-------------+-------------+
             |                           |
        AUTOMATIC                     CONDITION-SPECIFIC
             |                           |
      +------+-------+          +--------+--------+
      |              |          |        |        |
 Locator actions   expect()    Locator   URL    Network
      |              |          |        |        |
 Actionability   Retry until  waitFor  waitFor  Request /
 checks          assertion    state    URL      Response
                 passes
```

------------------------------------------------------------------------

## 34. Final Revision Checklist

### Fundamentals

-   [ ] I can explain why waits are needed.
-   [ ] I understand auto-waiting.
-   [ ] I can explain actionability in simple English.
-   [ ] I know that different actions have different checks.
-   [ ] I understand Locator uniqueness.

### Assertions and Locator waits

-   [ ] I can explain the difference between action auto-waiting and
    assertion retry.
-   [ ] I can use `expect()` for a UI result.
-   [ ] I can use `locator.waitFor()` for a specific state.
-   [ ] I know the four `locator.waitFor()` states.

### Explicit waits

-   [ ] I can use `waitForURL()`.
-   [ ] I can use `waitForRequest()`.
-   [ ] I can use `waitForResponse()`.
-   [ ] I understand page load states.
-   [ ] I know why `networkidle` is not a universal readiness signal.
-   [ ] I know why `waitForTimeout()` is usually avoided.
-   [ ] I know why `waitForSelector()` is not the preferred approach.

### Troubleshooting

-   [ ] I can investigate a click timeout.
-   [ ] I understand what `force: true` does.
-   [ ] I can distinguish a Locator problem from a wait problem.
-   [ ] I can verify both an API response and the resulting UI when
    required.

### Interview readiness

-   [ ] I can answer the 22 practical scenarios.
-   [ ] I can write the examples without copying.
-   [ ] I can explain which synchronization method I would choose and
    why.

------------------------------------------------------------------------

## 35. Official References

Use official Playwright documentation for exact behaviour:

1.  [Actionability and
    auto-waiting](https://playwright.dev/docs/actionability)
2.  [Locators](https://playwright.dev/docs/locators)
3.  [Assertions](https://playwright.dev/docs/test-assertions)
4.  [Page API](https://playwright.dev/docs/api/class-page)
5.  [Locator API](https://playwright.dev/docs/api/class-locator)
6.  [Best practices](https://playwright.dev/docs/best-practices)

**Final takeaway:**

> Master the decision, not the API list. First identify the condition
> your test needs, then choose the Playwright mechanism that waits for
> that condition.
