---
layout: post
title: Designing an Authentication Procedure with Java Spring Boot
date:   2016-09-05 12:44:14
categories: others
---

[Java Spring Boot](http://projects.spring.io/spring-boot/) is a really cool web framework for building REST APIs. In this post, I will talk about what I like about Spring Boot, and show how you can use it to build a user authentication system.

<br/>

Suppose you are given a specification for a project in which users can make accounts and then use their accounts to access various services that you provide through a REST web interface. You need to design a simple interface that will allow users to create an account and then use their credentials request a temporary authentication token that can be used to access any service without a password.

<br/>

One logical solution to this problem would be to start with two simple interfaces, one for creating accounts and one for logging in with a set of credentials. We can assume that the credentials consist of an email and password combination. For this purpose, we can define two API endpoints: **/api/account/create** and **/api/account/login**.

<br/>

When /api/account/create is invoked, we will attempt to create an account with the provided credentials (checking that the email is not already in use and that the email and password meet our criteria). When /api/account/login is invoked, we will check to make sure that the email and password combination provided by the user is in our database, and then grant an authentication token.

<br/>

There are many ways that we can accept the user's credentials, but one easy way would be to accept them as a JSON object sent in an HTTP POST request. If the user has an email "example@example.com" and a password "fakepassword", we would expect the following JSON object from them when processing their login credentials:

<br/>

<p align="center">
{"email": "example@example.com", "password": "fakepassword"}
</p>

<br/>

When a user first makes an account, we might require them to give additional information, such as their first and last names. Then the JSON object might look something like this.

<br/>

<p align="center">
{"first": "Joe", "last": "Ledger", "email": "example@example.com", "password": "fakepassword"}
</p>

<br/>