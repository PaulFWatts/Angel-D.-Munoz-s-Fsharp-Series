![Cover image for Server Sent Events with Saturn and FSharp](https://res.cloudinary.com/practicaldev/image/fetch/s--H9DNP8hR--/c_imagga_scale,f_auto,fl_progressive,h_420,q_auto,w_1000/https://dev-to-uploads.s3.amazonaws.com/uploads/articles/ns5lg4r0oiiwq6x6owg5.png)

[![Angel D. Munoz](https://res.cloudinary.com/practicaldev/image/fetch/s--FXr-t6j_--/c_fill,f_auto,fl_progressive,h_50,q_auto,w_50/https://dev-to-uploads.s3.amazonaws.com/uploads/user/profile_image/35800/b392acdc-d3fc-4ab0-a1cc-4d200e1a4582.jpg)](https://dev.to/tunaxor)

For this week's F# content I've have some server side stuff. I spend most of my days writing frontend code either for a living or for entretainment but sometimes I go to the server or CLI stuff as well.

> The full code for this sample is here: [https://github.com/AngelMunoz/SaturnSSE](https://github.com/AngelMunoz/SaturnSSE)

I've been working in [perla](https://github.com/AngelMunoz/Perla) which is a cross-platform executable frontend dev-server/build-tool which is not tied to Nodejs or .NET meaning that you don't need to have .NET installed and neither Nodejs, at the same time it doesn't use npm or other things to handle dependencies it does so by leveraging [skypack](https://dev.tojspm%2C%20unpkg%2C%20jsdlivr%20as%20well/) and [import maps](https://github.com/WICG/import-maps) to let you use npm dependencies but from a CDN rather than locally.

> If you use .NET well it is provided as a .NET tool so you will be able to use it from CI as well and your local tools.

One of the things that dev servers do is to auto-reload when there's a change on the files you're editing on your frontend project. Since _perla_ runs on .NET I needed to figure out how to notify the client when a file has changed.

## [](https://dev.to/tunaxor/server-sent-events-with-saturn-and-fsharp-m6b#options)Options

-   Web sockets
-   Signalr (the same but not the same of web sockets)
-   Server Sent Events
-   Long polling (auto-ruled out)

### [](https://dev.to/tunaxor/server-sent-events-with-saturn-and-fsharp-m6b#web-sockets)Web sockets

This is the most common approach here due to the real-time/bi-directional nature of them, but to be honest I think they require a relatively lot of setup just or a client to be notified when it has to reload.

### [](https://dev.to/tunaxor/server-sent-events-with-saturn-and-fsharp-m6b#signalr)Signalr

Well... it's MS's library for real time stuff which suffers the same thing of web sockets for this use case Plus I'd need a library in the frontend to make it work, Which is not ideal I just want to hook up something listen and reload. Nothing more nothing less.

### [](https://dev.to/tunaxor/server-sent-events-with-saturn-and-fsharp-m6b#long-polling)Long polling

I don't want to set a loop to keep polling the server, I want the server to tell me when to do it.

### [](https://dev.to/tunaxor/server-sent-events-with-saturn-and-fsharp-m6b#server-sent-events)Server Sent Events

[SSE](https://developer.mozilla.org/en-US/docs/Web/API/Server-sent_events/Using_server-sent_events) are an ideal solution here because rather than a bi-directional approach this connection is unidirectional, from server -> client and the server is the one that let's the client know when something has changed each request has it's own life cycle so I don't need to worry about which client did what and so on I just let them know something happened and that's it.

In the case of SSE, it goes like this

-   Client creates a new EventSource the EventSource automatically does a GET request to the provided endpoint
-   Client adds event listeners for any particular event the client is interested in
-   Server receives request
    -   Server sets `cache-control: no-cache` and `content-type: text/event-stream`
    -   Server keeps the connection open until required (a while true works here)
-   When something hapens the server writes a new event to the response's body and flushes the content.
-   Client's listeners get invoked depending on the event sent from the server

In the javascript side it looks like this:  

```
const source = new EventSource("/sse");

source.addEventListener("open", function (event) {
  console.log("Connected");
});

// Let's skip ahead and set already our "reload" event
source.addEventListener("reload", function (event) {
  console.log("Reloading, file changed: ", event.data);
});
// Listen to any message sent not tied to a particular event
source.addEventListener("message", function (event) {
  console.log(event);
  console.log(event.data);
});

source.addEventListener("error", function (err) {
  console.error(err);
});
```

Enter fullscreen mode Exit fullscreen mode

> If you were to have multiple _SSE_ endpoints you would need to create a different source for each
> 
> -   `new EventSource("/sales/notifications")`
> -   `new EventSource("/games/some-id/scores")`

Let's see how does that look from F# thanfully we can do that in a single file  

```
dotnet new console -lang F# -o SSESample
dotnet add package Saturn
code SSESample # or rider or visual studio, your choice
```

Enter fullscreen mode Exit fullscreen mode

```
open System
open System.IO
open FSharp.Control.Tasks

open Giraffe
open Saturn

open Saturn.Endpoint.Router
open Microsoft.AspNetCore.Http

let sse next (ctx: HttpContext) =
    task {
        let res = ctx.Response
        ctx.SetStatusCode 200
        ctx.SetHttpHeader("Content-Type", "text/event-stream")
        ctx.SetHttpHeader("Cache-Control", "no-cache")

        while true do
            // VERY IMPORTANT
            // MESSAGES MUST START WITH `data:`
            // AND MUST END WITH `\n\n`
            // otherwise the client will keep buffering and no data will be consumed
            do! res.WriteAsync $"data:Hello, world!\n\n"
            do! res.Body.FlushAsync()
            // not required but prevent your CPU from running crazy :P
            do! Async.Sleep(TimeSpan.FromSeconds 2.5)

        // This won't be hit because we're expecting
        // the browser to break the connection
        // although you can indeed break from the loop above and get here
        return! text "" next ctx
    }

[<EntryPoint>]
let main args =
    let app =
        application {
            use_endpoint_router (router { get "/sse" sse })
            // serve an index file with the JS code from above
            use_static "wwwroot"
        }

    run app
    0
```

Enter fullscreen mode Exit fullscreen mode

That's all you need to suport a simple SSE Endpoint, if you're following .NET6 minimal endpoints or asp.net middleware that SSE endpoint will be relatively familiar to you we'e just modifying the HTTP Context and flushing out the context  

```
dotnet run
```

Enter fullscreen mode Exit fullscreen mode

open the networking tab in your browser's dev tools and reload the page you'll see the `/sse` request being made and each 2.5 seconds you'll see a message in the console with the "Hello, world!" string

Things that you can send in the event

-   Event `event:event-name`, e.g. `event:reload`
-   Data `data:any string content that doesn't end with \n\n`
-   Id `id:any content`

let's see for example this  

```
id:ABC-DFG-HIJ-LMN
event:signup
data:{ "id": 10500, "name": "Peter Parker", "email":"im-not@spiderman.com" }

```

Enter fullscreen mode Exit fullscreen mode

For your JS code to interpret that you'd need a listener like this  

```
source.addEventListener("signup", function (event) {
  const user = JSON.parse(event.data);
  const eventId = event.id;
  console.log(
    `Event: ${eventId}, User signed up: ${user.name} - ${user.email}`
  );
});
```

Enter fullscreen mode Exit fullscreen mode

It's important to know that the data is always a string so be sure to serialize it correctly from the backend so you can reliably use it from the client.

That's it! Quite simple right?

See you next week!

Yup, good bye...

Ahh I see you're here for the bonus, got it let's implement that reload on change event

## [](https://dev.to/tunaxor/server-sent-events-with-saturn-and-fsharp-m6b#bonus)Bonus

Let's add a file watcher and make this thing let the client know something changed in the backend  

```
dotnet new console -lang F# -o SSEFileWatcher
dotnet add package Saturn
dotnet add package FSharp.Control.Reactive
code SSESample # or rider or visual studio, your choice
```

Enter fullscreen mode Exit fullscreen mode

To avoid dealing with multiple subscriptions and weird disposal stuff we will work with `FSharp.Control.Reactive` which allows us to create obsevables from .NET events we'll have to combine a few of those into one stream because a file can change in many ways and we're interested in knowing it changed, not necessarily in what way. let's re-check our F# program with the new extra stuff and see how it went  

```

open System
open System.IO
open System.Text.Json

open FSharp.Control.Tasks
open FSharp.Control.Reactive

open Microsoft.AspNetCore.Http

open Giraffe
open Saturn
open Saturn.Endpoint.Router

// a simple inerface
type INotifierService =
    inherit IDisposable
    abstract OnFileChanged : IObservable<string>


// this function will build an INotifier service
// if you're using DI you can easily change this into
// a service factory and use services.AddSingleton<INotifierService>(getNotifier)
// you would need to read the path from the configuration
// and take IService collection as a paremeter though
let getNotifier (path: string) : INotifierService =
    let fsw = new FileSystemWatcher(path)

    // Filter events by filename and size
    fsw.NotifyFilter <- NotifyFilters.FileName ||| NotifyFilters.Size
    fsw.EnableRaisingEvents <- true

    // the next part is creating observables from .NET events and just getting
    // the Name of the file that changed.

    let changed =
        fsw.Changed
        |> Observable.map (fun args -> args.Name)

    let deleted =
        fsw.Deleted
        |> Observable.map (fun args -> args.Name)

    let renamed =
        fsw.Renamed
        |> Observable.map (fun args -> args.Name)

    let created =
        fsw.Created
        |> Observable.map (fun args -> args.Name)

    let obs =
        // merge all of the file events into a single stream of events
        Observable.mergeSeq [ changed
                              deleted
                              renamed
                              created ]

    // return a  INotifierService
    { new INotifierService with
        override _.Dispose() : unit = fsw.Dispose()
        override _.OnFileChanged: IObservable<string> = obs }

let sse next (ctx: HttpContext) =
    task {
        let res = ctx.Response
        ctx.SetStatusCode 200
        ctx.SetHttpHeader("Content-Type", "text/event-stream")
        ctx.SetHttpHeader("Cache-Control", "no-cache")
        // get our notifier
        let notifier = getNotifier @"C:\Users\scyth\Desktop"

        let onFileChanged =
            notifier.OnFileChanged
            // subscribe to events
            |> Observable.subscribe
                (fun filename ->
                    // Sync operations in the response are disallowed by default
                    // so we will wrap this in a task and run it immediately
                    task {
                        let data =
                            JsonSerializer.Serialize({| filename = filename |})
                        // write our event to the body
                        do! res.WriteAsync $"event:reload\ndata:{data}\n\n"
                        // Flush the contents so the client can read this information
                        do! res.Body.FlushAsync()
                    }
                    |> Async.AwaitTask
                    |> Async.StartImmediate)
        // Write an initial
        do! res.WriteAsync($"id:{ctx.Connection.Id}\nevent:start\ndata:{DateTime.Now}\n\n")
        do! res.Body.FlushAsync()

        // release resources when the client disconnects
        ctx.RequestAborted.Register
            (fun _ ->
                notifier.Dispose()
                onFileChanged.Dispose())
        |> ignore

        // keep the connection alive
        while true do
            do! Async.Sleep(TimeSpan.FromSeconds 1.)

        return! text "" next ctx
    }

[<EntryPoint>]
let main args =
    let app =
        application {
            use_endpoint_router (router { get "/sse" sse })
            // serve an index.html file with the first JS block above
            use_static "wwwroot"
        }

    run app
    0
```

Enter fullscreen mode Exit fullscreen mode

Here's an older version of this approach which enabled sync operations on that particular request but I think the aproach above is a better option.

That's it! For sure this time, so no fable today but still we managed to get some cool F# content for this week

I'll see you around on the next one