---
layout: post
title: Developing AetherShuffler
date: 2023-07-03 00:00:00 -0700
description: Developing AetherShuffler
categories: [AetherShuffler, React, Firebase]
tags: [AetherShuffler, React, Firebase]
Author: William Jeffreys
---

When I set out to develop AetherShuffler, I had a few goals in mind. I wanted to create a tool that would allow me to explore the vast quantities of cards available to me as a Magic: The Gathering player, and I also wanted to try out Next.js while I did it. Magic has over 25,000 cards in it, so one of the hardest things to do when building a deck is finding new cards to put in them. I thought that if there was a way to "shuffle" up a list of random-ish cards and be able to save what you find, that it would turn that step of the process into something pretty fun.

I figured there were 3 main tasks to complete:

1. Create the authentication flow.
2. Create the card search and display functionality.
3. Connect with a database to store user's favorited cards.

## Authentication

I knew this was going to be a pretty simple program, so I didn't want to have to write a whole api for authenticating. Luckily, the Firebase authentication SDK really simplified the process. The first step was to initialize the Firebase app:

    import { initializeApp } from "firebase/app";

    const firebaseConfig = {
    ...
    };

    const firebaseApp = initializeApp(firebaseConfig);

    export default firebaseApp;

Now with the app available to us, we can create the login screen. This was a bit tricky in Next.js. I had a lot of issues with infinite loops and imports not working. But I finally got it to work using a combination of an async useCallback and a useEffect:

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

We start by defining the function that will load the Firebase Log in form. It's placed inside a useCallback hook to memoize the function. That combined with the empty dependency array ensures that the function only runs once, preventing the infinite loop. We need this function to run asynchronously because firebaseui must be imported before we can get the instance from the sdk.

Under sign in options I'm allowing oauth with both Github and Google. The callbacks section allows you to run any functions once a succesful authentication occurs, so there I:

1.  Connect with the database and add a user and their userID.
2.  Dispatch a reducer action stored inside AuthContext to set the user to the user that Firebase returns to me.
3.  Navigate to the user's dashboard.

Inside AuthContext.js we see what happens with the user that we store:

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

Within the SETUSER case we store the user and their access token and update the app-wide state accordingly. We also have a case for logging out that removes all of the stored data.

Further down we have a useEffect with Firebase's OnAuthStateChanged() listener inside. This is so that everytime the app is rerendered, it goes and grabs the user's data again so that they remain logged in. Otherwise the user would just be logged out everytime something changes.

While mostly I expect people to sign in using the oauth options, I do provide a simple way of signing up via email and password. This is achieved in ~/auth/signup/page.js. There we find a modified TailwindUI form with a reducer I made to handle the state. The magic happens when the form is submitted:

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

RegisterNewUser() is defined inside ~/utils/RegisterNewUser.js:

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

First we grab the user input, then we check if it's valid using Yup object validation. Then, if it's valid we pass in the user input to RegisterNewUser.

There, Firebase has given us the function CreateUserWithEmailAndPassword. We just feed it the input and it does the rest of the work for us. Simple. 

With that, Authentication is finished and we can move on to the fun part, cards!

## Card Search and Display

Querying and displaying cards was an interesting challenge made possible by the Scryfall API. 