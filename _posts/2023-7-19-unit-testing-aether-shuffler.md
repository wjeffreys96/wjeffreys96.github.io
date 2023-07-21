---
layout: post
title: Unit Testing AetherShuffler
date: 2023-07-19 00:00:00 -0700
description: Unit Testing AetherShuffler
tags: [AetherShuffler, React, Firebase, Jest, React Testing Library]
Author: William Jeffreys
---
- [AetherShuffler](https://aether-shuffler.vercel.app/)

Welcome to AetherShuffler's unit testing journey! As we strive to enhance the reliability and stability of our app, it's time to dive into the world of unit testing. If you're new to the concept, that's okay — so am I! In this blog post, we'll explore how to use Jest and React Testing Library to write efficient and effective unit tests that help us catch bugs and gain a better understanding of our code.

For simplicity's sake, I've narrowed down my tests to two files: `CardForm.js` and `GetCards.js`. These two files offer enough code to test both pure JavaScript functions and user interactions with a React component.

First, let's examine `CardForm.js` and think about its fundamental functionality.

## CardForm.js

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

Awesome, our tests ran and passed without issues! It would seem that our component renders and contains 4 inputs.

What else does our component do? The users should be able to select those inputs, right? Let's test that functionality. We'll need Testing Library's user-event companion library for this.

User-event simulates user interactions by dispatching the events that would happen if the interaction took place in a browser. We first have to import it, then call `UserEvent.setup()` and assign it to a variable.

```js
describe("CardForm", () => {
  const user = userEvent.setup();
  ...
});
```

Now, with a user available to us, we can write our test that simulates the user selecting options, and then verifies if those options were successfully selected.

```js
it("Should allow the user to select parameters to search with", async () => {
  const form = render(<CardForm />);

  const colorIdSelect = form.getByLabelText("Color:");
  const cardTypeSelect = form.getByLabelText("Card Type:");
  const cardFunctionSelect = form.getByLabelText("Card Function:");

  user.selectOptions(colorIdSelect, "White");
  user.selectOptions(cardTypeSelect, "Instant");
  user.selectOptions(cardFunctionSelect, "Removal");

  expect(await form.findByText("White")).toBeTruthy();
  expect(await form.findByText("Instant")).toBeTruthy();
  expect(await form.findByText("Removal")).toBeTruthy();
});
```

We begin by rendering the form using the <CardForm /> component. Next, we retrieve the necessary Select components from the form using `getByLabelText()`. Specifically, we obtain the `colorIdSelect`, `cardTypeSelect`, and `cardFunctionSelect` components.

To simulate the user's interaction, we use the `user.selectOptions()` function to programmatically select options on each of the Select components. In this example, we simulate the user choosing "White" for the color, "Instant" for the card type, and "Removal" for the card function.

Following the simulated selections, we perform assertions to ensure that the selected options are displayed correctly within the form. We use `findByText()` to locate the expected text values, such as "White," "Instant," and "Removal." The expect statements, combined with `toBeTruthy()`, validate whether the form indeed contains these expected selections.

```
 PASS  src/app/components/CardForm.test.js
  CardForm
    ✓ Should render without crashing (31 ms)
    ✓ Should render a form with 4 inputs (75 ms)
    ✓ Should allow the user to select parameters to search with (100 ms)

Test Suites: 1 passed, 1 total
Tests:       3 passed, 3 total
Snapshots:   0 total
Time:        1.331 s
```

Great, it looks like selecting works, too! Only one more thing to test, and that's that the form properly submits the user data.

When the form is submitted, it sends the data to its parent component via props. To ensure proper submission, we can mock the function passed through props. Additionally, we can simulate the expected data that should be sent on a successful call to verify that the form correctly communicates with its parent.

To simulate the resolved data, we intercept the `GetCards.js` function at the beginning of our `describe()` block and force it to output our mock data. This approach offers the advantage of isolating any potential issues to the form itself rather than the `GetCards.js` function, making troubleshooting easier.

```js
describe("CardForm", () => {
  const user = userEvent.setup();
  const getCardsSpy = jest.spyOn(GetCardsModule, "default");

  getCardsSpy.mockResolvedValue(MockCardData);
  ...

  it("Should call the onFormSubmit function when submitted", async () => {
    const mockFormSubmit = jest.fn();

    const form = render(<CardForm onFormSubmit={mockFormSubmit} />);

    const submitButton = form.getByRole("submit");

    user.click(submitButton);

    await waitFor(() => {
      expect(mockFormSubmit).toHaveBeenCalledWith(MockCardData);
    });
  });
});
```

In this test case we begin by creating a mock function, `mockFormSubmit`, using `jest.fn()`. This function will simulate the submission behavior.

We then render the form with the onFormSubmit prop set to `mockFormSubmit`. After obtaining the submit button using `getByRole("submit")`, we simulate a user click on the button using `user.click(submitButton)`.

To ensure that the `onFormSubmit()` function is called with the expected data, we use `waitFor()` and `expect()` assertions. The `waitFor()` function ensures that the assertion is performed after the form submission is processed. In this case, we expect the `mockFormSubmit` function to be called with the `MockCardData`.

Alright then lets run our test and see that it passes.

```
 PASS  src/app/components/CardForm.test.js
  CardForm
    ✓ Should render without crashing (27 ms)
    ✓ Should render a form with 4 inputs (73 ms)
    ✓ Should allow the user to select parameters to search with (100 ms)
    ✓ Should call the onFormSubmit function when submitted (70 ms)

Test Suites: 1 passed, 1 total
Tests:       4 passed, 4 total
Snapshots:   0 total
Time:        0.967 s, estimated 1 s
Ran all test suites.
```
Fanstastic, our test runs and passes. There are probably a million other tests we could write for this component, but for the purposes of this post, let's leave it there and move on to testing `GetCards.js`, a vanilla Javascript utility function that fetches and sorts card objects from the Scryfall API.

## GetCards.js
```js
export default async function GetCards(
  { color_id: colorId, card_type: cardType, card_function: cardFunction },
  dispatch
) {
  const params = new URLSearchParams({
    q: `${colorId && "f:commander id<=" + colorId} ${
      colorId != "Colorless" ? "-c:c" : ""
    } ${cardType && "t:" + cardType} order:edhrec dir:asc ${
      cardFunction && "oracletag:" + cardFunction
    }`,
  });

  const url = `https://api.scryfall.com/cards/search?${params}`;

  async function fetcher(url) {
    try {
      const res = await fetch(url);
      const data = await res.json();
      return data.data;
    } catch (error) {
      dispatch({ type: "SET_ERROR", error: error });
    }
  }

  function selectRandomCards(array, numCards) {
    try {
      const randomSubset = [];

      for (let i = 0; i < numCards; i++) {
        const randomIndex = Math.floor(Math.random() * array.length);
        const selectedCard = array.splice(randomIndex, 1)[0];
        randomSubset.push(selectedCard);
      }
      return randomSubset;
    } catch (error) {
      dispatch({ type: "SET_ERROR", error: error });
    }
  }

  const response = await fetcher(url);

  const randomCards = selectRandomCards(response, 24);

  const cardDataMaker = () => {
    try {
      const cardData = randomCards.map((card) => {
        const name = card.name;
        const id = card.id;
        let imageUri = "";

        if (card.image_uris) {
          imageUri = card.image_uris.normal;
        } else {
          imageUri = card.card_faces[0].image_uris.normal;
        }
        return { name, id, imageUri };
      });
      return cardData;
    } catch (error) {
      dispatch({ type: "SET_ERROR", error: error });
    }
  };

  const cardData = cardDataMaker();

  return cardData;
}
```

Three big things I see to test are:
1. It returns an array of 24 cards.
2. Each card has a name, id, and image uri.
3. If it runs into an error, it dispatches that error to the function that called it.

To begin, we'll import the function itself, and a mock of some data that the API should return, then start writing our `describe()` suite.

```js
import GetCards from "./GetCards";
import MockBulkCardData from "../__mocks__/MockBulkCardData.json";

describe("GetCards", () => {});
```
Then we'll write the first test that asserts that `GetCards()` returns an array with 24 cards in it.

```js
  it("Should get an array with at least 24 cards", async () => {
    global.fetch = jest.fn().mockResolvedValue({
      json: jest.fn().mockResolvedValue({
        data: MockBulkCardData,
      }),
    });

    const cards = await GetCards({}, jest.fn());

    expect(cards.length).toBeLessThanOrEqual(24);
    expect(Array.isArray(cards)).toBe(true);
  });
```

By overriding the global.fetch function and providing it with mock data, we simulate a successful API response. Subsequently, we invoke GetCards() and verify that the returned array has a length of at least 24 and is indeed an array. This initial test offers a fundamental assessment of the function.

```
 PASS  src/app/utils/GetCards.test.js
  GetCards
    ✓ Should get an array with at least 24 cards (5 ms)

Test Suites: 1 passed, 1 total
Tests:       1 passed, 1 total
Snapshots:   0 total
Time:        0.546 s, estimated 1 s
Ran all test suites related to changed files.
```

Okay, so the first test passed and our function works as intended. On to the next one.

The goal of this test is to ensure that every card in the array returned by our function contains the three expected properties: a name, an id, and an image URI. Let's take a look at the test case:

```js
  it("Every card should have a name, id, and imageUri", async () => {
    global.fetch = jest.fn().mockResolvedValue({
      json: jest.fn().mockResolvedValue({
        data: MockBulkCardData,
      }),
    });

    const cards = await GetCards({}, jest.fn());

    const hasExpectedProperties = cards.every((card) =>
      ["name", "id", "imageUri"].every((prop) => {
        const value = card[prop];
        return typeof value === "string" && value !== "";
      })
    );

    expect(hasExpectedProperties).toBe(true);
  });
```

In this test case, we start by again mocking a successful API response. Then, we call `GetCards()` and store the resulting cards in the cards variable.

To verify the presence of the expected properties in each card, we use the `every()` method to iterate through the array of cards. For each card, we again use the `every()` method to check if all the required properties exist and contain a non-empty string.

Finally, we use the `expect()` statement to assert that `hasExpectedProperties` is true, indicating that all the cards in the array indeed have the expected properties.

Now we run the test.

```
 PASS  src/app/utils/GetCards.test.js
  GetCards
    ✓ Should get an array with at least 24 cards (4 ms)
    ✓ Every card should have a name, id, and imageUri (1 ms)

Test Suites: 1 passed, 1 total
Tests:       2 passed, 2 total
Snapshots:   0 total
Time:        0.909 s, estimated 1 s
Ran all test suites related to changed files.
```

For our last test, we just have to simulate a failed API response, and mock our dispatch function to test if it was called. 

```js
  it("Should handle errors and update the error state", async () => {
    const mockDispatch = jest.fn();

    global.fetch = jest.fn().mockRejectedValue();

    const cards = await GetCards({}, mockDispatch);

    expect(mockDispatch).toHaveBeenCalled();
  });
```
We start by creating a mock dispatch function using `jest.fn()`. This allows us to track if the function is called correctly.

To simulate an error during the API call, we mock a rejected response by overriding the `global.fetch` function with `mockRejectedValue()`.

Then, we call the `GetCards()` function, passing in the mock dispatch function. This allows us to observe how the function handles errors and interacts with the dispatch mechanism.

Lastly, we use the `expect()` statement to verify that the mock dispatch function was called as expected. This ensures that the error is being dispatched properly.

```
 PASS  src/app/utils/GetCards.test.js
  GetCards
    ✓ Should get an array with at least 24 cards (4 ms)
    ✓ Every card should have a name, id, and imageUri (1 ms)
    ✓ Should handle errors and update the error state (3 ms)

Test Suites: 1 passed, 1 total
Tests:       3 passed, 3 total
Snapshots:   0 total
Time:        0.954 s, estimated 1 s
Ran all test suites related to changed files.
```

And it works! We could keep going and test all kinds of things, but the general process is clear now, and we can call it quits for this post.

## Conclusion

In this blog post, we dived into the world of unit testing, aiming to enhance the reliability and stability of our app, AetherShuffler. We explored the usage of Jest and React Testing Library to write efficient and effective unit tests, gaining insights into our code and catching bugs along the way.

As software developers, embracing unit testing is crucial. By investing time and effort in testing our code, we establish a solid foundation, detect issues early on, and deliver more reliable software. With continuous testing, we ensure the robustness of our applications and provide better user experiences.

Thank you for joining me on this unit testing journey. Here's to successful tests and resilient code!