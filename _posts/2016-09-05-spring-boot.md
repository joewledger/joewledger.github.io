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

The next question is, how do we parse this data and store it logically in our program? Perhaps we can create a User class and store all of our user's data in an instance of a User object?

Lets define the following class:

{% highlight java %}

public class User {

    private String first;
    private String last;
    private String email;
    private String password;

}

{% endhighlight %}

<br/>

Of course we need a way to modify and access the data, so lets add some getters and setters.

{% highlight java %}

public class User {

    private String first;
    private String last;
    private String email;
    private String password;

    public String getFirst() {return first;}

    public void setFirst(String first) {this.first = first;}

    public String getLast() {return last;}

    public void setLast(String last) {this.last = last;}

    public String getEmail(){return email;}

    public void setEmail(String email) {this.email = email;}

    public String getPassword(){return password;}

    public void setPassword(String password) {this.password = password;}

}

{% endhighlight %}

<br/>

Now we get to whats cool about Spring Boot. If we tell our API endpoint to accept a User object as a parameter, it will automatically parse the JSON data and populate a User object for us. For all setter methods, Spring Boot will look for those fields in the JSON object and attempt to populate the object with their values.

The same can be done for creating JSON objects that are sent back to the client. If we return a ResponseEntity object of type User at an API endpoint, a JSON Object with all the fields corresponding to the getter methods of the User object will be returned.

<br/>

For example, lets create an API endpoint that accepts information about a User and returns the same information with the password replaced by asterisks.

{% highlight java %}

@RestController
public class FuzzPassword {

    @RequestMapping(value="/api/fuzz", method=RequestMethod.POST, produces="application/json",
                                                 consumes = MediaType.APPLICATION_JSON_UTF8_VALUE)
    public ResponseEntity<User> fuzzPassword(@RequestBody User user) {

        String password = user.getPassword();
        StringBuilder builder = new StringBuilder(password.length());
        for(int i = 0; i < password.length(); i++){
            builder.append("*");
        }
        user.setPassword(builder.toString());
        return new ResponseEntity<>(user, HttpStatus.OK);
    }
}
{% endhighlight %}

<br/>

Now for an input JSON object of 

{"first": "Joe", "last": "Ledger", "email": "example@example.com", "password": "fakepassword"}

<br/>

We'll get an output of

{"first": "Joe", "last": "Ledger", "email": "example@example.com", "password": "\*\*\*\*\*\*\*\*\*\*\*\*"}

<br/>

Now lets think about the case in which we want to return an authentication token, user id, and login status to the user after they submit their credentials. The following class should take care of that (getters and setters omitted for brevity).

{% highlight java %}

public class LoginResponse {

    private int code;
    private String message;
    private int uid;
    private String authToken;
}
{% endhighlight %}

<br/>

I won't delve into how to store and validate user passwords. But assuming that your database queries are set up correctly, we can use the following code to grant a user a LoginResponse when they give their credentials.

{% highlight java %}

@RestController
public class CreateAccount {

    @RequestMapping(value="/api/account/create", method=RequestMethod.POST, produces="application/json",
                                                 consumes = MediaType.APPLICATION_JSON_UTF8_VALUE)
    public ResponseEntity<LoginResponse> createAccount(@RequestBody User user) throws SQLException {

        int accountStatus = createAccountStatus(user);
        if(accountStatus == ErrorCodes.SUCCESS){
            return new ResponseEntity<>(Authentication.attemptLoginWithPassword(user), HttpStatus.OK);
        } else {
            return new ResponseEntity<>(new LoginResponse(accountStatus), HttpStatus.OK);
        }

    }

    /*
    Attempts to insert a user into a database using the information provided in a User object.
    If this is not possible, gives the relevant status code.
    If the operation is successful, returns 1
     */
    public static int createAccountStatus(User user) {
        try {
            if(!user.createAccountPossible()){
                return ErrorCodes.INVALID_EMAIL_OR_PASSWORD;
            }
            if(DBQueries.userInDatabase(user)){
                return ErrorCodes.EMAIL_IN_USE;
            }
            if(!DBQueries.insertUserIntoDatabase(user)){
                return ErrorCodes.USER_UPDATE_ERROR;
            }
        } catch (SQLException|NullPointerException e) {
            return ErrorCodes.DATABASE_CONNECTION_ERROR;
        }
        return ErrorCodes.SUCCESS;
    }
}

{% endhighlight %}

<br/>

Similarly, we can grant an authentication token when logging in with existing credentials.

{% highlight java %}

@RestController
public class Login {

    @RequestMapping(value="/api/account/login", method= RequestMethod.POST, produces="application/json",
                                                consumes = MediaType.APPLICATION_JSON_UTF8_VALUE)
    public ResponseEntity<LoginResponse> login (@RequestBody User user) throws SQLException{

        if(!user.loginPossible()){
            return new ResponseEntity<>(new LoginResponse(ErrorCodes.INVALID_EMAIL_OR_PASSWORD), HttpStatus.OK);
        }

        return new ResponseEntity<>(Authentication.attemptLoginWithPassword(user), HttpStatus.OK);
    }
}

{% endhighlight %}

<br/>

And there you have it! A simple way to set up interfaces for logging in and creating accounts with Java Spring Boot.
