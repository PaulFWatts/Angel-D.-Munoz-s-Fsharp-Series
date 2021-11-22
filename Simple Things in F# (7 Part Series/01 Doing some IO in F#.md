---
aliases: [dotnet]
tags: [fsharp]
---
-----
[Doing some IO in F# - DEV Community 👩‍💻👨‍💻](https://dev.to/tunaxor/doing-some-io-in-f-4agg)

As part of my effort to share the F# goodness I'll start a series of blog posts named "Simple things in F#" while most of these posts will cover .NET aspects in general (File IO, Http Requests, RESTful API's, etc) the code samples will be in F# and they will be as simple as possible.

> **_DISCLAIMER_**: I'll be using [Ply](https://github.com/crowded/ply) in most of the samples due to it's efficient task CE and the easy interoperability between C# tasks as well as F#'s async
> 
> **_DISCLAIMER 2_**: Remember that IO operations are never guaranteed so you will need to catch errors on your real code I will not discuss exception handling here since that's a complete different topic but keep it in mind please

Let's begin this series with some simple IO.  
the .NET BCL (Base Class Library) has a bunch of useful classes that allow us to do things in a "standard" way in this case we'll be using the [System.IO](https://docs.microsoft.com/en-us/dotnet/api/system.io?view=net-5.0) namespace.

#### [](https://dev.to/tunaxor/doing-some-io-in-f-4agg#path)Path

The [Path](https://docs.microsoft.com/en-us/dotnet/api/system.io.file?view=net-5.0) class has some cool methods that allow us to manipulate the path to files and directories in a safe, cross-platform manner it's worth mentioning that `Path` class will only deal with the strings that look like a system path, but they won't check if they exist in the disk that will happen when you do the IO operation.  

```
open System.IO
/// this will get the deepest directory name in a path like string
let dirname = Path.GetDirectoryName("./scripts/script.fsx")
// dirname = @".\scripts" // on windows
// dirname = @"./scripts" // on unix like systems

let ext =
    Path.GetExtension("./scripts/script.fsx")

let filename = Path.GetFileName("./scripts/script.fsx")

let filenamenoext =
    Path.GetFileNameWithoutExtension("./scripts/script.fsx")

let fullpath = Path.GetFullPath("./scripts/script.fsx")
printfn $"{ext}, {filename}, {filenamenoext}\n{fullpath}"
// .fsx, script.fsx, script
// c:\Users\scyth\repos\blogpostdrafts\scripts\scripts\script.fsx
```

Enter fullscreen mode Exit fullscreen mode

these are just a few samples the one I use the most is `Path.Combine` which takes any amount of strings and combines them in a single path like string. e.g.  

```
open System
open System.IO

let pathlike =
    Path.Combine(@"..\", $"{Guid.NewGuid()}",  $"{Guid.NewGuid()}", "finalfile.fsx")

printfn "%s" pathlike
// ..\19cc7bf9-ba2e-4cb9-ba00-aafc80b71dce\059b0260-501c-4be8-ad99-22625f2d9a7a\finalfile.fsx
// of course your guids will vary
```

Enter fullscreen mode Exit fullscreen mode

Also, there other two that I want to mention as well  

```
Path.GetInvalidFileNameChars()
Path.GetInvalidPathChars()
```

Enter fullscreen mode Exit fullscreen mode

before creating a file/path perhaps you want to check that you don't have disallowed characters and get an exception ahead.

That being said, with these you can start preparing safe path like strings let's put these in practice as well.

#### [](https://dev.to/tunaxor/doing-some-io-in-f-4agg#file)File

The [File](https://docs.microsoft.com/en-us/dotnet/api/system.io.file?view=net-5.0) class, it is a class that provide static methods to work with... yes, Files... this class provides both sync and async methods as well as streams, depending on your use cases each of the alternatives might be more efficient than others or more simple to work with.

Let's start by creating and deleting a file first  

```
// let's build a path from the system's temp path and a "Sample.txt" filename
let path =
    Path.Combine(Path.GetTempPath(), "Sample.txt")

printfn $"Does the file exists yet? {File.Exists(path)}"
let file = File.Create(path)
// let's close the file and check the name
file.Close()
printfn $"{file.Name}"
// and delete it afterwards
File.Delete(path)
// we can check if we did delete it
printfn $"Did we delete the file? {not (File.Exists(path))}"

```

Enter fullscreen mode Exit fullscreen mode

> **_NOTE_**: [The File System Is Unpredictable](https://blog.paranoidcoding.com/2009/12/10/the-file-system-is-unpredictable.html) so don't rely too much on `File.Exists` 😁

when you create a file in that way you get back an instance of a `FileStream` since I'm using the fsi interpreter it will not allow me to use `use` but remember that streams are disposable so you can use them this way as well  

```
#r "nuget: Ply"

open FSharp.Control.Tasks
open System
open System.IO

let createSampleFile() = 
    task {
        let path =
            Path.Combine(Path.GetTempPath(), "Sample.txt")
        /// notice ***use*** instead of ***let***
        use file = File.Create(path)
        let bytes = System.Text.Encoding.UTF8.GetBytes("Sample content")
        do! file.WriteAsync (ReadOnlyMemory bytes)
         // after the file gets out of the scope
         // the resources will be disposed automatically
    }
```

Enter fullscreen mode Exit fullscreen mode

To open an existing file we can use the following methods

-   File.Open
-   File.OpenRead
-   File.OpenText
-   File.OpenWrite

the `Open` method will lock the file until its closed either disposing it or manually closing it which is the one you may want when you want to ensure only you at a given moment is operating on that file.

`OpenRead`, `OpenWrite`, and `Open` will give you a `FileStream` instance while `OpenText` will give you a `StreamReader` they are all streams but depending on the operation you are performing perhaps you would want to prefer one thing above the other if you know you are operating on text files well, that's what `OpenText` is for since the `StreamReader` will be in UTF8 Encoding.

`OpenWrite` Will create a file if it doesn't exist, if it does then it will append whatever you write to it which could be good for a log file let's see a brief sample

> **_NOTE_**: We will ignore threading issues or similar things to keep it simple but you should not use this naïve approach to logging
> 
> To Run this, copy this content into a file named `script.fsx` (or whatever name you prefer) and type:
> 
> -   `dotnet fsi script.fsx`

```
#r "nuget: Ply"

open FSharp.Control.Tasks
open System
open System.Collections.Generic
open System.IO
open System.Text

let queue = new Queue<string>()

let log (logValue: string) = queue.Enqueue logValue

/// naive approach to logging 
let flushToLog () =
    task {
        let path = Path.Combine("./", "sample.log")
        /// note the use of ***use*** and not ***let***
        /// since this is a disposable stream 
        /// we use the automatic disposing mechanisms in .NET
        use file = File.OpenWrite(path)

        while queue.Count > 0 do
            // let's move the stream's position to the end so we don't overwrite what was previously written

            file.Position <- file.Length
            let line = queue.Dequeue()
            printfn "%s" line
            let bytes = Encoding.UTF8.GetBytes $"%s{line}\n"
            do! file.WriteAsync(ReadOnlyMemory bytes)
    }

task {
    for i in 1 .. 10 do
        log $"Logging: %i{i}"

    do! flushToLog ()

    for i in 11 .. 20 do
        log $"Logging: %i{i}"

    do! flushToLog ()

    for i in 21 .. 30 do
        log $"Logging: %i{i}"

    do! flushToLog ()
}
|> Async.AwaitTask
// we need to run this synchronously
// so the fsi can finish executing the tasks
|> Async.RunSynchronously
```

Enter fullscreen mode Exit fullscreen mode

that script defines a `log` function that puts a string in a string `Queue`.

It also defines a function `flushToLog` which when called opens a file called **sample.log** and as log as the `Queue` has strings on it, it moves the `FileStream`'s position to the end, it takes a string from the queue converts it into a `UTF8 byte array` which writes asynchronously (as if you were using async/await in C# or javascript) then the file moves out of scope and gets disposed automatically

as part of the exercise we then write 31 lines (including the last line break) to the file in three batches, you may notice that if the file doesn't exist it gets created the first time, if you run that script multiple times you will have 31 lines repeated over and over again without deleting the previous content.

Keep in mind that there are simpler API's that can be used if you are not going to do complex operations in a file, instead of managing a `FileStream` you could use `File.WriteAllLinesAsync` or `File.AppendAllLinesAsync`, replace the `flushToLog` function with the following  

```
let flushToLog () =
    task {
        let path = Path.Combine("./", "sample.log")
        // do! File.WriteAllLinesAsync(path, queue)
        do! File.AppendAllLinesAsync(path, queue)
        queue.Clear()
    }
```

Enter fullscreen mode Exit fullscreen mode

comment and un-comment the lines starting with `do!` to see the difference between each other 😀

to read the content's we can use the following API's and their async counterparts

-   File.ReadAllLines - returns an array of strings
-   File.ReadAllText - returns the whole content as a single string

these will give us a simpler way to deal with the contents rather than using the StreamReader or the FileStream and manually read bytes and move the position of the stream

> To Run this, copy this content into a file named `script.fsx` (or whatever name you prefer) and type:
> 
> -   `dotnet fsi script.fsx`

```
#r "nuget: Ply"

open FSharp.Control.Tasks
open System
open System.IO


task {
    let path = Path.Combine("./", "sample.log")
    let! lines = File.ReadAllLinesAsync path
    let! content = File.ReadAllTextAsync path
    printfn $"Content:\n\n{content}"
    printfn $"Lines in file: %i{lines.Length}"
}
|> Async.AwaitTask
// we need to run this synchronously
// so the fsi can finish executing the tasks
|> Async.RunSynchronously
```

Enter fullscreen mode Exit fullscreen mode

that will print the content of the file to the console and the lines in the file, in my case since I've played with it a couple of times it told me my log has 90 lines

Let's move back to operations on the file as a whole since we now kind of know how to open/read/write/delete a file what about copying and moving files around?

> To Run this, copy this content into a file named `script.fsx` (or whatever name you prefer) and type:
> 
> -   `dotnet fsi script.fsx`

```
open System
open System.IO
let path = Path.Combine("./", "sample.log")
let filename = Path.ChangeExtension(path, "txt")
File.Copy(path, filename)
```

Enter fullscreen mode Exit fullscreen mode

We used the `Copy` method this time plus a nice utility of the `Path` class, after running that you should have a _New_ file with the name **sample.txt**

> To Run this, copy this content into a file named `script.fsx` (or whatever name you prefer) and type:
> 
> -   `dotnet fsi script.fsx`

```
open System
open System.IO

let source = Path.Combine("./", "sample.txt")

let destiny =
    // assuming that the directory "samples" exists
    Path.Combine("./", "samples", "moved.txt")

File.Move(source, destiny)
```

Enter fullscreen mode Exit fullscreen mode

So, that's it there are other IO operations with files that can be done, but I refrained myself to the most common ones or the ones I feel newcomers to the language might feel lost in the next post I'll continue the IO posts with the Directory so stay tuned!

As always feel free to ping me on twitter or in the comments below 😁