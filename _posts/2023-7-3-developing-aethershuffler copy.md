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
 
 We start by defining the function that will load the Firebase auth ui. It's placed in a callback to prevent infinite loops, and also because you can't use async functions in a useEffect. We need this to be async because we need to make sure we have the firebaseui imported before we can get the instance from the sdk. There are a few ways of d
 

Similarly, I didn't want to get lost in the weeds of styling, so I decided to use Tailwind CSS and their prebuilt ui components. In doing so I've unintentionally fallen in love with Tailwind CSS, but more on that later. 