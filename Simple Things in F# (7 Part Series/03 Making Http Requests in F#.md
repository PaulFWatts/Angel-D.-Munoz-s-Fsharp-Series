---
aliases: [dotnet]
tags: [fsharp]
---
-----
[Making Http Requests in F# - DEV Community 👩‍💻👨‍💻](https://dev.to/tunaxor/making-http-requests-in-f-1n0b)

This is the third post in Simple things in F#. Today we will talk about doing HTTP Requests

> **_DISCLAIMER_**: I'll be using [Ply](https://github.com/crowded/ply) in most of the samples due to it's efficient task CE and the easy interoperability between C# tasks as well as F#'s async. Http operations are by nature `async` operations, that means using `async/await` in C#/JavaScript or using `async {}` in F# which can be a little annoying when you need to append `|> Async.AwaitTask` to every function/method that returns a Task.

When you come to the F# ecosystem you will find that there is a great amount of F# specific libraries meaning that the library was designed to be used from F# but there's also a even bigger amount of libraries that use C# as the code base.

What do they have in common?

-   They are .NET libraries
-   You can use them from any of the .NET languages (C#, F#, VB)

Sometimes this means that the library is not _idiomatic_ for the language you're using and there may surface interop issues between languages but don't let that stop you from trying libraries here and there and you're not wrong in trying to consume a C# library from F# or in the reverse order.

Why do I mention this? because today we'll see three ways to do Http Requests with different libraries, we'll first explore the BCL's (Base Class Library) [System.Net.Http](https://docs.microsoft.com/en-us/dotnet/api/system.net.http?view=net-5.0) then we'll proceed to use [Flurl](https://flurl.dev/) and finally we'll check [FsHttp](https://github.com/ronaldschlenker/FsHttp), and depending on your taste or needs you may want to use one or the other.

## [](https://dev.to/tunaxor/doing-some-io-in-f-4agg#systemnethttp)System.Net.Http

This is part of the BCL, so it's very likely that you'll see lot of code out there using [HttpClient](https://docs.microsoft.com/en-us/dotnet/api/system.net.http.httpclient?view=net-5.0). this `HttpClient` provides some constructs that allow you to create Http Requests in any shape or form, however since this is part of the BCL it might be clunky on some aspects but it is very useful to build either libraries or services on top of it.

Let's begin by downloading a web page, kindly notice that the `HttpClient` class implements the `IDisposable` interface meaning that this class can release any resources it's using once we're done with it, a common way to do this is by writing the `use` keyword in F# or `using` in C#.  

```
#r "nuget: Ply"

// the following open statement makes the `task {}` CE available
open FSharp.Control.Tasks
open System.Net.Http
open System.IO

task {
    /// note the ***use*** instead of ***let***
    use client = new HttpClient()
    let! response = 
        client.GetStringAsync("https://dev.to/tunaxor/doing-some-io-in-f-4agg")
    do! File.WriteAllTextAsync("./response.html", response)
    // after the client goes out of scope
    // it will get disposed automatically thanks to the ***use*** keyword
}
|> Async.AwaitTask
// we run synchronously
// to allow the fsi to finish the pending tasks
|> Async.RunSynchronously
```

Enter fullscreen mode Exit fullscreen mode

> To Run this, copy this content into a file named `script.fsx` (or whatever name you prefer) and type:
> 
> -   `dotnet fsi script.fsx`

Keep in mind that by getting the contents as strings, you are putting that string in memory which might not be efficient if the website/content you're requesting is quite big and also if the content is a binary file that can't be represented with strings like a PDF, an Excel File or similar it will just corrupt the file. The way you can do this is by using the streams themselves  

```
#r "nuget: Ply"

// the following open statement makes the `task {}` CE available
open FSharp.Control.Tasks
open System.Net.Http
open System.IO

task {
    /// open the file and note the ***use*** keyword in the file and the client
    use file = File.OpenWrite("./dummy.pdf")
    use client = new HttpClient()
    let! response = client.GetStreamAsync("https://www.w3.org/WAI/ER/tests/xhtml/testfiles/resources/pdf/dummy.pdf")
    /// copy the response contents to the file asynchronously
    do! response.CopyToAsync(file)
    // both file and client will be disposed automatically
    // after they get out of scope
}
|> Async.AwaitTask
// we run synchronously
// to allow the fsi to finish the pending tasks
|> Async.RunSynchronously

```

Enter fullscreen mode Exit fullscreen mode

> To Run this, copy this content into a file named `script.fsx` (or whatever name you prefer) and type:
> 
> -   `dotnet fsi script.fsx`

this should have downloaded a PDF file. The code we used for it is pretty small about 20 LoC including whitespace and comments, now we know how to get strings and binary files with `HttpClient` what about posting Json? let's see and for the following examples we will be using [JsonPlaceholder](https://jsonplaceholder.typicode.com/)  

```
#r "nuget: Ply"

// the following open statement makes the `task {}` CE available
open FSharp.Control.Tasks
open System.Net.Http
open System.Net.Http.Json
// model of a "Post" from the jsonplaceholder website
type Post =
    { userId: int
      id: int
      title: string
      body: string }

task {
    use client = new HttpClient()
    // use an anonymous record to create a partial post
    let partialPost =
        {| userId = 1
           title = "Sample"
           body = "Content" |}

    let! response =
        let url = "https://jsonplaceholder.typicode.com/posts"

        client.PostAsJsonAsync(url, partialPost)

    let! createdPost = response.Content.ReadFromJsonAsync<Post>()
    printfn $"Id: {createdPost.id} - Title: {createdPost.title}"
    // Id: 101 - Title: Sample
}
|> Async.AwaitTask
// we run synchronously
// to allow the fsi to finish the pending tasks
|> Async.RunSynchronously
```

Enter fullscreen mode Exit fullscreen mode

> To Run this, copy this content into a file named `script.fsx` (or whatever name you prefer) and type:
> 
> -   `dotnet fsi script.fsx`

once again we're using a disposable `HttpClient` instance, we're also using an anonymous record of a partial `Post` as the payload of the request, and thanks to the `System.Net.Http.Json` namespace we can also read the content as json from the response's content, we don't need to parse it or handle the streams for that.

In case we just need to do a simple GET that has a Json response we can do it as well  

```
#r "nuget: Ply"

// the following open statement makes the `task {}` CE available
open FSharp.Control.Tasks
open System.Net.Http
open System.Net.Http.Json

type Post =
    { userId: int
      id: int
      title: string
      body: string }

task {
    use client = new HttpClient()
    let url = "https://jsonplaceholder.typicode.com/posts"

    let! posts = client.GetFromJsonAsync<Post[]>(url)

    printfn $"%A{posts}" // prints the 100 post array to the console
}
|> Async.AwaitTask
// we run synchronously
// to allow the fsi to finish the pending tasks
|> Async.RunSynchronously
```

Enter fullscreen mode Exit fullscreen mode

> To Run this, copy this content into a file named `script.fsx` (or whatever name you prefer) and type:
> 
> -   `dotnet fsi script.fsx`

This should give you an idea how to do other kind of http verbs as well and shows basic usage of the `HttpClient` but further operations may be a little cumbersome and that's why I want to show other libraries as well. Let's move on to a relatively popular (2.5k gh stars at the moment of writing) library for HTTP requests

This is a nice library with an API designed to be consumed from C# and you can see at first hand given the fluent style it uses, however it is a really nice library with more ergonomic methods that may make you more productive on the long run than crafting the Http Requests yourself with the BCL's `HttpClient`.

Let's try to replicate the first example we had downloading an html page  

```
#r "nuget: Ply"
#r "nuget: Flurl.Http"

open FSharp.Control.Tasks
open System.IO
open Flurl.Http

task {
    let! content =
        let url = "https://dev.to/tunaxor/doing-some-io-in-f-4agg"

        url.GetStringAsync()

    do! File.WriteAllTextAsync("./response.html", content)
}
|> Async.AwaitTask
// we run synchronously
// to allow the fsi to finish the pending tasks
|> Async.RunSynchronously
```

Enter fullscreen mode Exit fullscreen mode

> To Run this, copy this content into a file named `script.fsx` (or whatever name you prefer) and type:
> 
> -   `dotnet fsi script.fsx`

After executing it, the html page should have downloaded. We can open it and it should work in the same way as the `HttpClient` downloaded page.

> Notice that we didn't need to handle the lifecycle of any `HttpClient` that's done by the library itself, yay! extra points for that 😁

Thankfully dev.to articles look really great since they are rendered on the server also, the same thing of keeping the string in memory applies so let's try the stream sample  

```
#r "nuget: Ply"
#r "nuget: Flurl.Http"

open FSharp.Control.Tasks
open System.IO
open Flurl.Http

task {
    use file = File.OpenWrite("./response.html")

    let! content =
        "https://dev.to/tunaxor/doing-some-io-in-f-4agg"
            .GetStreamAsync()

    do! content.CopyToAsync(file)
}
|> Async.AwaitTask
// we run synchronously
// to allow the fsi to finish the pending tasks
|> Async.RunSynchronously
```

Enter fullscreen mode Exit fullscreen mode

> To Run this, copy this content into a file named `script.fsx` (or whatever name you prefer) and type:
> 
> -   `dotnet fsi script.fsx`

quite nice right? that's one of the benefits of using libraries they are meant to ease some of the pains/productivity issues that you may have using base class library elements.

To further illustrate the point of this you can even skip all of the IO stuff yourself and let the library do this for you.  

```
#r "nuget: Ply"
#r "nuget: Flurl.Http"

open FSharp.Control.Tasks
open System.IO
open Flurl.Http

task {
    let path = Path.GetFullPath(".")

    let! result =
        "https://www.w3.org/WAI/ER/tests/xhtml/testfiles/resources/pdf/dummy.pdf"
            .DownloadFileAsync(path, "dummy.pdf")

    printfn "%s" result
}
|> Async.AwaitTask
// we run synchronously
// to allow the fsi to finish the pending tasks
|> Async.RunSynchronously
```

Enter fullscreen mode Exit fullscreen mode

> To Run this, copy this content into a file named `script.fsx` (or whatever name you prefer) and type:
> 
> -   `dotnet fsi script.fsx`

Let's continue to the JSON part which is as easy as the other samples we've seen  

```
#r "nuget: Ply"
#r "nuget: Flurl.Http"

open FSharp.Control.Tasks
open Flurl.Http

type Post =
    { userId: int
      id: int
      title: string
      body: string }

task {
    let! postResult =
        "https://jsonplaceholder.typicode.com/posts"
            .WithHeaders(
                {| Accept = "application/json"
                   X_MY_HEADER = "my-header-value" |},
                true // replace _ with -
            )
            .PostJsonAsync(
                {| userId = 1
                   title = "Sample"
                   body = "Content" |}
            )
            .ReceiveJson<Post>()

    printfn "%A" postResult
}
|> Async.AwaitTask
// we run synchronously
// to allow the fsi to finish the pending tasks
|> Async.RunSynchronously
```

Enter fullscreen mode Exit fullscreen mode

> To Run this, copy this content into a file named `script.fsx` (or whatever name you prefer) and type:
> 
> -   `dotnet fsi script.fsx`

Even if we include a headers object (and we can even include our custom headers there) the code to create an HTTP request is fairly simple and short, Flurl also has methods for `PATCH`, `PUT`, `OPTIONS` among others so be sure it can cover most of your needs.

Uploading files is slightly different because most of the time the request you need to send hast to be a multipart form and even then the files should be the last items in the request so, let's take a look how it's done with Flurl  

```
#r "nuget: Ply"
#r "nuget: Flurl.Http"

open FSharp.Control.Tasks
open System.IO
open Flurl.Http

task {
    let path = Path.GetFullPath("./dummy.pdf")

    try
        let! response =
            // I don't have a jsonplaceholder endpoint for files
            // so I'll put a fake one here
            "https://sampleurl.nox"
                .PostMultipartAsync(fun content ->
                    content
                        .AddString("firstName", "Jane")
                        .AddString("lastName", "Smith")
                        .AddString("email", "jane@smith.lol")
                        .AddFile("pdfresume", path, "application/pdf")
                    // remember in F# functions always return
                    // so let's ignore the content since we've already set what we need
                    |> ignore)

        printfn "Status code: %i" response.StatusCode
    with ex -> printfn "%s" ex.Message
    // Call failed. No such host is known. (sampleurl.nox:443): POST https://sampleurl.nox
}
|> Async.AwaitTask
// we run synchronously
// to allow the fsi to finish the pending tasks
|> Async.RunSynchronously
```

Enter fullscreen mode Exit fullscreen mode

> To Run this, copy this content into a file named `script.fsx` (or whatever name you prefer) and type:
> 
> -   `dotnet fsi script.fsx`

In this case I don't have an endpoint to test like jsonplaceholder but if you have one feel free to replace the URL and the parameters with yours so you can test it yourself.

Even if we have used a C# library in these F# samples the code works just fine and no kittens died, so by the next time you find a shiny .NET library don't feel less of an F# developer just use what the .NET ecosystem has for you. It's worth mentioning that not every library behaves so well when you use them from other language like `F# <-> C#` But if there are no other alternatives you may be able to write a slim wrapper that helps you ease those pain points, some examples are of these wrappers are

-   [ClosedXML.SimpleSheets](https://github.com/Zaid-Ajaj/ClosedXML.SimpleSheets)
-   [Npgsql.FSharp](https://github.com/Zaid-Ajaj/Npgsql.FSharp)
-   [FSharp.CosmosDb](https://github.com/aaronpowell/FSharp.CosmosDb)

> Edit: Even Flurl itself can have a [slim wrapper](https://gist.github.com/akhansari/96c67d6abf943ed616dd4ba89f9c47b0) 😁 thanks [@akhansari](https://twitter.com/akhansari/status/1371475381467951112?s=20)

But I can hear you "What about a more F#'ish solution?" this is where [FsHttp](https://github.com/ronaldschlenker/FsHttp) comes in

This library offers a more streamlined experience for F# developers this might make you feel more at home if you're looking for an F# library.

For these examples I'll be using the Domain Specific Language (DSL) flavor of this library which exposes what's called Computation Expressions ([CE](https://docs.microsoft.com/en-us/dotnet/fsharp/language-reference/computation-expressions)) which for our purposes let's just say it's some sort of a `Functional Builder` Pattern (if you want to learn more check that link 😁)

Let's download the html page once again.  

```
#r "nuget: Ply"
#r "nuget: SchlenkR.FsHttp"

open FSharp.Control.Tasks
open System.IO
open FsHttp
open FsHttp.DslCE
open FsHttp.Response

task {
    let! response =
        httpAsync {
            GET "https://dev.to/tunaxor/doing-some-io-in-f-4agg" 
        }
    let! content = response |> toTextAsync
    do! File.WriteAllTextAsync("./response.html", content)
}
|> Async.AwaitTask
// we run synchronously
// to allow the fsi to finish the pending tasks
|> Async.RunSynchronously
```

Enter fullscreen mode Exit fullscreen mode

> To Run this, copy this content into a file named `script.fsx` (or whatever name you prefer) and type:
> 
> -   `dotnet fsi script.fsx`

the `httpAsync` CE exposes some methods that are known as _custom operations_. In this case `GET` is a method exposed by the `EagerAsyncHttpBuilder` that takes a `string` as the parameter. Once this CE runs, it returns an `Async<Response>` value but that `let!` (which is the equivalent to `let result = await httpAsync { GET "" }`) is binding the result of the `Async<Response>` to the response variable which at that point it's just a `Response` object.

Let's continue with the stream version knowing that it will not keep the string in memory as the other example did.  

```
#r "nuget: Ply"
#r "nuget: SchlenkR.FsHttp"

open FSharp.Control.Tasks
open System.IO
open FsHttp
open FsHttp.DslCE
open FsHttp.Response

task {
    use file = File.OpenWrite("./response.html")
    let! response = 
        httpAsync { 
            GET "https://dev.to/tunaxor/doing-some-io-in-f-4agg" 
        }
    let! content = response |> toStreamAsync
    do! content.CopyToAsync(file)
}
|> Async.AwaitTask
// we run synchronously
// to allow the fsi to finish the pending tasks
|> Async.RunSynchronously
```

Enter fullscreen mode Exit fullscreen mode

> To Run this, copy this content into a file named `script.fsx` (or whatever name you prefer) and type:
> 
> -   `dotnet fsi script.fsx`

Let's move on to the Json requests. In F# Json Serialization/Deserialization is an interesting topic because we're used so much to use things like `Option`, `Result`, `Discriminated Unions` and other F# specific constructs that sometimes they don't translate well to plain json or from plain json and that's why often F# libraries leave the that aspect out of scope to other libraries like [Thoth.Json](https://thoth-org.github.io/Thoth.Json/) or [FSharp.SystemTextJson](https://github.com/Tarmil/FSharp.SystemTextJson). For now we'll fall back to the BCL and use the [System.Text.Json](https://docs.microsoft.com/en-us/dotnet/api/system.text.json?view=net-5.0) namespace since we are only using primitive types in our model  

```
#r "nuget: Ply"
#r "nuget: SchlenkR.FsHttp"

open FSharp.Control.Tasks
open FsHttp
open FsHttp.DslCE
open FsHttp.Response
open System.Text.Json

type Post =
    { userId: int
      id: int
      title: string
      body: string }

let content =
    JsonSerializer.Serialize(
        {| userId = 1
           title = "Sample"
           body = "Content" |}
    )

task {
    let! response =
        httpAsync {
            POST "https://jsonplaceholder.typicode.com/posts"
            body
            json content
        }

    let! responseStream = response |> toStreamAsync
    let! responseValue = JsonSerializer.DeserializeAsync<Post>(responseStream)
    printfn "%A" responseValue
}
|> Async.AwaitTask
// we run synchronously
// to allow the fsi to finish the pending tasks
|> Async.RunSynchronously
```

Enter fullscreen mode Exit fullscreen mode

> To Run this, copy this content into a file named `script.fsx` (or whatever name you prefer) and type:
> 
> -   `dotnet fsi script.fsx`

FsHttp also has a [Json extensions module](https://github.com/ronaldschlenker/FsHttp#response-content-transformations) that you can use to dynamically access the json data once parsed but I'll skip that for now since it returns a Json object that we must access via the dynamic operator it provides (`json?page.AsInteger()`).

In the case of the multipart request here's the code as well  

```
#r "nuget: Ply"
#r "nuget: SchlenkR.FsHttp"

open FSharp.Control.Tasks
open FsHttp
open FsHttp.DslCE
open System.IO

task {
    let path = Path.GetFullPath("./dummy.pdf")

    try
        let! response =
            httpAsync {
                // feel free to use your endpoint in case you have one
                POST "https://sampleurl.nox"
                multipart
                valuePart "firstName" "Jane"
                valuePart "lastName" "Smith"
                valuePart "email" "jane@smith.lol"
                filePartWithName "pdfresume" path
            }

        printfn "StatusCode %A" response.statusCode
    with ex -> printfn "%s" ex.Message
    // --- FsHttp: running in FSI - Try registering printer...
    // --- FsHttp: Printer successfully registered.
    // One or more errors occurred. (No such host is known. (sampleurl.nox:443))
}
|> Async.AwaitTask
// we run synchronously
// to allow the fsi to finish the pending tasks
|> Async.RunSynchronously
```

Enter fullscreen mode Exit fullscreen mode

> To Run this, copy this content into a file named `script.fsx` (or whatever name you prefer) and type:
> 
> -   `dotnet fsi script.fsx`

There you have it, a few ways to create http requests with three different libraries.

___

One of the cool things of developing F# programs is that you have access to the whole .NET ecosystem so you don't have to re-invent the wheel often you can just re-use what may be decades battle tested code so you can confidently work with it be it C#, F#, or VB.

I'll see you on the next one and as always you can comment below or pinging me on twitter 😁 I hope you have an excellent start of the week.