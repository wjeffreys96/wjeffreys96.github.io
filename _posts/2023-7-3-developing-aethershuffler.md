---
layout: post
title: Developing AetherShuffler
date: 2023-07-03 00:00:00 -0700
description: Developing AetherShuffler
categories: [AetherShuffler, React, Firebase]
tags: [AetherShuffler, React, Firebase]
Author: William Jeffreys
---

When I set out to develop AetherShuffler, I had a few goals in mind. I wanted to create a tool that would allow me to explore the vast quantities of cards available to me as a Magic: The Gathering player, and I also wanted to try out Next.js and Tailwind while I did it. For simplicity's sake, I've narrowed the scope of this app to work specifically for the Commander format. Magic has over 25,000 cards in it, so one of the hardest things to do when building a deck is finding new cards to put in them. I thought that if there was a way to "shuffle" up a list of random-ish cards and be able to save what you find, that it would turn that step of the process into something pretty fun.

I figured there were 3 main tasks to complete:

1. Create the authentication flow.
2. Create the card search and display functionality.
3. Connect with a database to store user's favorited cards.

## Authentication

I knew this was going to be a pretty simple program, so I didn't want to have to write a whole api for authenticating. Luckily, the Firebase authentication SDK really simplified the process. The first step was to initialize the Firebase app:

```js
    import { initializeApp } from "firebase/app";

    const firebaseConfig = {
    ...
    };

    const firebaseApp = initializeApp(firebaseConfig);

    export default firebaseApp;
```

Now with the app available to us, we can create the login screen. This was a bit tricky in Next.js. I had a lot of issues with infinite loops and imports not working. But I finally got it to work using a combination of an async useCallback and a useEffect:

```js
const auth = getAuth(firebaseApp);
const ctx = useContext(AuthContext);
const router = useRouter();

const loadFirebaseUI = useCallback(async () => {
  const firebaseui = await import("firebaseui");
  const ui =
    firebaseui.auth.AuthUI.getInstance() || new firebaseui.auth.AuthUI(auth);
  const uiConfig = {
    signInOptions: [
      firebase.auth.GithubAuthProvider.PROVIDER_ID,
      firebase.auth.GoogleAuthProvider.PROVIDER_ID,
    ],
    callbacks: {
      signInSuccessWithAuthResult: function (authResult) {
        const db = getDatabase();
        const user = authResult.user;
        const userId = user.uid;
        set(ref(db, "users/" + userId), {
          username: user.displayName,
        });
        ctx.dispatch({ type: "SETUSER", payload: user });
        router.push("/dashboard");
      },
    },
  };

  ui.start(".firebase-auth-container", uiConfig);
}, []);

useEffect(() => {
  loadFirebaseUI();
}, []);
```

We start by defining the function that will load the Firebase login form. It's placed inside a useCallback hook to memoize the function. That combined with the empty dependency array ensures that the function only runs once, preventing the infinite loop. We need this function to run asynchronously because firebaseui must be imported before we can get the instance from the sdk.

Under sign in options I'm allowing oauth with both Github and Google. The callbacks section allows you to run any functions once a succesful authentication occurs, so there I:

1.  Connect with the database and add a user and their userID.
2.  Dispatch a reducer action stored inside AuthContext to set the user to the user that Firebase returns to me.
3.  Navigate to the user's dashboard.

Inside AuthContext.js we see what happens with the user that we store:

```js
"use client";
import React, { createContext, useEffect, useReducer } from "react";
import { getAuth, onAuthStateChanged, signOut } from "firebase/auth";
import firebaseApp from "../lib/firebase";

const auth = getAuth(firebaseApp);

const initialState = {
  user: null,
  token: null,
};

const reducer = (state, action) => {
  switch (action.type) {
    case "SETUSER":
      const user = action.payload;
      const token = user.accessToken;
      return {
        ...state,
        user: user,
        token: token,
      };
    case "LOGOUT":
      localStorage.removeItem("user");
      localStorage.removeItem("token");
      signOut(auth);
      return {
        ...state,
        user: null,
        token: null,
      };

    default:
      return state;
  }
};

export const AuthContext = createContext(initialState);

export const AuthProvider = ({ children }) => {
  const [state, dispatch] = useReducer(reducer, initialState);

  useEffect(() => {
    onAuthStateChanged(auth, (user) => {
      if (user) {
        dispatch({ type: "SETUSER", payload: user });
      }
    });
  }, []);

  const authContextValue = {
    user: state.user,
    token: state.token,
    dispatch,
  };

  return (
    <AuthContext.Provider value={authContextValue}>
      {children}
    </AuthContext.Provider>
  );
};
```

Within the SETUSER case we store the user and their access token and update the app-wide state accordingly. We also have a case for logging out that removes all of the stored data.

Further down we have a useEffect with Firebase's `OnAuthStateChanged()` listener inside. This is so that everytime the app is rerendered, it goes and grabs the user's data again so that they remain logged in. Otherwise the user would just be logged out everytime something changes.

While mostly I expect people to sign in using the oauth options, I do provide a simple way of signing up via email and password. This is achieved in ~/auth/signup/page.js. There we find a modified TailwindUI form with a reducer I made to handle the state. The magic happens when the form is submitted:

```js
const handleSubmit = (e) => {
  e.preventDefault();
  const email = emailRef.current.value;
  const password = passwordRef.current.value;
  const username = usernameRef.current.value;

  const formIsValid =
    state.emailValid &&
    state.passwordValid &&
    state.passwordAgainValid &&
    state.usernameValid;

  if (formIsValid) {
    try {
      RegisterNewUser(auth, email, password, username, ctx, router, dispatch);
    } catch (error) {
      console.log("register error: ", error);
    }
  } else {
    console.log("form is not valid");
  }
};
```

`RegisterNewUser()` is defined inside ~/utils/RegisterNewUser.js:

```js
import { getDatabase, ref, set } from "firebase/database";
import { createUserWithEmailAndPassword } from "firebase/auth";

export default function RegisterNewUser(
  auth,
  email,
  password,
  username,
  ctx,
  router,
  dispatch
) {
  const db = getDatabase();

  createUserWithEmailAndPassword(auth, email, password)
    .then((userCredential) => {
      // Signed in
      const user = userCredential.user;
      const uid = user.uid;
      ctx.dispatch({ type: "SETUSER", payload: user });
      set(ref(db, "users/" + uid), {
        username: username,
      });
      router.push("/dashboard");
    })
    .catch((error) => {
      if (error.code === "auth/email-already-in-use") {
        dispatch({
          type: "SET_ERROR_GENERAL",
          value: "That email is already in use",
        });
      }
    });
}
```

First we grab the user input, then we check if it's valid using Yup object validation. Then, if it's valid we pass in the user input to RegisterNewUser.

There, Firebase has given us the function `CreateUserWithEmailAndPassword()`. We just feed it the input and it does the rest of the work for us. Simple.

With that, Authentication is finished and we can move on to the fun part, cards!

## Card Search and Display

Being able to search and display Magic's 25,000+ cards seems like a daunting task, but luckily we have Scryfall's API to help out. They provide a REST-like interface to query card information, including images of the card faces, and this is what I use in AetherShuffler.

First I provide a simple form for the user to fill out. This provides the parameters I use when querying the API. This form is found in ~/components/CardForm.js. On submit, we call `GetCards()` which we import from ~/utils/GetCards.js, and pass in the users input:

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
    const randomSubset = [];

    for (let i = 0; i < numCards; i++) {
      const randomIndex = Math.floor(Math.random() * array.length);
      const selectedCard = array.splice(randomIndex, 1)[0];
      randomSubset.push(selectedCard);
    }
    return randomSubset;
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

  const cardData = `cardDataMaker()`;

  return cardData;
}
```

The `URLSearchParams()` may look like gibberish to you, especially if you've never played Magic before, but there is a method to this madness. The Scryfall API query syntax is designed to allow you to search for Magic: The Gathering cards based on various criteria. It leverages specific keywords and operators to build complex search queries. Here's a breakdown of how the keywords work and their relationship to the game mechanics:

1.  Color Filtering: The `color_id` parameter is used to filter cards based on their color. In the code snippet, the colorId variable is used to construct the color filter. The `f:commander id<=` part of the query checks for cards that are legal in the Commander format and have a color identity less than or equal to the specified colorId.

2.  Colorless Filtering: The `colorId != "Colorless" ? "-c:c" : ""` part of the query ensures that colorless cards are not included in the search results unless you specifically want them to be.

3.  Card Type Filtering: The `card_type` parameter allows you to filter cards based on their type, such as creatures, enchantments, or artifacts. The `cardType` variable is used to construct the type filter. The `t:` part of the query checks for cards that have the specified card type.

4.  Sorting: The `order:edhrec dir:asc` part of the query specifies the sorting order of the search results. In this case, the order:edhrec sorts the cards based on their popularity in the Commander format (EDH), and dir:asc sorts them in ascending order.

5.  Card Function Filtering: The `card_function` parameter allows you to filter cards based on their function or role in the game, such as "ramp," "card draw," "removal," etc. The `cardFunction` variable is used to construct the function filter. The `oracletag:` part of the query checks for cards that have the specified function as an Oracle tag. There is more information on what Oracle Tags are on the Scryfall website.

Now that we've established the parameters and constructed the URL, we create a function called `selectRandomCards()`. This function takes in an array of cards and the desired number of cards to be selected from that array.

The purpose of `selectRandomCards()` is twofold. Firstly, it provides a "shuffle" effect to the cards. Secondly, it limits the number of cards returned by the `GetCards()` function. Normally, the API returns approximately 500 cards, even with pagination. Working with such a large number of cards can be unwieldy, so we set a limit of 24 cards.

Furthermore, the cards returned by the API are ordered based on popularity. Consequently, you would only see the most commonly played cards. To address this, we order the array in such a way that the cards you see when shuffling are still relevant in the commander format, but we introduce randomness to ensure that you also come across cards you may not have seen before.

This randomness is achieved using a for loop within the function.

```js
function selectRandomCards(array, numCards) {
  const randomSubset = [];

  for (let i = 0; i < numCards; i++) {
    const randomIndex = Math.floor(Math.random() * array.length);
    const selectedCard = array.splice(randomIndex, 1)[0];
    randomSubset.push(selectedCard);
  }
  return randomSubset;
}
```

At this point, our card array looks like this:

```js
[
  {
    "object": "card",
    "id": "0b8aff2c-1f7b-4507-b914-53f8c4706b3d",
    "oracle_id": "6c2c8bf3-9bf8-4a86-89d3-3bb36260dc51",
    "multiverse_ids": [
      485496
    ],
    "mtgo_id": 81573,
    "arena_id": 71955,
    "tcgplayer_id": 215343,
    "cardmarket_id": 466274,
    "name": "Azusa, Lost but Seeking",
    ...
  },
  {
    "object": "card",
    ...
  }
]
```

I've removed about 2000 pieces of data from that list. The API provides an overwhelming amount of information for each card, much more than what we actually need. To simplify things, I created a function called `cardDataMaker()`.

```js
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
```

The purpose of `cardDataMaker()` is to extract specific data from the randomCards array, which is generated by `selectRandomCards()`. Within `cardDataMaker()`, we iterate over each card in the randomCards array and extract the name, ID, and image URIs of the cards.

Now, here's where things get a bit more interesting. In Magic, some cards have artwork on both sides. Initially, when I attempted to extract the image URIs, the app would break because cards with artwork on both sides don't have a card.image_uri property. Instead, they have a card_faces array containing the two image_uris for each side of the card.

To handle this scenario, I added a check within `cardDataMaker()`. If a card has image_uris, we assign the value of card.image_uris.normal to the imageUri variable. However, if the image_uris property is absent, we assume the card has card_faces and assign the value of card.card_faces[0].image_uris.normal to the imageUri variable. This approach has proven to work flawlessly in handling both cases.

Now our array looks like this:

```js
[
  {
    "name": "Dockside Extortionist",
    "id": "9e2e3efb-75cb-430f-b9f4-cb58f3aeb91b",
    "imageUri": "https://cards.scryfall.io/normal/front/9/e/9e2e3efb-75cb-430f-b9f4-cb58f3aeb91b.jpg?1673147774"
  },
  {
    "name": "Esper Sentinel",
    "id": "f3537373-ef54-4578-9d05-6216420ee349",
    "imageUri": "https://cards.scryfall.io/normal/front/f/3/f3537373-ef54-4578-9d05-6216420ee349.jpg?1626093502"
  },
  ...
]
```

Hooray! Simple card data. With this simplified array, we can move on to displaying the cards and allowing users to interact with them.

To render the list of cards, I built a component called CardList.js, found in ~/components/CardList.js. This component ingests the cardData array returned by `getCards.js` and runs it through a function called `cardListMaker()`.

```js
const cardListMaker = () => {
  if (cardData) {
    try {
      return cardData.map((card) => {
        const isFavorited = favoritedCards.includes(card.id);
        const isOverlay = overlayCard === card.id;

        return (
          <div key={card.name} className="relative flex flex-col ">
            <CardLayout
              AddToFavorites={AddToFavorites}
              key={card.name}
              name={card.name}
              id={card.id}
              imageUri={card.imageUri}
              onClick={() => setOverlayCard(card.id)}
              isOverlay={isOverlay}
              isFavorited={isFavorited}
              hideOverlay={hideOverlay}
            />
          </div>
        );
      });
    } catch (error) {
      return <div className="text-red-600 my-12">No Cards Found!</div>;
    }
  } else {
    return <div className="text-red-600 my-12">No Cards Found!</div>;
  }
};
```

This function checks if cardData exists and is truthy, a necessary check as cardData may not return anything if the user searches something that doesn't exist, and then it tries to render a list of components called `CardLayout` found in ~/components/CardLayout.js.

```js
export default function CardLayout({
  name,
  id,
  imageUri,
  onClick,
  isOverlay,
  isFavorited,
  hideOverlay,
  AddToFavorites,
}) {
  const handleOverlayClick = (event) => {
    event.stopPropagation();
    hideOverlay();
  };

  return (
    <>
      <div key={id} className="flex justify-center my-5">
        <img
          className="cursor-pointer"
          onClick={onClick}
          height="80%"
          width="80%"
          src={imageUri}
          alt={name}
        />
        {isOverlay && (
          <div
            className="absolute top-0 left-0 flex items-center justify-center w-full h-full"
            onClick={handleOverlayClick}
          >
            <button
              className="w-24 h-24 flex items-center justify-center opacity-50 hover:opacity-75 rounded-full bg-black"
              onClick={(event) => {
                event.stopPropagation();
                AddToFavorites(name, id, imageUri);
              }}
            >
              <span className="text-white text-lg">
                {isFavorited ? "✔" : "+"}
              </span>
            </button>
          </div>
        )}
        <br />
        <hr />
      </div>
    </>
  );
}
```

Together, these two components form the list of cards that you see when you press "Shuffle" on the home page of the app. You'll also notice some logic that handles adding the cards to favorites, which I'll cover in the last section about saving cards to user favorites. The cards themselves are just divs with a background image sourced from the image_uri property that Scryfall provides us. The user can tap/click the card to reveal a button to add the cards to their favorites.

So far, we've covered Authentication and Card Search and Display, leaving only one more thing to go over, saving the cards to a database.

## Storing User Favorites

Since I was already using Firebase to authenticate, it made sense to utilize their Realtime Database to store users favorited cards. Integrating this was very simple.

We've already connected to the firebase app when we set up authentication, so now when we want to access the database, all we have to do is call `getDatabase()` and firebase will handle the rest. For CRUD operations we have `set()` for creating and updating, `onValue()` for reading, and `delete()` for deleting.

Within the scope of this app, when a user wants to add a card to their favorites they search for a list of cards, and once they find a card they want to save they tap it and click the '+' that appears. Behind the scenes this is what happens:

```js
function AddToFavorites(name, id, imageUri) {
  try {
    const db = getDatabase();
    const user = ctx.user;
    const uid = user.uid;
    set(ref(db, "users/" + uid + "/favorites/" + id), {
      name: name,
      id: id,
      imageUri: imageUri,
    }).then(() => {
      setFavoritedCards((prevFavorites) => [...prevFavorites, id]);
    });
  } catch (error) {
    if (error.message === "Cannot read properties of null (reading 'uid')") {
      setError("Please sign in to add cards to favorites.");
    } else {
      setError(error.message);
    }
  }
}
```

First we try to connect to the database. Then we grab the user info from our context and create an entry into the database under the user's uid that holds the favorited cards name, id, and image_uri.

The `setFavortiedCards()` call is logic for tracking which displayed cards have already been favorited. This provides feedback to the user in the form of a checkmark that replaces the '+' on the button so they know that the card has already been favorited.

On the user's dashboard, located at ~/components/dashboard/page.js, we display all their favorited cards, and allow deletion of favorites.

```js
const handleDelete = (id) => {
  const deleteRef = ref(db, "users/" + ctx.user.uid + "/favorites/" + id);
  remove(deleteRef);
};

const getFavorites = () => {
  const faveRef = ref(db, "users/" + ctx.user.uid + "/favorites");
  onValue(faveRef, (snapshot) => {
    const dataArray = [];
    const data = snapshot.val();
    for (let key in data) {
      dataArray.push(data[key]);
    }
    setFavorites(dataArray);
  });
};
```

Deletion is very simple and I think speaks for itself. Reading the user's favorites is also very simple. We make a reference to the currently signed in user and pass it into `onValue()`. This returns a JSON blob that I then convert to an array by looping over it and pushing the values to `dataArray`.

Then, we map over the array and again create a list of `CardLayout` components, this time with a button underneath that allows the user to delete the card from their favorites.

```js
<div className="h-auto flex justify-center flex-wrap px-3">
  {favorites.length > 0 ? (
    favorites.map((card) => {
      return (
        <div key={card.id} className="flex flex-col">
          <CardLayout
            key={card.name}
            name={card.name}
            id={card.id}
            imageUri={card.imageUri}
          />
          <div className="flex justify-center">
            <button
              // key={card.name}
              className="bg-red-500 hover:bg-red-700 text-white text-sm rounded py-2 px-6"
              onClick={() => {
                handleDelete(card.id);
              }}
            >
              <svg
                xmlns="http://www.w3.org/2000/svg"
                fill="none"
                viewBox="0 0 24 24"
                strokeWidth="1.5"
                stroke="currentColor"
                className="w-6 h-6"
              >
                <path
                  strokeLinecap="round"
                  strokeLinejoin="round"
                  d="M14.74 9l-.346 9m-4.788 0L9.26 9m9.968-3.21c.342.052.682.107 1.022.166m-1.022-.165L18.16 19.673a2.25 2.25 0 01-2.244 2.077H8.084a2.25 2.25 0 01-2.244-2.077L4.772 5.79m14.456 0a48.108 48.108 0 00-3.478-.397m-12 .562c.34-.059.68-.114 1.022-.165m0 0a48.11 48.11 0 013.478-.397m7.5 0v-.916c0-1.18-.91-2.164-2.09-2.201a51.964 51.964 0 00-3.32 0c-1.18.037-2.09 1.022-2.09 2.201v.916m7.5 0a48.667 48.667 0 00-7.5 0"
                />
              </svg>
            </button>
          </div>
        </div>
      );
    })
  ) : (
    <EmptyState />
  )}
</div>
```

And that's all there is to it! Firebase made the backend of this app incredibly simple and easy, which was perfect for the scope of this application. 

## Conclusion

This app was a lot of fun to build. It presented some unique challenges, but having overcome those I feel like I'm a better programmer than I started, which is always the goal, right? 

The next post I'll be making will be about unit testing AetherShuffler, so stay tuned, and thanks for reading!