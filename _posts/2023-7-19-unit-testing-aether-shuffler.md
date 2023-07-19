---
layout: post
title: Unit Testing AetherShuffler
date: 2023-07-19 00:00:00 -0700
description: Unit Testing AetherShuffler
tags: [AetherShuffler, React, Firebase, Jest, React Testing Library]
Author: William Jeffreys
---

Welcome to AetherShuffler's unit testing journey! As we strive to enhance the reliability and stability of our app, it's time to dive into the world of unit testing. If you're new to the concept, that's okay — so am I! In this blog post, we'll explore how to use Jest and React Testing Library to write efficient and effective unit tests that help us catch bugs and gain a better understanding of our code.

I've narrowed down my tests to two files. This decision stems from the fact that when I developed AetherShuffler, I utilized Next.js' App Dir, which lacks sufficient support for testing. At the time of writing this post, even Next.js' website doesn't provide documentation for it. Fortunately, these two files don't rely on any Next.js specific code, and they offer enough code to test both pure JavaScript functions and user interactions with a React component.

First, let's examine `CardForm.js` and think about its fundamental functionality.

```js
import { useReducer, useRef } from "react";
import Select from "./UI/Inputs/Select";
import GetCards from "../utils/GetCards";

const initialState = {
  error: "",
};

function reducer(state, action) {
  switch (action.type) {
    case "SET_ERROR":
      return { ...state, error: action.error };

    default:
      throw new Error(`Unhandled action type: ${action.type}`);
  }
}

export default function CardForm({ onFormSubmit, submitRef }) {
  const [state, dispatch] = useReducer(reducer, initialState);
  const colorIdRef = useRef(null);
  const cardTypeRef = useRef(null);
  const cardFunctionRef = useRef(null);

  const colorIdOptions = [
    { name: "", value: "" },
    { name: "White", value: "White" },
    ...
  ];

  const cardTypeOptions = [
    { name: "", value: "" },
    { name: "Artifact", value: "Artifact" },
    ...
  ];

  const cardFunctionOptions = [
    { name: "", value: "" },
    { name: "Removal", value: "removal" },
    ...
  ];

  const handleSubmit = (e) => {
    e.preventDefault();

    const colorId = colorIdRef.current.value;
    const cardType = cardTypeRef.current.value;
    const cardFunction = cardFunctionRef.current.value;

    const formData = {
      color_id: colorId,
      card_type: cardType,
      card_function: cardFunction,
    };

    const sendForm = async () => {
      const data = await GetCards(formData, dispatch);
      onFormSubmit(data);
    };

    sendForm();
  };

  return (
    <div className="space-y-10 divide-y divide-gray-900/10">
      <div className="grid grid-cols-1 gap-x-8 gap-y-2 pt-10 md:grid-cols-3 lg:mx-24">
        <div className="px-4 sm:px-0">
          <h2 className="text-base font-semibold leading-7 text-gray-900">
            Card Search
          </h2>
          <p className="text-sm leading-6 text-gray-600 my-6">
            Enter the parameters for the cards you wish to find. You can search
            by any combination of color, type, and function.
            <br />
            <br />
            Tap/Click on a card and press the plus button to add it to your
            favorites. Favorites can be viewed and managed by clicking on your
            profile picture and selecting "Favorites".
          </p>
          {state.error && (
            <p className="text-red-600">{state.error}</p>
          )}
        </div>

        <form
          onSubmit={handleSubmit}
          className="bg-white shadow-sm ring-1 ring-gray-900/5 sm:rounded-xl md:col-span-2"
        >
          <div className="px-4 py-6 sm:p-8">
            <div className="grid grid-cols-1 gap-x-6 gap-y-8 sm:grid-cols-6">
              <div className="sm:col-span-3">
                <label
                  htmlFor="color_id"
                  className="block text-sm font-medium leading-6 text-gray-900"
                >
                  Color:
                </label>
                <div className="mt-2">
                  <Select
                    options={colorIdOptions}
                    ref={colorIdRef}
                    name="color_id"
                    id="color_id"
                    autoComplete="color_id"
                    placeholder="red"
                  />
                </div>
              </div>

              <div className="sm:col-span-3">
                <label
                  htmlFor="card_type"
                  className="block text-sm font-medium leading-6 text-gray-900"
                >
                  Card Type:
                </label>
                <div className="mt-2">
                  <Select
                    options={cardTypeOptions}
                    ref={cardTypeRef}
                    name="card_type"
                    id="card_type"
                    autoComplete="card_type"
                    placeholder="instant"
                  />
                </div>
              </div>

              <div className="sm:col-span-3">
                <label
                  htmlFor="card_function"
                  className="block text-sm font-medium leading-6 text-gray-900"
                >
                  Card Function:
                </label>
                <div className="mt-2">
                  <Select
                    options={cardFunctionOptions}
                    ref={cardFunctionRef}
                    name="card_function"
                    id="card_function"
                    autoComplete="card_function"
                    placeholder="function"
                  />
                </div>
              </div>
            </div>
          </div>
          <div className="flex items-center justify-center gap-x-6 border-t border-gray-900/10 px-4 py-4 sm:px-8">
            <button
              role="submit"
              ref={submitRef}
              type="submit"
              className="rounded-md bg-indigo-600 px-3 py-2 text-sm font-semibold text-white shadow-sm hover:bg-indigo-500 focus-visible:outline focus-visible:outline-2 focus-visible:outline-offset-2 focus-visible:outline-indigo-600"
            >
              Shuffle
            </button>
          </div>
        </form>
      </div>
    </div>
  );
}
```

This React component is a simple form, styled by Tailwind CSS, with a reducer to handle the state, and some refs to grab the input. When submitted, it calls `GetCards.js`, which is responsible for making the request to the Scryfall API that grabs the card data. It then passes that data via props to it's parent component.

So how do we test it? What kinds of tests should we run on it? Let's start building the test file and answer these questions.

We'll begin by creating a file called `CardForm.test.js` right next to where our component lives in our app, and importing some of things we'll need for testing at the top.

```js
import CardForm from "./CardForm";
import { render, waitFor } from "@testing-library/react";
import * as GetCardsModule from "../utils/GetCards";
import MockCardData from "../__mocks__/MockCardData.json";
import userEvent from "@testing-library/user-event";

describe("CardForm", () => {});
```

That describe block is where all our test's will live. It's referred to as a "suite" of tests by Jest, our main testing Library, and it takes a string that defines its name, and a callback function where the tests will go. The things imported will become relevant soon.

For our first test, we'll keep it incredibly simple. Let's just make sure the component renders without crashing. For that we'll need that render function that React Testing Library gives us.

```js
describe("CardForm", () => {
  it("Should render without crashing", () => {
    const form = render(<CardForm />);
    expect(form).toBeTruthy();
  });
});
```

In this test, we use the `it()` function to create a test within the test suite for the `CardForm` component. Inside the test, we render the `<CardForm />` component and assign it to the `form` variable. Then, we use `expect()` and apply the `.toBeTruthy()` matcher function to validate that the `form` is truthy. This ensures that the `<CardForm />` component renders properly, as it would otherwise be falsy, or throw an error if it didn't.

Now that our test is written, let's open a terminal and run `npm test`. If all goes according to plan, we should see this output:

```
 PASS  src/app/components/CardForm.test.js
  CardForm
    ✓ Should render without crashing (28 ms)

Test Suites: 1 passed, 1 total
Tests:       1 passed, 1 total
Snapshots:   0 total
Time:        0.825 s, estimated 1 s
Ran all test suites.
```

We see that our component renders without crashing, and our tests ran without errors. Nice! Let's see what else we can test.

We know that there should be 4 different inputs rendered in our component: 3 `<Select>` components and a submit button. So let's write a test that looks for them and passes if it finds them.

```js
it("Should render a form with 4 inputs", () => {
  const form = render(<CardForm />);

  let inputs = [];

  const selects = form.getAllByRole("Select");
  const buttons = form.getAllByRole("submit");

  inputs = inputs.concat(selects, buttons);

  expect(inputs.length).toBe(4);
});
```

In this test, we ensure that the rendered `<CardForm />` component contains a form with exactly four inputs. First, we render the `<CardForm />` component and initialize an empty array called `inputs`. Then, we use the `form.getAllByRole()` query to retrieve elements with the roles "Select" and "submit" from the form. The resulting arrays of `selects` and `buttons` are concatenated and assigned to `inputs`. Finally, we assert that the length of `inputs` is exactly 4 using `expect().toBe()`. This test demonstrates how to use Jest's querying capabilities effectively. For more details on available querying options, please refer to the Jest documentation.


Let's run `npm test`, and see what happens.

```
  CardForm
    ✓ Should render without crashing (30 ms)
    ✓ Should render a form with 4 inputs (77 ms)

Test Suites: 1 passed, 1 total
Tests:       2 passed, 2 total
Snapshots:   0 total
Time:        0.85 s, estimated 1 s
Ran all test suites.
```

Awesome, our tests ran and passed without issues! It would seem that our component renders and contains 4 inputs. What else does our component do? The users should be able to select those inputs, right? Let's test that functionality. We'll need Testing Library's user-event companion library for this. 

user-event simulates user interactions 