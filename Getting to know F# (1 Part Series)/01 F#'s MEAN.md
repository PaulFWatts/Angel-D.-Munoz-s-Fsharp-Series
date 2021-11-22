If there's something that people think (or used to think) when node is mentioned it's very likely to be the famous [MEAN](https://www.mongodb.com/mean-stack) stack which stands for Mongo Express Angular Node, there are also variants like [MERN](https://www.mongodb.com/mern-stack) which just swapped Angular -> React, the rest is the same node + mongo as the base of your web stack.

But is there an alternative in F#?

I'd say there's a **_SAFE_**er alternative but I'll leave that until the end, let's try to get 1-1 version for each part of the MEAN stack

There's not a lot of options here since most of the .NET landscape is about SQL, but if you have mongo databases you can indeed use Mongo from F#, you can do it via two libraries:

-   [Mongo .NET Driver](https://mongodb.github.io/mongo-csharp-driver/)
-   [Mondocks](https://github.com/AngelMunoz/Mondocks)

the first one is the official MongoDB driver for .NET which is written in C# but can be consumed from F# without many issues, the second one is a small library I wrote that provides you with [MongoDB Commands](https://docs.mongodb.com/manual/reference/command/) in a way that should be familiar if you are used to javascript, you can execute those with the mongo driver itself, either way, you can use both side-by-side so it's win-win for you. it's also worth mentioning that if you choose PostgreSQL you can also go [NoSQL](https://www.npgsql.org/doc/types/jsonnet.html) as well but to be honest I have not tried that route.

this is an interesting part when you come to F# because there is some variety when it comes to your web server framework

-   [Falco](https://github.com/pimbrouwers/Falco)
-   [Giraffe](https://giraffe.wiki/)
-   [Saturn Framework](https://saturnframework.org/)
-   [ASP.NET](https://dev.toall%20of%20the%20above%20are%20built%20on%20top%20of%20this%20one/)

Granted, it is not the thousands of JS frameworks there are but these will cover your use cases in a pinch the good thing is that if you find middleware/libraries compatible with ASP.NET then you'll be able to use those from any of the others! so win-win situation again

## [](https://dev.to/tunaxor/f-s-mean-1g2b#face-to-face)Face to Face

let's have a brief reminder of how an express app looks like (taken from express website)  

```
const express = require('express')
const app = express()
const port = 3000

app.get('/', (req, res) => {
  res.send('Hello World!')
})

app.listen(port, () => {
  console.log(`Example app listening at http://localhost:${port}`)
})
```

Enter fullscreen mode Exit fullscreen mode

this is our goal, to have something that looks as ergonomic (if not better) than this. Of course, I know once express Apps get bigger things don't look pretty anymore, but I believe F# provides better security in that aspect due to the top-down nature of F#

### [](https://dev.to/tunaxor/f-s-mean-1g2b#falco)Falco

Falco it's one of the most (if not the most in this article) slim libraries for ASP.NET  

```
module HelloWorld.Program

open Falco
open Falco.Routing
open Falco.HostBuilder

let helloHandler : HttpHandler =
    "Hello world"
    |> Response.ofPlainText

[<EntryPoint>]
let main args =
    webHost args {
        endpoints [ get "/" helloHandler ]
    }
    0
```

Enter fullscreen mode Exit fullscreen mode

As you can see here, we first define our handler which is basically passing down directly the type of response we want (text in this case), in our main function we create a new `webHost` and specify the routes. Plain simple right? Falco defines an `HttPHandler` as a function that takes the following form  

```
let handler =  
  fun (ctx: HttpContext) -> 
    task { return! "" |> Response.ofPlainText ctx }
```

Enter fullscreen mode Exit fullscreen mode

this is a difference to express which decides to expose both `req`, `res` objects, in falco they are present inside the HTTP context `ctx`

### [](https://dev.to/tunaxor/f-s-mean-1g2b#giraffe)Giraffe

Giraffe is a more popular option which is also more mature that provides a similar flavor to falco  

```
let webApp =
    choose [
        route "/ping"   >=> text "pong"
        route "/"       >=> htmlFile "/pages/index.html" ]

type Startup() =
    member __.ConfigureServices (services : IServiceCollection) =
        // Register default Giraffe dependencies
        services.AddGiraffe() |> ignore

    member __.Configure (app : IApplicationBuilder)
                        (env : IHostingEnvironment)
                        (loggerFactory : ILoggerFactory) =
        // Add Giraffe to the ASP.NET Core pipeline
        app.UseGiraffe webApp

[<EntryPoint>]
let main _ =
    Host.CreateDefaultBuilder()
        .ConfigureWebHostDefaults(
            fun webHostBuilder ->
                webHostBuilder
                    .UseStartup<Startup>()
                    |> ignore)
        .Build()
        .Run()
    0
```

Enter fullscreen mode Exit fullscreen mode

there's a lot more to look in here right? the main reason for that is that this same startup and host code are hidden behind the `webHost` builder in the `Falco` sample but as I mentioned before, both are built on top of ASP.NET so it's not weird that Both Falco and Giraffe can be set up in the same way.

> what I want to mean is that Falco can use this setup as well.

Let's focus on this part for a little bit  

```
let webApp =
    choose [
        route "/ping"   >=> text "pong"
        route "/"       >=> htmlFile "/pages/index.html" ]
```

Enter fullscreen mode Exit fullscreen mode

routes in Giraffe are defined differently than Falco, while both are an array of functions Giraffe defines an HttpHandler like [this](https://giraffe.wiki/docs#httphandler)  

```
let handler =
    fun (next : HttpFunc) (ctx : HttpContext) ->
        task {return! text "" next ctx }
```

Enter fullscreen mode Exit fullscreen mode

now if you find confusing this symbol `>=>` don't worry too much about it, it just means that you can [compose](https://giraffe.wiki/docs#compose-) these functions, which can be a fancy word for a way to **chain HttpHandlers** e.g.  

```
let handler =
    route "/"
    >=> setHttpHeader "X-Foo" "Bar"
    >=> setStatusCode 200
    >=> setBodyFromString "Hello World"
```

Enter fullscreen mode Exit fullscreen mode

In the end a handler in Giraffe it's just a function and it has access to the HttpContext as well.

### [](https://dev.to/tunaxor/f-s-mean-1g2b#saturn-framework)Saturn Framework

Saturn is the most opinionated of all of these (except perhaps ASP.NET, but as you can see it can be used in al sorts of ways in any case) but it aims to improve developer experience and ergonomics while creating web servers in F#  

```
// mvc style controller
let userController = controller {
    index (fun ctx -> "Index handler" |> Controller.text ctx) //View list of users
    add (fun ctx -> "Add handler" |> Controller.text ctx) //Add a user
    create (fun ctx -> "Create handler" |> Controller.text ctx) //Create a user
    show (fun ctx id -> (sprintf "Show handler - %i" id) |> Controller.text ctx) //Show details of a user
    edit (fun ctx id -> (sprintf "Edit handler - %i" id) |> Controller.text ctx)  //Edit a user
    update (fun ctx id -> (sprintf "Update handler - %i" id) |> Controller.text ctx)  //Update a user
}
// function style routing
let appRouter = router {
    get "/" (htmlView Index.layout)
    get "/hello" (text "Hello world!")
    forward "/users" userController
}

let app = application {
    use_router appRouter
}

run app
```

Enter fullscreen mode Exit fullscreen mode

Saturn provides a DSL that is easy to read and it's self-explanatory, Saturn offers a functional MVC style while also allowing you to only use functions when needed, there are also other kinds of helpers that you can use to fully customize how your requests are served

I won't put samples of ASP.NET since they are quite big for the recommended approach and Microsoft can explain better than I on their docs website, but the gist is that ASP.NET powers all of the above so you're not missing anything from them

> MEAN - Mongo Express Angular Node
> 
> MERN - Mongo Express React Node

Unlike the javascript landscape, the F# space has settled on a few ways to do front end development the two main ways to do so are

-   [Feliz](https://dev.toand%20elmish%20alternatives/)
-   [Bolero](https://fsbolero.io/)

both will take your F# code into the browser, Feliz uses an `F# -> JS` approach thanks to the [Fable Compiler](https://fable.io/) while Bolero uses the power of WebAssembly to run natively in the browser.

## [](https://dev.to/tunaxor/f-s-mean-1g2b#feliz)Feliz

If you've done React before Feliz will have you at home  

```
module App

open Feliz

let counter = React.functionComponent(fun () ->
    let (count, setCount) = React.useState(0)
    Html.div [
        Html.button [
            prop.style [ style.marginRight 5 ]
            prop.onClick (fun _ -> setCount(count + 1))
            prop.text "Increment"
        ]

        Html.button [
            prop.style [ style.marginLeft 5 ]
            prop.onClick (fun _ -> setCount(count - 1))
            prop.text "Decrement"
        ]

        Html.h1 count
    ])

open Browser.Dom

ReactDOM.render(counter, document.getElementById "root")
```

Enter fullscreen mode Exit fullscreen mode

As you can see you can use Hooks, props, and render as you would in a normal react application, however, this will get improved once Fable3 lands out

> ![unknown tweet media content](https://res.cloudinary.com/practicaldev/image/fetch/s--u-2C71vY--/c_limit%2Cf_auto%2Cfl_progressive%2Cq_auto%2Cw_880/https://pbs.twimg.com/media/EnnsNiAW8AY_iup.png)
> 
> Playing with newest [#fsharp](https://twitter.com/hashtag/fsharp) 5, [@IonideProject](https://twitter.com/IonideProject), [@FableCompiler](https://twitter.com/FableCompiler) and [@zaid\_ajaj](https://twitter.com/zaid_ajaj)'s Feliz features. This looks like a really nice minimalistic React component 🥳
> 
> 21:55 PM - 24 Nov 2020
> 
>  [![Twitter reply action](https://res.cloudinary.com/practicaldev/image/fetch/s--fFnoeFxk--/c_limit%2Cf_auto%2Cfl_progressive%2Cq_auto%2Cw_880/https://dev.to/assets/twitter-reply-action-238fe0a37991706a6880ed13941c3efd6b371e4aefe288fe8e0db85250708bc4.svg)](https://twitter.com/intent/tweet?in_reply_to=1331355724866334722) [ ![Twitter retweet action](https://res.cloudinary.com/practicaldev/image/fetch/s--k6dcrOn8--/c_limit%2Cf_auto%2Cfl_progressive%2Cq_auto%2Cw_880/https://dev.to/assets/twitter-retweet-action-632c83532a4e7de573c5c08dbb090ee18b348b13e2793175fea914827bc42046.svg) ](https://twitter.com/intent/retweet?tweet_id=1331355724866334722) [![Twitter like action](https://res.cloudinary.com/practicaldev/image/fetch/s--SRQc9lOp--/c_limit%2Cf_auto%2Cfl_progressive%2Cq_auto%2Cw_880/https://dev.to/assets/twitter-like-action-1ea89f4b87c7d37465b0eb78d51fcb7fe6c03a089805d7ea014ba71365be5171.svg)](https://twitter.com/intent/like?tweet_id=1331355724866334722) 

## [](https://dev.to/tunaxor/f-s-mean-1g2b#bolero)Bolero

Bolero allows you to do [Elmish](https://elmish.github.io/) programming and any kind of component programming as well, it's quite similar to React  

```
let myElement name =
    div [] [
        h1 [] [text "My app"]
        p [] [textf "Hello %s and welcome to my app!" name]
    ]
```

Enter fullscreen mode Exit fullscreen mode

as in Feliz above, this is a slight different DSL that allows you to write your views, Bolero also allows you to program using HTML templates that provide hot reload (pretty common in javascript based tools) which is kind of hard to get when you go native  

```
<!-- hello.html -->
<div id="${Id}">Hello, ${Who}!</div>
```

Enter fullscreen mode Exit fullscreen mode

in these HTML templates, you basically define "holes" that can be filled with any information you want  

```
type Hello = Template<"hello.html">

let hello =
    Hello()
        .Id("hello")
        .Who("world")
        .Elt()
```

Enter fullscreen mode Exit fullscreen mode

these are type-checked at compile time as well, so you can be safe that you're not losing benefits at all.

Node is a nice way to do things, especially now that there's a LOT of javascript developers around the world these developers can use the full extent of their knowledge to create apps using javascript for every single part of their stack as we saw above, Node is the pillar of this but... is that true for .NET as well?

.NET became open source and cross-platform some years ago with that .NET really opened itself to compete in places where it wasn't present before (at least not in an official way \[Mono\]), like Linux, but that has changed over the years you can target every part of the stack as well with .NET and that means you can either use F# or C# as well.

[![.NET Availability](https://res.cloudinary.com/practicaldev/image/fetch/s--QDy4QUr1--/c_limit%2Cf_auto%2Cfl_progressive%2Cq_auto%2Cw_880/https://dev-to-uploads.s3.amazonaws.com/i/fvn1t7syvg13zcvi4ij9.png)](https://res.cloudinary.com/practicaldev/image/fetch/s--QDy4QUr1--/c_limit%2Cf_auto%2Cfl_progressive%2Cq_auto%2Cw_880/https://dev-to-uploads.s3.amazonaws.com/i/fvn1t7syvg13zcvi4ij9.png)

### [](https://dev.to/tunaxor/f-s-mean-1g2b#something-safeer)Something **_SAFE_**er

> Hey! well it's cool but I don't want to be looking for each individual piece, is there something that I can build that already exists and helps me figure out what's one way to do things?

Indeed there is! Enter the [SAFE Stack](https://safe-stack.github.io/) which will show you how you can have an equivalent of the MEAN stack in F# land.

-   Saturn
-   Azure
-   Fable
-   Elmish

Although some names are used in the acronym, feel SAFE that you are not locked into each of those, you can swap parts of it, for example instead of Saturn you can use Giraffe/Falco, you can choose AWS or Heroku instead as well granted that the default templates may not include those alternatives, but there's nothing that stops you from going your own way, you're not locked in that aspect. Check the SAFE website I'm pretty sure they can explain better in their docs than I what the SAFE stack is and what you can accomplish with it

F# is pretty safe and versatile I can almost guarantee that even if you don't use F# daily at work if you learn F# it will improve greatly the way you do software development my javascript has benefitted quite a lot from it, and I think (at least I'd love to think) that I can get simpler solutions after F# than what I used to have before.

In any case, please let me know if you have further doubts or comments below 😁 you can also reach me on Twitter.