---
title: "Elm AWS Cognito Login/Register Template with TypeScript"
description: "Example/Template of how to use AWS Cognito with Elm"
date: 2022-06-15T17:01:16+02:00
lastmod: 2022-06-15T17:01:16+02:00
draft: false
author: "Auryn Engel"

tags: ["Elm", "AWS", "Cognito", "TypeScript"]
categories: ["development", "template"]

---

For a current small project I needed a login/IAM provider. Here I immediately thought of aws cognito. The project is completely on aws, so I wanted to set the login to cognito. Since I work with Elm, I went directly to the search, for libs and examples. But as so often in Elm, it's just faster to do it yourself. Here you also have complete control over the source code. After a little research I found the [article](https://medium.com/@charlotteneill/elm-with-aws-cognito-8eae4fb858d0) by Charlotte Neill.
Her article is really great, but I wanted to add my own two cents. She has already mentioned all the advantages of cognito and elm there, so I won't go into details here.

For me it's more the communication from JS/TS to Elm that is exciting and I wanted to write the code in TypeScript, just to have more trust in my code. How exactly to create a UserPool on Cognito is described in Charlotte Neills article as well and can be found a lot on google.

In my code I use TypeScript and rollup to transpile and import the code. The code can be found on my [github](https://github.com/auryn31/elm-cognito-ts-example). These steps are defined in the `package.json`.

You can find why I like Elm so much [here](https://blog.auryn.dev/posts/starting-with-elm/) and introduction about Elm Ports [here](https://guide.elm-lang.org/interop/ports.html).

Working with Elm Ports is amazingly easy. You define the ports in a Port Module File, in this case the `Cognito.elm`. For a login for example:

```elm
port login : { emailAddress : String, password : String } -> Cmd msg
```

The login call is given an email address and a password. Important, you can only give one parameter to a function. In this case an object with the two parameters. The port creates a `Cmd msg`. So a JS function is called, but we ignore the result. We just tell js to do this with the parameters.

On the JS side we have to define this port. It will be subscribed to the login function of the ports. The data comes in in the previously defined format and the function is called when the command is triggered by Elm.

```ts
app.ports.login.subscribe((data: { emailAddress: string; password: string }) => {
    console.log('data: ' + JSON.stringify(data));
    // do stuff
});
```

But if we want a response from JS, we have to subscribe to a port. Similar steps are needed here. Again a port is defined, but this time it returns a `Sub msg`. In addition, it must accept a function that goes to `Msg`, since we want to get the messages back later.

```elm
port loginSuccess : (String -> msg) -> Sub msg
```

The JS part looks very similar again:

```ts
app.ports.loginSuccess.send('Eve');
```

A message is sent on the port.

Now we have to subscribe to it in the Elm code:

`Main.elm`:

```elm

type Msg
    = 
    ...
    | CognitoLoginSuccess String

subscriptions : Model -> Sub Msg
subscriptions _ =
        Cognito.loginSuccess CognitoLoginSuccess
```

If now in the JS code the one message is sent on the LoginSuccess port, we get a message at elm. You can imagine this quite well as a pipeline. The ports are described in more detail on the [documentation page](https://guide.elm-lang.org/interop/ports.html).

The Elm documentation is very good anyway. Most of it can be read there very informative. Overall it is very easy to call js from Elm and vice versa.

One mistake I made and searched for a while, if you don't use an Elm port, but use it in the js code as `subscribe`, then the application crashes. This is because Elm throws out code that is not used, so you call the `subscribe` on `undefined` because the `app.ports` do not know the function.

Otherwise the login and register part is strait forward.

```ts
// add port for login
app.ports.login.subscribe(function (data: { emailAddress: string; password: string }) {

  var authenticationData = {
    Username: data.emailAddress,
    Password: data.password,
  };

  const authenticationDetails = new AmazonCognitoIdentity.AuthenticationDetails(authenticationData);

  var userData = {
    Username: data.emailAddress,
    Pool: userPool,
  };
  var cognitoUser = new AmazonCognitoIdentity.CognitoUser(userData);

  // login user
  cognitoUser.authenticateUser(authenticationDetails, {
    onSuccess: function (result) {
      // send success to port
      const user = userPool.getCurrentUser();
      app.ports.loginSuccess.send(user?.getUsername() ?? 'please refresh page');
    },
    onFailure: function (err) {
      var errorMessage = err.message || JSON.stringify(err);
      // send error to port
      app.ports.errors.send(errorMessage);
    },
  });
});
```

The Amazon Cognito code is used to log in or register: `amazon-cognito-identity-js`. After registration the user has to be unlocked in the AWS Console, then he can log in. The lib `amazon-cognito-identity-js` also stores the tokens and login information in the local storage, so for an initial load you can check for the current user `const currentUser = userPool.getCurrentUser();`. Here you get automatically the current logged in user.

And that's all it is. This is the complete login and auth flow with aws cognito.

All details can be found in the [Github Repo](https://github.com/auryn31/elm-cognito-ts-example).
