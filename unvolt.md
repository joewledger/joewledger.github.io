---
layout: page
title: UnVolt
permalink: /projects/unvolt
---

The [UnVolt App](http://www.unvoltapp.com/#about-unvolt) is an Android/iOS mobile application that helps users make passive behavioral changes that minimize energy use and promote sustainability. Over the past summer, I worked on improving the backend of the UnVolt Engine. One major contribution that I made was discovering and fixing a security issue with the UnVolt Engine. Previously, the Engine was not salting the hashes of user's passwords, leaving us vulnerable to a password breach if an attacker gained access to the server. I was able to fix the issue and improve the overall security of the system.

After fixing the security issue in the existing backend, I began to prototype a long-term replacement for the backend. I used Java Spring Boot and PostgreSQL to create a REST API in which the mobile applications could communicate with the server via JSON objects passed in HTTPS POST requests. By the end of the summer, I had finished implementing the User Authentication process, as well as providing mechanisms for giving users recommendations, savings summaries, and allowing users to follow or disregard recommendations in the application.

The iOS version of the UnVolt application is in beta testing will be released in the near future. We anticipate releasing the Android version of the UnVolt application by December of 2016.