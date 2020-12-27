## Mock Service Worker Tutorial Part 2

This is Part 2 of my Mock Service Worker Tutorial series. In [Part 1](https://jacquesblom.com/mock-service-worker-tutorial) we learned how to install MSW and write some basic tests.

In this article we're going to dive deeper into MSW, looking at:

-   Testing POST requests.
-   Testing requests that have route parameters.
-   Some more testing best practices.
-   Re-using handlers across tests.
-   Selectively mocking error states.

To follow along, clone [the repo](https://github.com/jacques-blom/taskhero-web/tree/part-2) and switch to the part-2 branch:

```bash
git clone git@github.com:jacques-blom/taskhero-web.git
cd taskhero-web
git checkout part-2
yarn
```

Run the tests in watch mode:

```bash
yarn test src/App.test.tsx --watch
```
---

# How to test POST requests with MSW

## What we're testing

In our next test, we'll test whether the flow of inserting a task works:

![Inserting a task](https://cdn.hashnode.com/res/hashnode/image/upload/v1608542960790/nodAkUOtH.gif)

## 1. Add the handler

Our Taskhero app inserts tasks by POSTing to `/tasks`. Let's add a new handler to `src/mocks/handlers.ts` to handle a POST to that endpoint:

```js
// src/mocks/handlers.ts

import {v4} from 'uuid'

// Use rest.post instead of rest.get
rest.post(getApiUrl('/tasks'), (req, res, ctx) => {
    // Make sure we receive a request body as a string
    if (typeof req.body !== 'string') throw new Error('Missing request body')

    // Parse the request body
    const newTask = JSON.parse(req.body)

    // Emulate our real API's behaviour by throwing if we don't receive a label
    if (newTask.label.length === 0) {
        return res(ctx.status(400), ctx.json({message: 'Missing label'}))
    }

    // Emulate our real API's behaviour by responding with the new full task object
    return res(
        ctx.json({
            id: v4(),
            label: newTask.label,
            completed: false,
        }),
    )
}),
```

In our handler we're emulating how our real API would respond in different scenarios:

1. We're throwing if we don't receive a body.
1. We're throwing if the user doesn't provide a label.
1. We're responding with the new task object if the task was inserted successfully.

> 💡 We can make our handlers as realistic as possible by responding the same way our real API would in different scenarios.

## 2. Write the test

Now let's test whether a task is inserted successfully. Before we start, let's extract our logic that waits for loading to complete, to make things easier:

```jsx
// src/App.test.tsx

const waitForLoading = () => {
    return waitForElementToBeRemoved(() =>
        screen.getByRole("alert", { name: "loading" })
    )
}
```

> 💡 If you want to know why and how I am using getByRole, check out my post: [Don't use getByTestId](https://jacquesblom.com/dont-use-getbytestid).

Let's add our test:

```jsx
// src/App.test.tsx

it("inserts a new task", async () => {
    render(<App />, { wrapper: GlobalWrapper })
    await waitForLoading()

    const insertInput = screen.getByRole("textbox", { name: /insert/i })

    // Type a task and press enter
    userEvent.type(insertInput, "New task")
    fireEvent.keyUp(insertInput, { keyCode: 13 })

    // Test the loading state
    expect(insertInput).toBeDisabled()

    // Test the success state
    await waitFor(() => expect(insertInput).not.toBeDisabled())
    expect(insertInput).toHaveValue("")

    // Test whether the task is displaying on the page
    expect(screen.getByTestId(/task-/)).toHaveTextContent("New task")
})
```

In the above test we're testing the whole flow of inserting a task.

### Testing best practice: Write fewer, longer tests

This is a practice I've recently started using more. Instead of breaking up each assertion into its own test, combine all the assertions for a given flow into one test.

This means you don't have to set up the environment for each assertion, so:

1. You have less code in your tests.
1. They're quicker to write.
1. They're quicker to run.

I got this idea from Kent C. Dodds's article: [Write fewer, longer tests
](https://kentcdodds.com/blog/write-fewer-longer-tests).

My feeling on how to split up tests is to write a test for a given user flow or state. So for this flow we'll write one test for successfully inserting a task, and another for whether the error state is handled.

## 3. Testing the failure case

Now we can write a test for the failure case, which is when a user tries to insert a task without a label. This will also cover testing any other error from the API.

![Insert failure](https://cdn.hashnode.com/res/hashnode/image/upload/v1608543032973/UfFj4D0eQ.gif)

```jsx
// src/App.test.tsx

it("displays an error message if the API fails", async () => {
    render(<App />, { wrapper: GlobalWrapper })
    await waitForLoading()

    const insertInput = screen.getByRole("textbox", { name: /insert/i })

    // Just press enter without typing a label
    fireEvent.keyUp(insertInput, { keyCode: 13 })

    // Wait for loading to complete
    await waitFor(() => expect(insertInput).not.toBeDisabled())

    // Expect an error alert to display
    expect(screen.getByRole("alert").textContent).toMatchInlineSnapshot()
})
```

> 💡 When testing a UI interacting with an API, always test how the UI reacts to all possible API responses, including errors. Even if your API never throws, something else like a network error could occur.

### Testing best practice: Expecting certain text content, and using snapshots to help you

In our above example, to test that the error being displayed is actually the error from the API, we're expecting the error to display.

If we just tested for the presence of an alert we wouldn't know whether we were displaying the correct error.

To make life a bit easier, we use `toMatchInlineSnapshot`, which we start by calling without passing in a string (`.toMatchInlineSnapshot()`). Then, when we run the test for the first time Jest will automatically change it to `.toMatchInlineSnapshot('"Missing label"')`.

Then, if our message ever changes, Jest will ask us whether or not we want to update the snapshot. Try to change the error message in `src/mocks/handlers.ts` to see for yourself!

---

# How to test requests that have route parameters with MSW

## What we're testing

In our next test, we'll test whether the flow of checking a task, calling the API, and then finally marking it as checked in the UI works:

![Toggle checked](https://cdn.hashnode.com/res/hashnode/image/upload/v1608543443920/x24XulzQ_.gif)

When a task is marked complete, the app makes a POST request to the `/task/1` endpoint, where `1` is the ID of the task.

## 1. Add the handlers

```jsx
// src/mocks/handlers.ts

rest.post(getApiUrl('/task/:id'), (req, res, ctx) => {
    // Make sure we receive a request body as a string
    if (typeof req.body !== 'string') throw new Error('Missing request body')

    // Parse the request body
    const newTask = JSON.parse(req.body)

    // Get the task ID from the route parameter
    const taskId = req.params.id

    // Emulate our real API's behavior by responding with the updated task object
    return res(
        ctx.json({
            id: taskId,
            label: 'Example',
            completed: newTask.completed,
        }),
    )
}),
```

> 💡 You can use the `:paramname` syntax to specify that this endpoint will take in a parameter. Just like Express.js routes, this parameter will be accessible from the `req.params` object.

For this test we're also going to have to display a task on the page. To do this, let's create a handler in `src/mocks/handlers.ts`:

```jsx
// src/mocks/handlers.ts

export const singleTask = rest.get(getApiUrl("/tasks"), (req, res, ctx) => {
    return res(
        ctx.json([
            {
                id: v4(),
                label: "Example",
                completed: false,
            },
        ])
    )
})
```

You'll notice we're exporting it from the file, rather than passing it to the `handlers` array. That's because passing it to the `handlers` array would override our existing mock for `/tasks`. We could have just included this in the test itself, but I know we're going to reuse it. And adding it here makes it easy to reuse.

> 💡 You can make handlers easily reusable by exporting them from a handlers file and importing them in your individual tests.

## 2. Write the test

```jsx
// src/App.test.tsx

// Import our singleTask handler
import { singleTask } from "./mocks/handlers"

it("toggles the task completed state", async () => {
    // Mock a single task on the page
    server.use(singleTask)

    render(<App />, { wrapper: GlobalWrapper })
    await waitForLoading()

    // Click the checkbox
    userEvent.click(screen.getByRole("checkbox", { name: /mark as completed/ }))

    // Expect it to be disabled while loading
    expect(screen.getByRole("checkbox")).toBeDisabled()

    // Wait for the checkbox to be checked
    await waitFor(() => expect(screen.getByRole("checkbox")).toBeChecked())

    // Click the now-checked checkbox
    userEvent.click(
        screen.getByRole("checkbox", { name: /mark as uncompleted/ })
    )

    // Wait for the checkbox to be unchecked
    await waitFor(() => expect(screen.getByRole("checkbox")).not.toBeChecked())
})
```

## 3. Testing the failure case

![Toggle failure.gif](https://cdn.hashnode.com/res/hashnode/image/upload/v1608543584011/-8a0AaRYb.gif)

To test this failure case, instead of adding logic to conditionally throw in our `/task/:id` handler, let's override our handler in this test to always throw:

```jsx
// src/App.test.tsx

it("handles toggling the completed state failing", async () => {
    // Re-use our singleTask handler to display a single task on the page
    server.use(singleTask)

    // Return an error response from the API when we try to call this endpoint
    server.use(
        rest.post(getApiUrl("/task/:id"), (req, res, ctx) =>
            res(ctx.status(500), ctx.json({ message: "Something went wrong" }))
        )
    )

    render(<App />, { wrapper: GlobalWrapper })
    await waitForLoading()

    // Click the checkbox
    userEvent.click(screen.getByRole("checkbox", { name: /mark as completed/ }))

    // Expect the error to display once loading has completed
    await waitFor(() => {
        return expect(
            screen.getByRole("alert").textContent
        ).toMatchInlineSnapshot()
    })

    // Make sure the checkbox stays unchecked
    expect(screen.getByRole("checkbox")).not.toBeChecked()
})
```

> 💡 You can mock your API in a given test to always throw, which makes testing failure cases super simple and predictable.

---

# We're done! What did we learn?

In this article, we learned:

1. How to test POST requests and their effect on the app when they respond.
1. How to add route parameters to your handler paths.
1. How to export individual handlers for re-use in multiple tests.
1. Why it's better to write fewer, longer tests.
1. Why you should `expect` certain text content, and how snapshots make it easy.
1. How to test failure cases by writing handlers that always throw.

# Further reading

If you're interested in testing and using Mock Service Worker, I am planning on releasing a bunch more content about it. [Click here](https://jacquesblom.com/subscribe) to subscribe and be notified when I release new content.

Also, feel free to [Tweet at me](https://twitter.com/jacques_codes) if you have any questions.

If you found this post helpful, and you think others will, too, please consider spreading the love and sharing it.