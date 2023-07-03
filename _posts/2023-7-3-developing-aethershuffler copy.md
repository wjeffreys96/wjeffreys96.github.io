---
layout: post
title: Developing AetherShuffler
date: 2023-07-03 00:00:00 -0700
description: Developing AetherShuffler
categories: [AetherShuffler, React, Firebase]
tags: [AetherShuffler, React, Firebase]
Author: William Jeffreys
---

When I set out to develop AetherShuffler, I had a few goals in mind. I wanted to create a tool that would allow me to explore the vast quantities of cards available to me as a Magic: The Gathering player, and I also wanted to create a tool that would allow me to try out the Next.js app router. I figured there were 3 main tasks to complete: 

1. Create the authentication flow
2. Create the card search and display functionality
3. Connect with the database to store the user's favorited cards

## Authentication

 I knew this was going to be a pretty simple program, so I didn't want to have to create a whole api for authentication and storing user data. Luckily, integrating the Firebase authentication SDK and the database services really simplified the process. To authenticate they provide a
 
 
 
 Similarly, I didn't want to get lost in the weeds of styling, so I decided to use Tailwind CSS and their prebuilt ui components. In doing so I've unintentionally fallen in love with Tailwind CSS, but more on that later. For the card data and images, The Scryfall API had me covered. I was able to get all the data I needed from them, and they even have a CDN for the card images.