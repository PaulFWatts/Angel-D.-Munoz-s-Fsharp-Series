In the last post, I gave an overview of how this project is setup and what is roughly trying to do.

An experimental repo about a Giraffe monolith you can either develop with docker or without it

Using docker-compose is the usual way to get both the database and the project running

you can just run `docker-compose up` and voila the database and the container will be built for you.

> **NOTE**: If you get red squiggly lines do a dotnet restore locally so ionide can pick up your dependencies

or if you're not using docker you will need to have a database running locally then just do as you would normally

## Debug

To debug just download the docker extension for vscode, be sure your container is running (`docker-compose up`) and press F5 you can hit breakpoints now

you can either do a local `dotnet publish -c Release` or use the dockerfile present in the root `docker build -t imagename:version` that will give you a fresh…

In this post I'll do a deeper dive into the F# specific parts of it, let's start with the File layout at the first glance, it might seem to have quite a lot of content and you may think everything is dispersed and it might be but, I'd like to invite you to use better tools to navigate your code, sometimes we want to take file layouts to organize things in our minds and that isn't really (or shouldn't be) the way software actually works, inside those files you might have different namespaces/modules that don't really accommodate to your file layout.

Once we get VSCode up and running with [Ionide](https://ionide.io/) VSCode extension we can see that there's not a lot of F# at all

[![File structure](https://res.cloudinary.com/practicaldev/image/fetch/s--mG50a38w--/c_limit%2Cf_auto%2Cfl_progressive%2Cq_auto%2Cw_880/https://dev-to-uploads.s3.amazonaws.com/i/my4252n4fdy36cou08et.png)](https://res.cloudinary.com/practicaldev/image/fetch/s--mG50a38w--/c_limit%2Cf_auto%2Cfl_progressive%2Cq_auto%2Cw_880/https://dev-to-uploads.s3.amazonaws.com/i/my4252n4fdy36cou08et.png)

> If you are unfamiliar with F# know that F# requires file ordering and scoping is defined from the top file to the bottom and in a similar way the same can be said of the project files

Let's start with our `Types.fs` file  

```
namespace Somnifero.Types

open MongoDB.Bson
open System
open MongoDB.Bson.Serialization.Attributes

type AuthResponse = { User: string }

type LoginPayload = { email: string; password: string }

type SignUpPayload =
    { email: string
      password: string
      name: string
      lastName: string
      invite: string }

// The main reason for this is that we store the password
// but we don't actually use it every time so we prevent 
// serialization issues within the mongo driver
[<BsonIgnoreExtraElements>]
type User =
    { _id: ObjectId
      name: string
      lastName: string
      email: string
      invite: string }

type PaginatedResult<'T> = { items: seq<'T>; count: int64 }
```

Enter fullscreen mode Exit fullscreen mode

One of the things I do most in F# it's to just write a `Types.fs` file that contains most of my type definitions that I can use in the whole project, I skipped some of those types in here since they are not relevant to this post, as you will see.

These types simply represent the data I'm working on within the application, I didn't use any specific approach like DDD or anything like that, I just know these are the shapes of the data I'd like to work with so far.

Our next file is `Database.fs`  

```
namespace Somnifero

open FSharp.Control.Tasks
open System
open MongoDB.Driver
open MongoDB.Bson
open Somnifero.Types
open System.Threading.Tasks
open System.Text.RegularExpressions
open System.Security.Cryptography

[<RequireQualifiedAccess>]
module Database = ...

[<RequireQualifiedAccess>]
module Users = ...

[<RequireQualifiedAccess>]
module Rooms = ...
```

Enter fullscreen mode Exit fullscreen mode

Remember that file structure in disk does not really represent our actual program? the types in `Types.fs` file are under the `Somnifero.Types` namespace and this file is under the `Somnifero` namespace it's not either good or bad design (maybe?) it's just that don't feel like you must constrain yourself with how things look in the disk and how things actually work in the program.

> This file is more complex, but I'll keep it simple since those other topics will be covered in other posts.

There are three modules here

-   Database
-   Users
-   Rooms

the last two modules depend on the first and as far as I know, my Database module could be private since I don't really need to use it anywhere else.

> I may say that F# helped me here to ensure my program had some logic to it I can only use what already exists which can be frustrating if you come from other languages.

The next step is `ViewModels.fs` this is another simple file that also defines types and you might wonder

> What's the difference between this file and the types file?

and I'd answer: _None really_

the main reason this file is in here, it's the usage these types are going to be used to define the kind of data I will be able to use on my `Razor Pages` since razor pages use C# inside them I might want to define mutable types and I wouldn't want to mix them or make others believe they share the same purpose within the project itself, that being said, I could just have added a comment over them inside the `Types.fs` file and continued with my day. Also, these types are never going to go inside my database, so that's why they are below the `Database` module.  

```
namespace Somnifero.ViewModels

open System.Collections.Generic


type HeaderData =
    { routeGroups: seq<IDictionary<string, string>> }

type FooterData =
    { routeGroups: seq<IDictionary<string, string>>
      extraData: Option<IDictionary<string, string>> }
```

Enter fullscreen mode Exit fullscreen mode

The next file is named `Hubs.fs` this file includes SignalR hubs for realtime communication (i.e. Websockets) between my server and my clients I'll skip the contents for that file in this post.

Our Next file is `Handlers.fs` this one is the meat and bones of our website it includes all the functions that handle HTTP requests to our server  

```
namespace Somnifero.Handlers

open System
open System.Text.Json
open System.Text.Json.Serialization
open System.Security.Claims

open FSharp.Control.Tasks

open Microsoft.AspNetCore.Authentication
open Microsoft.AspNetCore.Authentication.Cookies
open Microsoft.AspNetCore.Http

open Microsoft.Extensions.Logging

open Giraffe
open Giraffe.Razor.HttpHandlers

open BCrypt.Net

open Somnifero
open Somnifero.Types
open Somnifero.ViewModels


module Public = ...

module Auth = ...

module Api = ...

module Portal = ...
```

Enter fullscreen mode Exit fullscreen mode

It contains three modules as I tried to define how these handlers are used

-   Public  
    This module contains any function that handles public HTTP endpoint it doesn't matter if it returns JSON or HTML.
    
-   Auth  
    This one handles any endpoint that relates to security things, I would put login, signup, password resets, or similar things.
    
-   Api  
    This one handles all of the endpoints I'll be using as if it was a Restful API most likely anything here will return JSON.
    
-   Portal  
    This module includes anything related to the "/portal/" section of my website
    

As you can see there are no clear guidelines here to structure your Websites or API's I just followed this structure because that's what I'm used to, and I encourage you to either study a pattern to improve over this or to simply use what you feel is best just document it for your teammates or even yourself in the future

The last file on the list is `Program.fs` this is often a complex file for me because it involves ASP.NET configuration, and configuration is something puzzling for me. Let's get the code explanation running

there are four values in this file namely

-   webApp
-   configureApp
-   configureServices
-   main

The `webApp` is basically our router where we define which routes are we going to capture and what will we be answering with (the handlers from the `Handlers.fs` file)

`configureApp` is where we add middleware typically all the stuff that is like `app.Use*` like `UseRouting()`, `UseStaticFiles`, `UseGiraffe(webApp)` and so on.

`configureServices` is where we add different services to the built-in DI of ASP.NET (Yes, you can use DI services in Giraffe web servers)

`main` it's the equivalent to your `public static int main` it serves as the entry point of your application.

Before I finish with this post, let's follow the code for the next 3 requests

-   `"/"`
-   `"/portal/home"`
-   `"/auth/login"`

the router begins with the function `choose []` which basically takes a list of HTTP handlers  
the first route is

`GET >=> route "/" >=> Public.Index`

I read it as handle the `GET` request to the `"/" endpoint` with the `Public.Index` function  
the index function it's quite simple  

```
    let Index =
        fun (next: HttpFunc) (ctx: HttpContext) ->
            task {
                let data = ...

                if ctx.User.Identity.IsAuthenticated
                then return! redirectTo false "/portal/home" next ctx
                else return! razorHtmlView "Index" None data None next ctx
            }
```

Enter fullscreen mode Exit fullscreen mode

its value is a simple async function that takes a `next` function and an HTTP Context the `HttpContext` is the ASP.NET HttpContext so you can find anything you'd expect from ASP.NET context there, the `next` function is basically the next handler on the request's life, for example, a token validation middleware or any other kind of request processing function

We check that the user is authenticated and return a redirect or an HTML file (which I covered in the last post)

This one is a bit more complicated since it uses a subRoute, the subRoute is a way to handle segments of URLS that are part of a hierarchy  

```
subRoute 
    "/portal"
    (requiresAuthentication (challenge CookieAuthenticationDefaults.AuthenticationScheme)
    >=> choose [ GET >=> route "/home" >=> Portal.Index ])
```

Enter fullscreen mode Exit fullscreen mode

`requiresAuthentication` is a function that comes as part of Giraffe where you specify the kind of authentication you're requesting (it can be JWT in case you're using tokens), in this case, Cookie authentication. then we use again `choose` to **_choose_** between the two available routes, we handle a GET request with the `Portal.Index` handler

this handler is quite simple actually  

```
let Index: HttpHandler =
        fun (next: HttpFunc) (ctx: HttpContext) -> task { return! razorHtmlView "Portal/Index" None None None next ctx }
```

Enter fullscreen mode Exit fullscreen mode

it's almost a one-liner telling Giraffe that we want to render an HTML file and we're not passing any information to the rendering engine.

this time we'll be handling a `POST` request to the `/auth/login` endpoint **but** before we get to the login function, we'll check for an AntiForgeryToken if it's part of the request then we'll continue with the request, otherwise we'll use the InvalidCSRFToken handler  

```
POST >=> 
    (choose 
        [ route "/auth/login"
               >=> 
               validateAntiforgeryToken Public.InvalidCSRFToken
               >=> Auth.Login 
        ]
    )
```

Enter fullscreen mode Exit fullscreen mode

The Login function as you would guess, checks the user credentials that were provided in the POST request  

```
    let Login: HttpHandler =
        fun (next: HttpFunc) (ctx: HttpContext) ->
            task {
                // pull the credentials from the json body
                let! loginpayload = JsonSerializer.DeserializeAsync<LoginPayload>(ctx.Request.Body)
                let! queryResult = Users.TryFindUserByEmail loginpayload.email true
                // handle the different cases there might be ahead
                // like checking credentials or if the user even exists and sign in
                let response = ...

                return! response
            }
```

Enter fullscreen mode Exit fullscreen mode

And that's it I think that's most of what you need to make sense of the project.

As always feel free to share this post and leave a comment below. you can ping me on Twitter as well 😁