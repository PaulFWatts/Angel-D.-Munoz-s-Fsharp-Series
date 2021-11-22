---
aliases: [dotnet]
tags: [fsharp]
---
-----
[Doing some IO in F# - 2 - DEV Community 👩‍💻👨‍💻](https://dev.to/tunaxor/doing-some-io-in-f-2-3b6d)

This is the second post in Simple things in F# I felt the IO chapter is quite big so I split the `File` and `Directory`

> **_DISCLAIMER_**: Remember that IO operations are never guaranteed so you will need to catch errors on your real code I will not discuss exception handling here since that's a complete different topic but keep it in mind please,

### [](https://dev.to/tunaxor/doing-some-io-in-f-4agg#directory)Directory

The `Directory` class offers similar functionality to the `File` class it has Create and Delete methods as well as helpers to enumerate the directories  

```
open System.IO

let path = Path.Combine(".", "my-dir")
let dirinfo = Directory.CreateDirectory(path)
printfn $"%s{dirinfo.Name} - %s{dirinfo.FullName}"
// my-dir - C:\Users\scyth\repos\blogpostdrafts\my-dir

// To Delete the directory
// use either dirinfo.Delete
// dirinfo.Delete()
// or Directory.Delete
Directory.Delete(dirinfo.FullName)
```

Enter fullscreen mode Exit fullscreen mode

> If you try to delete a directory that is not empty you will get an exception, you can just pass a Boolean to recursively delete everything within that directory
> 
> ```
> Directory.Delete(dirinfo.FullName, true)
> // or
> dirinfo.Delete(true)
> ```

that is pretty nice and if you pay attention to the IDE hints, you'll see that `CreateDirectory` returns a DirectoryInfo instance which is another class that also has more less the same methods that Directory has, the I think is that `DirectoryInfo` operates on a specific directory instead of a general purpose class for directories

What if we want to get the files and directories on the current path?  

```
open System.IO

let files =
    Directory.EnumerateFiles(".")
    |> Seq.fold (fun prev next -> $"{prev}\n\t{next}") ""

let directories =
    Directory.EnumerateDirectories(".")
    |> Seq.fold (fun prev next -> $"{prev}\n\t{next}") ""

printfn $"files:\t{files}\n directories:\t{directories}"
```

Enter fullscreen mode Exit fullscreen mode

that will get you something like  

```
files:
        .\README.md
        .\sample.log
        .\simple-things-in-fsharp-1.md
 directories:
        .\.git
        .\.ionide
        .\samples
        .\scripts
```

Enter fullscreen mode Exit fullscreen mode

depending on your system of course, now this will only get you the path to the files or directories back if you want to operate with those you would need to that path and the `File` or `Directory` classes to do what you need but you can create a `DirectoryInfo` instance yourself  

```
open System.IO

let dir = DirectoryInfo(".")

let dirs = dir.EnumerateDirectories()
let files = dir.EnumerateFiles()

printfn "directories: %s\nfiles: %s" (dirs.GetType().ToString()) (files.GetType().ToString())
// directories: System.IO.Enumeration.FileSystemEnumerable`1[System.IO.DirectoryInfo]
// files: System.IO.Enumeration.FileSystemEnumerable`1[System.IO.FileInfo]
```

Enter fullscreen mode Exit fullscreen mode

If we want to recursively travel all the directories within a directory we can use `SearchOption.AllDirectories` which will get us a flat list of directories under the specified path  

```
open System.IO

let dir = DirectoryInfo(".")
let directories = dir.GetDirectories("*.*", SearchOption.AllDirectories)
printfn "%A" directories
(*
[|C:\Users\scyth\repos\blogpostdrafts\.git;
  C:\Users\scyth\repos\blogpostdrafts\.ionide;
  C:\Users\scyth\repos\blogpostdrafts\samples;
  C:\Users\scyth\repos\blogpostdrafts\scripts;
  C:\Users\scyth\repos\blogpostdrafts\.git\hooks;
  C:\Users\scyth\repos\blogpostdrafts\.git\info;
  C:\Users\scyth\repos\blogpostdrafts\.git\logs;
  C:\Users\scyth\repos\blogpostdrafts\.git\objects;
  C:\Users\scyth\repos\blogpostdrafts\.git\refs;
  C:\Users\scyth\repos\blogpostdrafts\samples\samples2;
  C:\Users\scyth\repos\blogpostdrafts\.git\logs\refs;
  C:\Users\scyth\repos\blogpostdrafts\.git\objects\info;
  C:\Users\scyth\repos\blogpostdrafts\.git\objects\pack;
  C:\Users\scyth\repos\blogpostdrafts\.git\refs\heads;
  C:\Users\scyth\repos\blogpostdrafts\.git\refs\remotes;
  C:\Users\scyth\repos\blogpostdrafts\.git\refs\tags;
  C:\Users\scyth\repos\blogpostdrafts\samples\samples2\samples3;
  C:\Users\scyth\repos\blogpostdrafts\.git\logs\refs\heads;
  C:\Users\scyth\repos\blogpostdrafts\.git\logs\refs\remotes;
  C:\Users\scyth\repos\blogpostdrafts\.git\refs\remotes\origin;
  C:\Users\scyth\repos\blogpostdrafts\samples\samples2\samples3\samples4;
  C:\Users\scyth\repos\blogpostdrafts\.git\logs\refs\remotes\origin|]
*)
```

Enter fullscreen mode Exit fullscreen mode

that's what gets printed in my case. You can play with the pattern to filter out unwanted directories

Let's now do a relatively common exercise create a tree like structure of the directory layout

> **_NOTE_**: Please keep in mind that this might be a naïve way to implement this and is not the best solution it's just a way to do it and always refer to a more experimented colleague when you want to do things like these  

```
open System.IO

// first let's define our data type that will 
type Hierarchy =
    { Name: string
      Files: string list
      Directories: Hierarchy list }


let getFiles (dir: DirectoryInfo) =
    dir.EnumerateFiles()
    |> Seq.map (fun file -> file.FullName)
    |> List.ofSeq

let getDirectories (dir: DirectoryInfo) =
    dir.EnumerateDirectories()
    |> Seq.filter (fun dir -> not (dir.Name.StartsWith(".")))
    |> List.ofSeq

/// we'll define a recursive function that will take a `List<DirectoryInfo>`
/// and returns a `List<Hierarchy>`
let rec getDirHierarchy (directories: DirectoryInfo list) =
    directories
    |> List.map
        (fun dir ->
            { Name = dir.FullName
              Files = getFiles dir
              Directories = getDirHierarchy (getDirectories dir) })

let getHierarchy (path: string) =
    let dir = DirectoryInfo(path)

    { Name = dir.FullName
      Files = getFiles dir
      Directories = getDirHierarchy (getDirectories dir) }

let hierarchy = getHierarchy "."
printfn $"%A{hierarchy}"
(*
{ Name = "C:\Users\scyth\repos\blogpostdrafts"
  Files =
   [C:\Users\scyth\repos\blogpostdrafts\README.md;
    C:\Users\scyth\repos\blogpostdrafts\sample.log;
    C:\Users\scyth\repos\blogpostdrafts\simple-things-in-fsharp-1.md;
    C:\Users\scyth\repos\blogpostdrafts\simple-things-in-fsharp-2.md]
  Directories =
   [{ Name = "C:\Users\scyth\repos\blogpostdrafts\samples"
      Files = [C:\Users\scyth\repos\blogpostdrafts\samples\moved.txt]
      Directories =
       [{ Name = "C:\Users\scyth\repos\blogpostdrafts\samples\samples2"
          Files =
           [C:\Users\scyth\repos\blogpostdrafts\samples\samples2\moved2.txt]
          Directories =
           [{ Name =
               "C:\Users\scyth\repos\blogpostdrafts\samples\samples2\samples3"
              Files = []
              Directories =
               [{ Name =
                   "C:\Users\scyth\repos\blogpostdrafts\samples\samples2\samples3\samples4"
                  Files = []
                  Directories = [] }] }] }] };
    { Name = "C:\Users\scyth\repos\blogpostdrafts\scripts"
      Files =
       [C:\Users\scyth\repos\blogpostdrafts\scripts\http.fsx;
        C:\Users\scyth\repos\blogpostdrafts\scripts\stif-1-2.fsx;
        C:\Users\scyth\repos\blogpostdrafts\scripts\stif-1-3.fsx;
        C:\Users\scyth\repos\blogpostdrafts\scripts\stif-1-4.fsx;
        C:\Users\scyth\repos\blogpostdrafts\scripts\stif-1-5.fsx;
        C:\Users\scyth\repos\blogpostdrafts\scripts\stif-1-6.fsx;
        C:\Users\scyth\repos\blogpostdrafts\scripts\stif-1-7.fsx;
        C:\Users\scyth\repos\blogpostdrafts\scripts\stif-1-8.fsx;
        C:\Users\scyth\repos\blogpostdrafts\scripts\stif-1.fsx]
      Directories = [] }] }
*)
```

Enter fullscreen mode Exit fullscreen mode

to move a directory and all of it's contents simply use the `DirectoryInfo.MoveTo` or `Directory.Move` but please keep in mind that `MoveTo` is a method of an instance of `DirectoryInfo`  

```
open System.IO

Directory.Move("./samples", "./samples2")

// check the directory was moved before running the last bits

let dir = DirectoryInfo("./samples2")
// move it back to it's original place
dir.MoveTo("./samples")
```

Enter fullscreen mode Exit fullscreen mode

So, That's it! I think with this we can close the IO chapter these are common operations of the IO namespace but keep in mind this already exists [here](https://docs.microsoft.com/en-us/dotnet/standard/io/common-i-o-tasks) but since most samples are in C# you might feel intimidated or feel that you are not doing in an _F# idiomatic way_ I'd say that you should not worry about it remember that your code runs in `.NET` so don't feel afraid to us existing libraries from other languages such as C# and VB. Feel f

As always feel free to comment below or to ping me in twitter 😁