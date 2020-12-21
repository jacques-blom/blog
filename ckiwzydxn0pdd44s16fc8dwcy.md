## Don't use getByTestId

Building interfaces that are accessible to everyone has always been a bit of a black box to me. I do know, however, that not enough apps on the web are built in an accessible way.

Thankfully web standards include a lot of ways that you can make apps accessible. It can be complicated, though. And you can't always tell whether or not you've built something accessible.

One method that has changed how I build my interfaces is using `getByRole` from React Testing Library instead of `getByTestId`.

Note: `getByRole` actually comes from DOM Testing Library, meaning it's available in many of the [Testing Libraries](https://testing-library.com/). This article will use React Testing Library as an example though.

There are also a [few more](https://testing-library.com/docs/guide-which-query) accessible queries exposed by DOM Testing Library, but we'll focus on `getByRole`.

> If you use `getByRole` as much as possible, over queries like `getByTestId`, you will write more accessible code.

# Our non-accessible component

In our example, we have a todo list item that you can toggle checked by clicking on the checkbox. Try it out for yourself:

<iframe src="https://codesandbox.io/embed/accessible-react-tests-drlp5?fontsize=14&hidenavigation=1&theme=dark&view=preview"
     style="width:100%; height:300px; border:0; border-radius: 4px; overflow:hidden;"
     title="delicate-sky-omxj6"
     allow="accelerometer; ambient-light-sensor; camera; encrypted-media; geolocation; gyroscope; hid; microphone; midi; payment; usb; vr; xr-spatial-tracking"
     sandbox="allow-forms allow-modals allow-popups allow-presentation allow-same-origin allow-scripts"
   ></iframe>

Our Task component is built like this:

<iframe src="https://codesandbox.io/embed/accessible-react-tests-drlp5?fontsize=14&hidenavigation=1&module=%2Fsrc%2Fnot-accessible%2FTask.tsx&theme=dark&view=editor"
     style="width:100%; height:500px; border:0; border-radius: 4px; overflow:hidden;"
     title="delicate-sky-omxj6"
     allow="accelerometer; ambient-light-sensor; camera; encrypted-media; geolocation; gyroscope; hid; microphone; midi; payment; usb; vr; xr-spatial-tracking"
     sandbox="allow-forms allow-modals allow-popups allow-presentation allow-same-origin allow-scripts"
   ></iframe>

If you try to focus on the checkbox with your keyboard to mark the task as completed you'll see that you can't. And it won't work with a screen reader either because we don't have any accessible labels in our UI.

Instead of trying to figure out how to make it accessible by studying the WAI-ARIA spec, let's try and do it using tests!

You can clone the repo to follow along, or just read further.

```bash
# Git clone
git clone git@github.com:jacques-blom/accessible-react-tests.git
git checkout tutorial-start

# Install dependencies
yarn

# To start the app
yarn start
```

Then, run the tests in watch mode:

```bash
yarn test --watch
```

# Our current test

Let's first look at our current test:

```jsx
// src/Task.test.tsx

it("toggles the task checked state", () => {
    render(<Task />)

    // Get the checkbox element
    const checkbox = screen.getByTestId("checkbox")
    const checkIcon = screen.getByTestId("checkIcon")

    // Click it
    userEvent.click(checkbox)

    // Expect the checkbox to be checked
    expect(checkIcon).toHaveStyle("opacity: 1")

    // Click it again
    userEvent.click(checkbox)

    // Expect the checkbox to be unchecked
    expect(checkIcon).toHaveStyle("opacity: 0")
})
```

Our test doesn't test whether the app is accessible - it just tries to find an element (a `div` in our case) that has a specific `data-testid` prop.

> Tests using getByTestId don't test accessibility. But we can change our tests to help us build accessible UI.

# Step 1: Change our test

We're going to make our app more accessible by taking a TDD approach: first rewriting our test to use `getByRole`, then changing our code to make the test pass!

Let's rather test our app the way an assistive technology would query our UI. An assistive technology can't just look at our dark circle and determine that it's a checkbox - we actually need to tell it that it's a checkbox.

Instead of querying for the checkbox by testId, we're going to query it by an accessible role:

```jsx
const checkbox = screen.getByRole("checkbox")
```

This will try to find an element on the page that has identified itself as a [checkbox](https://developer.mozilla.org/en-US/docs/Web/Accessibility/ARIA/Roles/checkbox_role).

You can find the role that best describes the interactive element you want to test by going through the full list of roles [here](https://developer.mozilla.org/en-US/docs/Web/Accessibility/ARIA/ARIA_Techniques#Roles).

Let's modify our test:

```diff
// src/Task.test.tsx

 it("toggles the task checked state", () => {
   render(<Task />);

-  const checkbox = screen.getByTestId("checkbox");
+  const checkbox = screen.getByRole("checkbox");
   const checkIcon = screen.getByTestId("checkIcon");

   // Checked
   userEvent.click(checkbox);
   expect(checkIcon).toHaveStyle("opacity: 1");

   // Not checked
   userEvent.click(checkbox);
   expect(checkIcon).toHaveStyle("opacity: 0");
 });
```

You'll now see that our test fails. That's because our current element is just a `div`. DOM Testing Library even gives us a list of possible accessible elements on the page to help us along:

![Test failing](https://cdn.hashnode.com/res/hashnode/image/upload/v1608449793083/OnIxeOcXi.png)

> Using getByRole forces you to have accessible elements in your UI.

# Step 2: Change our code

Let's start by adding a checkbox input element to our `Checkbox` component.

```diff
const Checkbox = ({ checked, onChange }: CheckboxProps) => {
  return (
    <div
      data-testid="checkbox"
      className="checkbox"
      onClick={() => onChange(!checked)}
    >
      <img
        alt="check icon"
        src="/check.svg"
        style={{ opacity: checked ? 1 : 0 }}
        data-testid="checkIcon"
      />
+     <input type="checkbox" />
    </div>
  );
};
```

> Use native HTML elements as much as possible to make life easy. You will have to deal with overriding browser styling, but the upside is that you get accessibility out of the box.

Next, instead of relying on the `div`'s `onClick` event, we'll use the checkbox's `onChange` event:

```diff
const Checkbox = ({ checked, onChange }: CheckboxProps) => {
  return (
    <div
      data-testid="checkbox"
      className="checkbox"
-     onClick={() => onChange(!checked)}
    >
      <img
        alt="check icon"
        src="/check.svg"
        style={{ opacity: checked ? 1 : 0 }}
        data-testid="checkIcon"
      />
-    <input type="checkbox" />
+    <input type="checkbox" onChange={(event) => onChange(event.target.checked)} />
    </div>
  );
};
```

Our test is now passing again!

![Test failing 2.png](https://cdn.hashnode.com/res/hashnode/image/upload/v1608451038147/MdBY35Qa6.png)

But we now have an ugly checkbox breaking our design. 😢

![Ugly checkbox.gif](https://cdn.hashnode.com/res/hashnode/image/upload/v1608459959851/XxNKZS_GI.gif)

So let's add some CSS to fix this.

```scss
// src/Task.scss

.checkbox {
  ...
  position: relative;

  > input[type="checkbox"] {
    // Make the input float above the other elements in .checkbox
    position: absolute;
    top: 0;
    left: 0;

    // Make the input cover .checkbox
    width: 100%;
    height: 100%;
  }
  ...
}
```

Now the checkbox (almost) covers our styled checkbox.

![Checkbox 1.png](https://cdn.hashnode.com/res/hashnode/image/upload/v1608451055592/x4wAUSFj6.png)

We also need to remove the default margin that comes with the checkbox, and add `overflow: hidden` to `.checkbox` so that the checkbox isn't clickable outside our circular design:

```scss
// src/Task.scss

.checkbox {
  ...
  // Prevent the input overflowing outside the border-radius
  overflow: hidden;

  > input[type="checkbox"] {
    ...

    // Remove default margin
    margin: 0;
  }
  ...
}
```

![Checkbox 2.png](https://cdn.hashnode.com/res/hashnode/image/upload/v1608451064565/s0O_zCMIH.png)

Finally, now that our checkbox input is fully covering our custom checkbox, we can hide it:

```scss
// src/Task.scss

.checkbox {
  ...
  > input[type="checkbox"] {
    ...

    // Hide the input
    opacity: 0;
  }
  ...
}
```

Now we're back to our old design and behavior, and our checkbox is (almost) accessible. Try tabbing to it and hitting the spacebar to toggle the checked state:

![Back to old behavior.gif](https://cdn.hashnode.com/res/hashnode/image/upload/v1608455952852/jHw6LfgAy.gif)

I say it's almost accessible because someone using keyboard navigation instead of a mouse can't see if the checkbox is focused. So let's add a focus state:

```scss
// src/Task.scss

.checkbox {
  ...
  // Show an outline when the input is focused
  &:focus-within {
    box-shadow: 0 0 0 1px #fff;
  }
  ...
}
```

We're using `:focus-within` on `.checkbox` to apply a style to it if anything inside it is focused:

![Focus state.gif](https://cdn.hashnode.com/res/hashnode/image/upload/v1608459740894/Rjo48jeJd.gif)

> Always make sure your UI can be navigated using a keyboard. Accessible HTML elements help but make sure you include focus states if you override the default browser outline style.

Finally, we want to label our checkbox with something meaningful so that screen readers can tell the user what the checkbox is for.

We can either add a `<label>` element, or we can use the `aria-label` prop. Since we don't want our label to be visible, we'll go for the latter:

```jsx
// src/Task.tsx

<input
    type="checkbox"
    onChange={(event) => onChange(event.target.checked)}
    // Add an aria-label
    aria-label={checked ? "mark unchecked" : "mark checked"}
/>
```

To make the label as helpful as possible, we're showing a different label depending on whether the task is checked.

We can now modify our test to find a checkbox with that label, to make sure our label is set. To do this we pass a `name` parameter to our `getByRole` call:

```jsx
const checkbox = screen.getByRole("checkbox", { name: "mark as checked" })
```

> Make sure your UI has helpful labels that can be used by screen readers. Query your elements by those labels in your tests to make sure they're there.

But we need to find it by a different label depending on whether the checkbox is checked or not. We can refactor things a bit to make this easy.

Our final test looks like this:

<iframe src="https://codesandbox.io/embed/accessible-react-tests-drlp5?fontsize=14&hidenavigation=1&module=%2Fsrc%2Faccessible%2FTask.test.tsx&theme=dark&view=editor"
     style="width:100%; height:500px; border:0; border-radius: 4px; overflow:hidden;"
     title="Accessible React Tests"
     allow="accelerometer; ambient-light-sensor; camera; encrypted-media; geolocation; gyroscope; hid; microphone; midi; payment; usb; vr; xr-spatial-tracking"
     sandbox="allow-forms allow-modals allow-popups allow-presentation allow-same-origin allow-scripts"
   ></iframe>

And here is our final, accessible UI:

<iframe src="https://codesandbox.io/embed/accessible-react-tests-drlp5?fontsize=14&hidenavigation=1&initialpath=%2F%3Faccessible%3D1&theme=dark&view=preview"
     style="width:100%; height:300px; border:0; border-radius: 4px; overflow:hidden;"
     title="Accessible React Tests"
     allow="accelerometer; ambient-light-sensor; camera; encrypted-media; geolocation; gyroscope; hid; microphone; midi; payment; usb; vr; xr-spatial-tracking"
     sandbox="allow-forms allow-modals allow-popups allow-presentation allow-same-origin allow-scripts"
   ></iframe>

What did we improve here in our test?

1. Added a `getCheckbox` function to fetch our checkbox by the checked or unchecked label to clean things up.
1. Expect the checkbox to be checked, instead of checking whether our styled check is visible or not. This makes our code more resilient to change...

# How getByRole makes your tests resilient to changing code

Because we are now testing our code in a way that it will be used (find a checkbox input), rather than the way it's built (find an element with a specific test ID), our tests are more resilient to refactoring.

If we completely changed how our UI was built, even if we removed all our UI altogether and just kept the checkbox input, our tests will still pass.

I recently refactored a form from React Hook Form to Formik, and all my tests still worked, even though the underlying code was totally different. Plus, because of how I wrote my tests, my form was completely accessible!

%[https://twitter.com/jacques_codes/status/1339572271128653824]

---

# What we've learned

1. Using `getByRole` in your tests will test whether your UI is accessible.
1. `getByRole` makes your code resilient to refactoring.
1. When refactoring your UI to make it accessible, use a TTD approach. Write failing tests, then get your tests to pass.
1. UI is more accessible when it can be easily navigated using a keyboard and has meaningful accessible labels.
1. Use native browser elements to get out-of-the-box accessibility.

# Further reading

If you're interested in testing and accessibility, I am planning on releasing a bunch more content about it. Enter your email address below to be notified when I release new content.

%%[convertkit]
