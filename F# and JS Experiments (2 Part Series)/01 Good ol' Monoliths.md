In an era where everything is a SPA and everything uses Webpack, it seems that you should only be doing web development as a SPA, cramming a ton of javascript on your website. Monoliths or also called MPA's (Multi-Page Applications) for some reason are almost always ruled out. I'm not here to convince you but, I think that's not always the best approach and you shouldn't rule out things for the sake of ruling them out.

> This will be a series where I expose some of my experiments with F#

Today I'll be telling you about a website I made with [Giraffe](https://giraffe.wiki/), [Razor Pages](https://github.com/giraffe-fsharp/Giraffe.Razor), and ES modules javascript. No webpack, no bundlers, no weird setups, just Server Side Rendered HTML and a few lines of javascript sprinkled on top of that

An experimental repo about a Giraffe monolith you can either develop with docker or without it

Using docker-compose is the usual way to get both the database and the project running

you can just run `docker-compose up` and voila the database and the container will be built for you.

> **NOTE**: If you get red squiggly lines do a dotnet restore locally so ionide can pick up your dependencies

or if you're not using docker you will need to have a database running locally then just do as you would normally

## Debug

To debug just download the docker extension for vscode, be sure your container is running (`docker-compose up`) and press F5 you can hit breakpoints now

you can either do a local `dotnet publish -c Release` or use the dockerfile present in the root `docker build -t imagename:version` that will give you a fresh…

> the code I'll be showing is under the webrtcchat branch

That repo contains a few experiments I'll be posting in the following weeks I'll name some.

-   Multi-Page Application (this post)
-   ES Modules Scripts for SSR'd apps (this post)
-   Realtime Communication with Signalr
-   WebRTC Video Broadcasting
-   docker-compose setup
-   MongoDB as a database for an F# backend

the file structure is the following one  
[![Project Structure](https://res.cloudinary.com/practicaldev/image/fetch/s--qOamoZrr--/c_limit%2Cf_auto%2Cfl_progressive%2Cq_auto%2Cw_880/https://dev-to-uploads.s3.amazonaws.com/i/ppiu10qxuvtc9p4bcfnq.png)](https://res.cloudinary.com/practicaldev/image/fetch/s--qOamoZrr--/c_limit%2Cf_auto%2Cfl_progressive%2Cq_auto%2Cw_880/https://dev-to-uploads.s3.amazonaws.com/i/ppiu10qxuvtc9p4bcfnq.png)  
it has a bunch of docker files and files and folders that might make it a little bit messy but it isn't that messy at all

**WebRoot** is basically where the static files for the server reside (any images, javascript, css, etc)

**Views** is where the HTML that uses Razor Pages resides

The rest is either a docker file or F# code, the F# solution is quite simple actually

[![F# Code Actual Structure](https://res.cloudinary.com/practicaldev/image/fetch/s--CBA-bTav--/c_limit%2Cf_auto%2Cfl_progressive%2Cq_auto%2Cw_880/https://dev-to-uploads.s3.amazonaws.com/i/3mso4rh5vdv5xf0ymjqt.png)](https://res.cloudinary.com/practicaldev/image/fetch/s--CBA-bTav--/c_limit%2Cf_auto%2Cfl_progressive%2Cq_auto%2Cw_880/https://dev-to-uploads.s3.amazonaws.com/i/3mso4rh5vdv5xf0ymjqt.png)

One thing to remember is that F# requires File ordering, everything related to F# code cascades from top to bottom, so in this case `Types.fs` has the most accessible code in the whole application where `Program.fs` code is only accessible within itself. Let's review what kind of routes we have here  

```
let webApp =
    choose [ GET >=> route "/" >=> Public.Index
             route "/auth/signout"
             >=> signOut CookieAuthenticationDefaults.AuthenticationScheme
             >=> redirectTo false "/"
             GET >=> route "/broadcast" >=> Public.Broadcast
             GET >=> route "/watch-broadcast" >=> Public.Watch
             POST
             >=> (choose [ route "/auth/login"
                           >=> validateAntiforgeryToken Public.InvalidCSRFToken
                           >=> Auth.Login
                           route "/auth/exists" >=> Auth.CheckExists
                           route "/auth/signup"
                           >=> validateAntiforgeryToken Public.InvalidCSRFToken
                           >=> Auth.Signup ])
             subRoute
                 "/portal"
                 (requiresAuthentication (challenge CookieAuthenticationDefaults.AuthenticationScheme)
                  >=> choose [ GET >=> route "/home" >=> Portal.Index
                               GET >=> route "/me" >=> Portal.Me ])

             subRoute
                 "/api"
                 (requiresAuthentication (challenge CookieAuthenticationDefaults.AuthenticationScheme)
                  >=> choose [ GET >=> route "/me" >=> Api.Me ])

             setStatusCode 404 >=> text "Not Found" ]
```

Enter fullscreen mode Exit fullscreen mode

> Note: the **_\>=>_** combinator means "compose". You can read more [here](https://giraffe.wiki/docs#compose-)

This means that there are basically 10 routes that range from a simple request Html rendering calls to RestAPI endpoints and so on as you can see in that router, we have Cookie and AntiForgery checks as security measures.  
Let's start with our Index route  

```
module Public =
    // this is a small helper to create a tuple with a key and a boxed value
    let inline (+>) (a: string) (b: 'T): string * obj = a, box b

    let Index =

        fun (next: HttpFunc) (ctx: HttpContext) ->
            task {
                let data =
                    dict<string, obj>
                        [ "Title" +> "Welcome"
                          "HeaderData" +> { routeGroups = Seq.empty }
                          "FooterData"
                          +> { routeGroups = Seq.empty
                               extraData = None } ]
                    |> Some

                if ctx.User.Identity.IsAuthenticated
                then return! redirectTo false "/portal/home" next ctx
                else return! razorHtmlView "Index" None data None next ctx
            }
```

Enter fullscreen mode Exit fullscreen mode

[![Index Home](https://res.cloudinary.com/practicaldev/image/fetch/s--dlMzdVP4--/c_limit%2Cf_auto%2Cfl_progressive%2Cq_auto%2Cw_880/https://dev-to-uploads.s3.amazonaws.com/i/gy7tjaupbvom4ki18bb7.png)](https://res.cloudinary.com/practicaldev/image/fetch/s--dlMzdVP4--/c_limit%2Cf_auto%2Cfl_progressive%2Cq_auto%2Cw_880/https://dev-to-uploads.s3.amazonaws.com/i/gy7tjaupbvom4ki18bb7.png)

The index route is probably the simplest one, we just check if the user is authenticated, if the user is authenticated we do a redirect (since the index is our login page as well) and if not we return a `razorHtmlView` which will try to look for the file `Index.cshtml` we don't pass a model, but we pass some `ViewData`.

If you have used any kind of templating engine before, like jinja, ejs, jade (formerly pug), handlebars the following contents won't be much different, the Index file uses [Giraffe.Razor](https://github.com/giraffe-fsharp/Giraffe.Razor) as the view engine, so if you're an ASP.NET developer that might want to dive a bit into F# this can be an opportunity for you 😉.  

```
@using Somnifero.ViewModels;

@{
    Layout = "_Layout";
    var headerData = ViewData["HeaderData"] as HeaderData;
    var footerData = ViewData["FooterData"] as FooterData;
}
@section Header {
  @await Html.PartialAsync("_Header", headerData)
}
<main class="som-main">...

@section Scripts {
<script src="/js/index.js" type="module"></script>
}

@section Footer {
  @await Html.PartialAsync("_Footer", footerData)
}
```

Enter fullscreen mode Exit fullscreen mode

> you can see razor files use C# hence why I mentioned if you are a C# looking into F# territory you can go step by step instead of replacing C# from the get-go

I've omitted most of the HTML contents since they are the HTML you know and love there's nothing ultra strange about those. In this case, we just declare which file is our layout and pull the header and footer data from the `ViewData` dictionary and render those partials inside the `Header` and `Footer` accordingly. in our scripts section, we have one of our first niceties of the modern era of javascript an "ES module", even if it took us five years or more to get there and perhaps with some caveats since there might be users that yet don't have an advanced enough browser we can use today ES modules as part of our javascript files in the browser without transpiling them.  

```
// YES FINALLY just pull your library and import stuff from it
import { enhance } from 'https://unpkg.com/aurelia-script@1.5.2/dist/aurelia.esm.min.js'

class LoginFormViewModel {}

class SignupViewModel {}

enhance({
  host: document.querySelector('.loginsection'),
  root: LoginFormViewModel
});

enhance({
  host: document.querySelector('.signupsection'),
  root: SignupViewModel
});

function getCSRFTokenFromForm(form) {}
```

Enter fullscreen mode Exit fullscreen mode

Our javascript file is quite small and does what you would expect, make HTML dynamic 😁.

[![Register](https://res.cloudinary.com/practicaldev/image/fetch/s--03_2XYXU--/c_limit%2Cf_auto%2Cfl_progressive%2Cq_auto%2Cw_880/https://dev-to-uploads.s3.amazonaws.com/i/4ojo8ve8jexg6hlfww08.png)](https://res.cloudinary.com/practicaldev/image/fetch/s--03_2XYXU--/c_limit%2Cf_auto%2Cfl_progressive%2Cq_auto%2Cw_880/https://dev-to-uploads.s3.amazonaws.com/i/4ojo8ve8jexg6hlfww08.png)

Here we have two classes that are used in conjunction with the aurelia-script library. Basically, we just render our HTML right from the server, add a bit of javascript to it, and continue our lives.

> I'll be adding another post for aurelia-script in the future

This page is an example of how you can just pull your HTML from the server and sprinkle some functionality over it with an ES module.

what about if you want something not so static? let's say a chat and you just want to render the initial page then you want to let javascript get full control of it?  
let's go over the Portal page  

```
module Portal =

    let Index: HttpHandler =
        fun (next: HttpFunc) (ctx: HttpContext) -> task { return! razorHtmlView "Portal/Index" None None None next ctx }
```

Enter fullscreen mode Exit fullscreen mode

Boom, blazing fast! just render my file already!  

```

@{
    Layout = "../_Layout";
    ViewData["Title"] = "Portal";
    /// these will be empty since we didn't pass any information
    var headerData = ViewData["HeaderData"] as HeaderData;
    var footerData = ViewData["FooterData"] as FooterData;
}
@section Header {
  @await Html.PartialAsync("../Shared/_Header", headerData)
}
<!-- I don't want to have a lot of HTML, just render my stuff -->
<main class="som-main" name="portal"></main>

@section Scripts {
<script src="/js/portal/index.js" type="module"></script>
}

@section Footer {
  @await Html.PartialAsync("../Shared/_Footer", footerData)
}
```

Enter fullscreen mode Exit fullscreen mode

[![Simply render stuff](https://res.cloudinary.com/practicaldev/image/fetch/s--F8_GL8eW--/c_limit%2Cf_auto%2Cfl_progressive%2Cq_auto%2Cw_880/https://dev-to-uploads.s3.amazonaws.com/i/ih02w8l735z1gn349oqe.png)](https://res.cloudinary.com/practicaldev/image/fetch/s--F8_GL8eW--/c_limit%2Cf_auto%2Cfl_progressive%2Cq_auto%2Cw_880/https://dev-to-uploads.s3.amazonaws.com/i/ih02w8l735z1gn349oqe.png)

Now instead of just pulling the enhance to lift a small part of the HTML, we'll go full Aurelia and we will be pulling messages with Signalr and rendering a small chat PoC  

```
import { Aurelia, PLATFORM } from 'https://unpkg.com/aurelia-script@1.5.2/dist/aurelia_router.esm.min.js'
const aurelia = new Aurelia();
aurelia
    .use
    .standardConfiguration()
    .developmentLogging();

aurelia.start()
    .then(aur => aur.setRoot(PLATFORM.moduleName("/js/portal/app.js"), document.querySelector("[name=portal]")))
    .catch(error => {
        console.error(`Something went wrong starting the portal page:\n"${error.message}"`);
        UIkit.notification({
            message: 'There was an error starting this page, please reload to avoid data loss',
            status: 'danger',
            pos: 'top-right',
            timeout: 5000
        });
    });
```

Enter fullscreen mode Exit fullscreen mode

[![Chat](https://res.cloudinary.com/practicaldev/image/fetch/s--qket4BRG--/c_limit%2Cf_auto%2Cfl_progressive%2Cq_auto%2Cw_880/https://dev-to-uploads.s3.amazonaws.com/i/q82nhr6rvg546lr2khmz.png)](https://res.cloudinary.com/practicaldev/image/fetch/s--qket4BRG--/c_limit%2Cf_auto%2Cfl_progressive%2Cq_auto%2Cw_880/https://dev-to-uploads.s3.amazonaws.com/i/q82nhr6rvg546lr2khmz.png)

If you are interested you can check the [following files](https://github.com/AngelMunoz/Somnifero/tree/webrtcchat/WebRoot/js/portal)  
[![WebRoot/portal](https://res.cloudinary.com/practicaldev/image/fetch/s--Xqgq5eaU--/c_limit%2Cf_auto%2Cfl_progressive%2Cq_auto%2Cw_880/https://dev-to-uploads.s3.amazonaws.com/i/g3f0blzn75dkjnc6omuw.png)](https://res.cloudinary.com/practicaldev/image/fetch/s--Xqgq5eaU--/c_limit%2Cf_auto%2Cfl_progressive%2Cq_auto%2Cw_880/https://dev-to-uploads.s3.amazonaws.com/i/g3f0blzn75dkjnc6omuw.png)

I know there is a LOT going on in that project I'll try to expand on each part of the stack in further posts.

I hope this post kind of sheds light on how using F# is super simple (if you see the route handlers they are like... 50-60 lines big?) and that not everything needs to be a SPA and not everything needs a 2MB bundle, I used aurelia, but you can indeed use react, Vue whatever you like on your frontend stack.

If you liked it feel free to share it, if you have those C# friends that want... but don't want to fully commit to F# let them know they can still use razor pages.

As always you can comment below or ping me on Twitter 😁