---
title: "Why I love Elm"
description: "My first projects in Elm"
date: 2022-03-15T17:01:40+01:00
lastmod: 2022-03-15T17:01:40+01:00
draft: false
author: "Auryn Engel"

tags: ["HTML", "JavaScript", "CSS", "Elm"]
categories: ["Development"]

---
## Why bother?

The first time I tried elm was 2018, but at that time my frontend experience was still very small. At that time I had implemented a larger project in `react` and gained my first experience with `vue`. Elm had just overwhelmed me and I didn't know the concepts. So I was choosing to react all the time. For my private projects as well as for my professional ones.

My first fundamental functional experience was at the university with `haskell`. At that time I was overwhelmed from it like 2018 with `elm`. It was a completely different approach than OOP with `Java` or some basic lowlevel `C`. I hated it.

Then, 4 years later and 3 years more professional experience, I saw [this](https://www.youtube.com/watch?v=3n17wHe5wEw) talk from **Richard Feldman**. It opened my eyes. I had already some experience with rust in a professional context and was already hooked to functional programming. But I didn't see any need for pure functional for frontends. I mean, you already can write functional components in react, so why bother?

Then I remember all the pain of debugging and searching for issues with react components. How often does a component does not rerender or show its state because of some misconfigured or misused `useState` hooks? How often did I forget to not use `useState` with an object instead of spreading the values? So I wanted to give it a try.

## Second try

First, it was the same pain with `elm` as it was back in 2018 when I just wanted to bang my head against the screen or a near wall. But then I understood the concept of `elm`. It is just the same as with redux. There are messages, there is a model and there is a view based on the model that can trigger new messages. That's it. There cannot be anything else. The view always represents the model. There cannot be a model, that is not currently represented as a view and there cannot be any value in the model, that is not also in the view. The messages can be sent from the view, but the view cannot change the model in another way than over a message. Everything is strictly typed and is a function. It is super easy and complex at the same time.

It is explained nice in this image from [plaxdan](https://github.com/plaxdan/elm-lifecycle/blob/master/README.md):

![Elm lifecycle](/img/elm/elm-lifecycle.png)

### My first projects

And this is already the base concept. So I did some beginner tutorials, some starting projects, and easy `hello-world` pages. Then I did a small frontend for a ticketing system, but also really basic. To learn it in more depth I wanted to create something nice. Some real-world page. I saw the [basic-computer-games](https://github.com/coding-horror/basic-computer-games) repository and could not find any `elm` implementation. So I took the challenge and build the first game [Acey Ducey](https://github.com/coding-horror/basic-computer-games/tree/main/00_Alternate_Languages/01_Acey_Ducey/elm) in Elm. And then also another one [Awari](https://github.com/coding-horror/basic-computer-games/tree/main/00_Alternate_Languages/04_Awari), just for fun and for me. You can find these games live here: [Acey Ducey](https://auryn31.github.io/01_Acey_Ducey/) and [Awari](https://auryn31.github.io/04_Awari/).

### Example

To understand the basic concepts of `elm` I can highly recommend the [elm-guide](https://guide.elm-lang.org/). Here is a super easy example:

```elm
import Browser
import Html exposing (Html, button, div, text)
import Html.Events exposing (onClick)

main =
  Browser.sandbox { init = 0, update = update, view = view }

type Msg = Increment | Decrement

update msg model =
  case msg of
    Increment ->
      model + 1

    Decrement ->
      model - 1

view model =
  div []
    [ button [ onClick Decrement ] [ text "-" ]
    , div [] [ text (String.fromInt model) ]
    , button [ onClick Increment ] [ text "+" ]
    ]
```

A more detailed and more complex example is the [Awari](https://github.com/auryn31/01_Acey_Ducey/blob/main/src/Main.elm) game.

## Helpful resources

Some more interesting resources I found helpful while learning `elm`:

- [Json to Elm](https://noredink.github.io/json-to-elm/)
- [How to do CSS with Elm](https://elmprogramming.com/model-view-update-part-2.html)
- [How to optimize the Elm code](https://github.com/elm/compiler/blob/master/hints/optimize.md)
- [HTML to Elm](https://mbylstra.github.io/html-to-elm/)
- [Elm-live](https://www.elm-live.com/)

The part with css is more complex. There are different ways of doing so. I preferred the `Html.style` usage, since it is easy to use. But it is not typed and I am missing an easy-to-use `SCSS` usage. If I will use it in the future and have an easy example, I will create a post about it.

Also really helpful was [Elm-live](https://www.elm-live.com/). With the `---debug` command you can easily jump between the states and take a look at the views. It helped a lot during dev, also with the proxy command `--proxy-prefix=/api --proxy-host=http://localhost:8080/api`. My standard command during dev was: `elm-live src/Main.elm --proxy-prefix=/api --proxy-host=http://localhost:8080/api --open --start-page=resources/index.html -- --output=app.js --debug`. This build the `Main.elm` file into the `app.js` and opened my start page. I also used and `index.html` file to include some external resources.

## Conclusion

That is all for the moment. I am really in love with `elm` as I was with `svelt` after I used it the first time. There are new languages, frameworks, and concepts all the time. I try my best to be open-minded and think there is a better way of frontend development than react. Until now, I saw multiple ones. I would prefer `svelte` and also I would prefer `elm`. At the moment I would prefer `elm` over all the others. Mostly because of its type secure and immutability. But in projects, it always depends on the environment. For myself, I choose `elm` over all the others. For projects with clients and coworkers who have no experience in `elm` or in functional languages in general, I think `React` is still the most spread library to use.

A big thanks go to [Richard Feldman](https://www.youtube.com/watch?v=3n17wHe5wEw) and [Annaia Berry](https://www.youtube.com/watch?v=RFrKffrKCeU) for these amazing talks and super lectures!

I am a bit proud, that my first bigger `elm` projects are already example code for [basic-computer-games](https://github.com/coding-horror/basic-computer-games) and I hopefully will use `elm` more in the future. If you have any suggestions or questions, feel free to reach out to me.
