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

 I knew this was going to be a pretty simple program, so I didn't want to have to write a whole api for authenticating. Luckily, the Firebase authentication SDK really simplified the process. 
 
 
 
 Similarly, I didn't want to get lost in the weeds of styling, so I decided to use Tailwind CSS and their prebuilt ui components. In doing so I've unintentionally fallen in love with Tailwind CSS, but more on that later. 