# Playwright Window & Tab Handling --- Complete Practical & Interview-Ready Notes

> **Core idea:** In Playwright, a browser tab or popup window is
> represented by a `Page` object. Capture the correct `Page` and work
> with it directly.

------------------------------------------------------------------------

## 1. The One Mental Model to Remember

``` text
Browser
   ↓
BrowserContext = isolated session
   ↓
Pages = tabs / popup windows
```

The most important idea:

> **A tab or popup window is a `Page` object in Playwright.**

You normally do not use Selenium-style window handles and
`switchTo().window()`.

``` text
New tab / popup opens
        ↓
New Page is created
        ↓
Capture the Page
        ↓
Use that Page directly
```

------------------------------------------------------------------------

## 2. Browser → BrowserContext → Page

A Browser can have multiple BrowserContexts. Each BrowserContext can
have multiple Pages.

``` text
Browser
   |
   +-- Context 1 = Session 1
   |      |
   |      +-- Page 1 = Tab 1
   |      +-- Page 2 = Tab 2
   |
   +-- Context 2 = Session 2
          |
          +-- Page 1 = Tab 1
```

### Simple memory

-   **Browser** = actual browser process/instance.
-   **BrowserContext** = isolated browser session/environment.
-   **Page** = tab or popup window.

A Page always belongs to a BrowserContext.

### Important

Pages in the same context share context-level settings such as cookies,
permissions, locale and viewport configuration. Different contexts are
isolated from each other.

> **Memory hook:** Browser = browser, Context = session, Page = tab.

------------------------------------------------------------------------

## 3. Creating Pages

### Create a new Page in an existing context

``` ts
const context = await browser.newContext();

const pageOne = await context.newPage();
const pageTwo = await context.newPage();
```

Both Pages belong to the same context.

### Get the context from an existing Page

``` ts
const context = page.context();
```

### Get all currently open Pages

``` ts
const pages = context.pages();
```

`context.pages()` returns an array of Pages that are currently open in
that context. It is a snapshot of the Pages at the time you call it; it
is not a listener for future Pages.

------------------------------------------------------------------------

## 4. Scenario: A Page Opens a New Tab

Suppose:

``` text
Current Page
     ↓
Click "Open Customer"
     ↓
New tab opens
```

Use `page.waitForEvent('popup')` when the current Page opens the popup.

``` ts
const popupPromise = page.waitForEvent('popup');

await page.getByRole('link', {
  name: 'Open Customer'
}).click();

const newPage = await popupPromise;
```

Now:

``` text
page
  ↓
Original tab

newPage
  ↓
New popup/tab
```

You can interact with the new Page directly:

``` ts
await expect(
  newPage.getByRole('heading', { name: 'Customer Details' })
).toBeVisible();
```

### Golden rule

> **REGISTER THE WAIT → PERFORM THE ACTION → CAPTURE THE PAGE → USE THE
> PAGE**

Do not register the event wait after the action. The popup may open
before the listener is ready.

------------------------------------------------------------------------

## 5. `page.waitForEvent('popup')` --- MUST KNOW

Use this when a **specific Page opens another Page**.

``` text
Page A
   |
   +---- opens ----> Page B
          popup event
```

Example:

``` ts
const popupPromise = page.waitForEvent('popup');

await page.getByRole('button', {
  name: 'Open Report'
}).click();

const popup = await popupPromise;

console.log(popup.url());
```

The popup is a `Page` object. It belongs to the same BrowserContext as
its opener.

### Memory trick

> **PAGE → POPUP**

### Important

The `popup` event is emitted on the opener Page. It is for popups
relevant to that specific Page.

------------------------------------------------------------------------

## 6. `context.waitForEvent('page')` --- MUST KNOW

Use this when you want to detect a new Page created anywhere inside a
particular BrowserContext.

``` text
BrowserContext
    |
    +-- Page 1
    +-- Page 2
    +-- NEW Page  ← page event
```

Example:

``` ts
const context = page.context();

const newPagePromise = context.waitForEvent('page');

await page.getByRole('link', {
  name: 'Open Customer'
}).click();

const newPage = await newPagePromise;

console.log(newPage.url());
```

This is useful when: - A new tab is opened by a `target="_blank"`
link. - A popup is opened by another Page in the same context. - You
need to detect a new Page without tying the listener to one particular
opener.

### Memory trick

> **CONTEXT → NEW PAGE**

------------------------------------------------------------------------

## 7. `popup` vs `page` Event --- VERY IMPORTANT

  -----------------------------------------------------------------------
  API                                 What it detects
  ----------------------------------- -----------------------------------
  `page.waitForEvent('popup')`        A popup opened by this specific
                                      Page

  `context.waitForEvent('page')`      A new Page created anywhere in this
                                      BrowserContext
  -----------------------------------------------------------------------

### Important relationship

A popup event is also a BrowserContext `page` event.

So, when a Page opens a popup: - The opener Page emits `popup`. - The
BrowserContext emits `page`.

The difference is the scope of the listener, not two different types of
browser windows.

### Visual memory

``` text
page.waitForEvent('popup')

Page 1
   |
   +----> Page 2
```

``` text
context.waitForEvent('page')

BrowserContext
   |
   +-- Page 1
   +-- Page 2
   +-- NEW Page  ← detected
```

### Which one should I use?

-   If you know which Page opens the popup, use
    `page.waitForEvent('popup')`.
-   If you need to detect any new Page in the context, use
    `context.waitForEvent('page')`.

Do not blindly use `context.waitForEvent('page')` for every situation.
Choose based on the scope you need.

------------------------------------------------------------------------

## 8. `waitForEvent()` vs `on()` --- MUST KNOW

### `waitForEvent()`

Use it when you expect a particular event as part of a specific action.

``` ts
const popupPromise = page.waitForEvent('popup');

await page.getByText('Open').click();

const popup = await popupPromise;
```

Think:

> **I am waiting for an event that I expect now.**

### `on()`

Use it when you want to keep listening for events.

``` ts
page.on('popup', popup => {
  console.log('A popup opened:', popup.url());
});
```

Think:

> **Keep listening whenever this event happens.**

### Memory

``` text
waitForEvent()
    ↓
Wait for one expected event

on()
    ↓
Register a listener for events
```

------------------------------------------------------------------------

## 9. `Promise.all()` Pattern --- SHOULD KNOW

A common pattern is:

``` ts
const [newPage] = await Promise.all([
  page.waitForEvent('popup'),
  page.getByRole('link', {
    name: 'Open Customer'
  }).click()
]);
```

This starts the event wait and click together, so the listener is ready
before the popup event happens.

The important concept is not `Promise.all()` itself.

> **The important concept is: register the event wait before triggering
> the action.**

The separate promise pattern from section 4 is also correct and often
easier for beginners to read.

------------------------------------------------------------------------

## 10. Getting All Currently Open Pages

Use:

``` ts
const pages = context.pages();
```

Example:

``` ts
const context = page.context();

const openedPages = context.pages();

console.log(openedPages.length);

for (const currentPage of openedPages) {
  console.log(currentPage.url());
}
```

Mental model:

``` text
BrowserContext
   |
   +-- Page 1
   +-- Page 2
   +-- Page 3

context.pages()
      ↓
[Page 1, Page 2, Page 3]
```

### Important distinction

``` text
context.waitForEvent('page')
      ↓
Wait for a new Page event

context.pages()
      ↓
Get Pages currently open now
```

------------------------------------------------------------------------

## 11. Do Not Blindly Use `pages()[1]`

You may see:

``` ts
const pages = context.pages();
const newPage = pages[1];
```

This can work in a very simple test, but it is not a reliable way to
identify the Page created by a particular action.

For example:

``` text
Page 1 = original page
Page 2 = another existing page
Page 3 = the Page you expected
```

`pages[1]` would return Page 2, not Page 3.

If an action is expected to create a new Page, wait for the event and
capture that Page.

> **Memory hook:** `context.pages()` = see existing Pages.
> `waitForEvent()` = capture a Page created by an expected event.

------------------------------------------------------------------------

## 12. Do You Need `bringToFront()`?

Usually, no.

You can interact directly with the Page object:

``` ts
await newPage.getByRole('button', {
  name: 'Download'
}).click();
```

You do not normally need:

``` ts
await newPage.bringToFront();
```

`bringToFront()` is only useful when you specifically want that Page
brought to the front of the browser UI.

Playwright can interact with a Page even when it is not the frontmost
tab.

> **Memory hook:** Capture the Page → use the Page.

------------------------------------------------------------------------

## 13. Do You Need `waitForLoadState()` Every Time?

No.

After capturing a popup, you can use a Locator action or assertion:

``` ts
const popup = await popupPromise;

await expect(
  popup.getByRole('heading', { name: 'Report' })
).toBeVisible();
```

If you specifically need a document load milestone, use:

``` ts
await popup.waitForLoadState('domcontentloaded');
```

Do not add `waitForLoadState()` after every new Page automatically.

A load event does not necessarily mean the business UI or data is ready.
Prefer an assertion for the actual condition your test needs.

------------------------------------------------------------------------

## 14. What If the Action Can Open a Page, but You Do Not Know Which Page?

Use a BrowserContext listener:

``` ts
context.on('page', async newPage => {
  console.log('A new Page was created');

  await newPage.waitForLoadState('domcontentloaded');

  console.log(await newPage.title());
});
```

This keeps listening for new Pages in that context.

If you only need to capture one Page after a known action,
`context.waitForEvent('page')` is usually simpler.

For popups from one known opener, use `page.on('popup', ...)`.

------------------------------------------------------------------------

## 15. Multiple New Pages --- Practical Consideration

`waitForEvent()` resolves for the next matching event that satisfies its
predicate. If one action can open multiple Pages, a single wait captures
only one event.

For example, to wait for a specific URL:

``` ts
const pagePromise = context.waitForEvent('page', {
  predicate: newPage => newPage.url().includes('/customer')
});

await page.getByRole('button', { name: 'Open Customer' }).click();

const customerPage = await pagePromise;
```

The predicate helps select the Page you expect.

If the new Page has not navigated to its final URL yet, do not rely on
its URL being final at the instant of the event. You may need to wait
for the expected URL or UI state after capturing it.

------------------------------------------------------------------------

## 16. Closing a Page

Close a particular Page with:

``` ts
await newPage.close();
```

This closes that Page, not every Page in the context.

Closing the BrowserContext closes all Pages belonging to that context:

``` ts
await context.close();
```

### Memory

``` text
page.close()
    ↓
Close one Page

context.close()
    ↓
Close the context and its Pages
```

------------------------------------------------------------------------

## 17. No Selenium-Style Window Switching

### Selenium mental model

``` text
Window Handle
      ↓
switchTo().window()
      ↓
Work with selected window
```

### Playwright mental model

``` text
New Tab / Popup
      ↓
New Page object
      ↓
Capture Page
      ↓
Work with Page
```

There is normally no need for:

``` ts
driver.switchTo().window(...)
```

equivalent for regular Playwright tab handling.

------------------------------------------------------------------------

## 18. Real Project Example --- Salesforce Account

Imagine a Salesforce application:

``` text
Accounts list page
       ↓
Click "View Account"
       ↓
New Account Details tab
```

Test:

``` ts
const accountPagePromise = page.waitForEvent('popup');

await page.getByRole('link', {
  name: 'View Account'
}).click();

const accountPage = await accountPagePromise;

await expect(
  accountPage.getByRole('heading', {
    name: 'Account Details'
  })
).toBeVisible();
```

Here:

``` text
page
  ↓
Accounts list page

accountPage
  ↓
Account details tab
```

No window switching is required.

------------------------------------------------------------------------

## 19. API Cheat Sheet

  --------------------------------------------------------------------------------
  API                              Simple meaning          Priority
  -------------------------------- ----------------------- -----------------------
  `page.waitForEvent('popup')`     Wait for a popup from   MUST KNOW
                                   this Page               

  `context.waitForEvent('page')`   Wait for a new Page in  MUST KNOW
                                   this context            

  `context.pages()`                Get all currently open  MUST KNOW
                                   Pages                   

  `page.on('popup', ...)`          Keep listening for      SHOULD KNOW
                                   popups from this Page   

  `context.on('page', ...)`        Keep listening for new  SHOULD KNOW
                                   Pages in this context   

  `page.bringToFront()`            Bring Page to front of  SHOULD KNOW
                                   browser UI              

  `page.waitForLoadState()`        Wait for a document     SHOULD KNOW
                                   load milestone          

  `page.close()`                   Close this Page         MUST KNOW

  `context.close()`                Close the context and   SHOULD KNOW
                                   its Pages               

  `page.context()`                 Get the BrowserContext  MUST KNOW
                                   that owns this Page     
  --------------------------------------------------------------------------------

------------------------------------------------------------------------

## 20. Common Mistakes

### Mistake 1 --- Thinking in Selenium

Wrong mental model:

``` text
Window Handle → Switch
```

Correct:

``` text
New tab → New Page → Capture Page → Use Page
```

### Mistake 2 --- Waiting after clicking

Wrong:

``` ts
await page.getByText('Open').click();

const popup = await page.waitForEvent('popup');
```

Correct:

``` ts
const popupPromise = page.waitForEvent('popup');

await page.getByText('Open').click();

const popup = await popupPromise;
```

### Mistake 3 --- Using the old Page

After a new Page opens, use the captured Page for elements inside that
new tab.

``` ts
const popup = await popupPromise;

await popup.getByRole('heading').waitFor();
```

Do not keep using the original `page` if the element is in the popup.

### Mistake 4 --- Assuming `pages()[1]` is always the new Page

Avoid hard-coded indexes when an event can identify the new Page
reliably.

### Mistake 5 --- Thinking `bringToFront()` is mandatory

It is not required just to interact with a Page.

### Mistake 6 --- Adding `waitForLoadState()` everywhere

Wait for a load milestone only when you specifically need it. Prefer the
actual UI assertion when that is the requirement.

### Mistake 7 --- Using context and popup events as if they were unrelated

A popup also causes a BrowserContext `page` event. Choose the event
based on the scope you need.

------------------------------------------------------------------------

## 21. Top Practical Interview Scenarios

You should be able to explain and implement these without looking at
notes.

1.  A link opens a new tab. How do you capture it?
2.  A button opens a popup window. How do you handle it?
3.  What is the difference between `page.waitForEvent('popup')` and
    `context.waitForEvent('page')`?
4.  Does a popup also trigger the BrowserContext `page` event?
5.  When would you use `context.waitForEvent('page')` instead of
    `page.waitForEvent('popup')`?
6.  Why must the event wait be registered before clicking?
7.  What is the difference between `waitForEvent()` and `on()`?
8.  How do you get all currently open tabs?
9.  Does `context.pages()` wait for a future Page?
10. Why is `pages()[1]` not a reliable way to identify a new tab?
11. Do you need `bringToFront()` before interacting with a popup?
12. Do you need `waitForLoadState()` every time a popup opens?
13. How do you handle a popup when the opener Page is known?
14. How do you listen for any new Page in a BrowserContext?
15. What happens if an action opens multiple Pages?
16. How can you wait for a Page matching a particular URL?
17. How do you close one tab?
18. What is the difference between `page.close()` and `context.close()`?
19. Can multiple Pages exist inside one BrowserContext?
20. How would you validate a Salesforce Account Details page opened in a
    new tab?

------------------------------------------------------------------------

## 22. Final Mental Map

``` text
                          BROWSER
                             |
                       BROWSER CONTEXT
                         = SESSION
                             |
                +------------+------------+
                |            |            |
              PAGE 1       PAGE 2       PAGE 3
              Tab 1        Tab 2        Tab 3
```

### A specific Page opens another Page

``` text
Page opens popup
       ↓
page.waitForEvent('popup')
```

### Any new Page in the context

``` text
New Page created in BrowserContext
       ↓
context.waitForEvent('page')
```

### Get all Pages that exist now

``` text
context.pages()
```

### Keep listening

``` text
page.on('popup', ...)
context.on('page', ...)
```

### No switching

``` text
❌ switchTo().window()

✅ Capture Page → Use Page
```

------------------------------------------------------------------------

## 23. Final Revision Checklist

### Browser hierarchy

-   [ ] I can explain Browser → BrowserContext → Page.
-   [ ] I know a Page represents a tab or popup.
-   [ ] I know one context can contain multiple Pages.
-   [ ] I know Pages in different contexts are isolated.

### New Page handling

-   [ ] I can handle a popup using `page.waitForEvent('popup')`.
-   [ ] I can handle a new Page using `context.waitForEvent('page')`.
-   [ ] I understand the difference between popup and page events.
-   [ ] I register the wait before the action.
-   [ ] I can use `Promise.all()` when appropriate.

### Existing Pages and listeners

-   [ ] I can get all open Pages using `context.pages()`.
-   [ ] I know `context.pages()` is not a future-event listener.
-   [ ] I know when to use `on()` versus `waitForEvent()`.
-   [ ] I avoid relying on `pages()[1]`.

### Working with Pages

-   [ ] I can interact with the captured Page directly.
-   [ ] I know `bringToFront()` is not required for interaction.
-   [ ] I know when a load-state wait is useful.
-   [ ] I can close a Page and a BrowserContext.

### Interview readiness

-   [ ] I can answer the 20 practical scenarios.
-   [ ] I can write the popup handling code without copying.
-   [ ] I can explain why Playwright does not need Selenium-style window
    switching.

------------------------------------------------------------------------

## 24. Official References

Use official Playwright documentation for exact behaviour:

1.  [Pages and popup handling](https://playwright.dev/docs/pages)
2.  [Page API](https://playwright.dev/docs/api/class-page)
3.  [BrowserContext
    API](https://playwright.dev/docs/api/class-browsercontext)
4.  [Events](https://playwright.dev/docs/events)

**Final memory hook:**

> Browser = browser, BrowserContext = isolated session, Page =
> tab/window. Wait for the right event, capture the Page, and use that
> Page directly.
