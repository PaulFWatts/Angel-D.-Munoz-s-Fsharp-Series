Hello there!  
Last time I wrote about using Avalonia and Avalonia.FuncUI to write a simple todo app, you can check that here:

this time we'll do something similarly simple...

A Cross-platform music player!

One of the points I discussed on my last post was about how Avalonia can replace [Electron](https://www.electronjs.org/) in some circumstances and I went to look into [electron apps](https://www.electronjs.org/apps?category=music) and found out that there are lots and lots of music players and youtube downloaders  
and thought to myself

> I've never done a music player before

I've always believed anything in my desktop is way more complicated of what I feel I'm capable of (the same goes for the libraries I use) but I felt this time I was ready to at least try and if I failed I would have learned something new for sure and if not... Let's go for it!

First, I set some goals what do I want this thing to do?

1.  I want it to play music .mp3 and .wav are enough.
2.  I want to have a dead-simple UI just a playlist and a media bar.
3.  I don't want/need to play music from the internet.

Of course in my mind, I had waaaaaay more objectives... I was having a brainstorm sickness I was already building a startup from it... but I came back to earth and settled on those three realistic goals, If I ever continued with that I would have already a solid foundation to grow those ideas up.

Second, What do I need to play music?  
I needed a solution that worked in Windows, MacOS, and Ubuntu and if I'm telling everyone that Avalonia is cross-platform I can't expect everyone to have to write native code for that... so I went on the look on .netcore land and I found out that the VLC library has .net standard libraries to support Windows/Linux/MacOS/Android/iOS/UWP with the help of Xamarin that also meant that the core functionality was on a .netstandard library and of course that also meant it's cross-platform ✅

Third Icons...  
no app looks cool without an icon or two... I was kind of lost here since I was not going to use a ton of icons and I didn't want to rely on the internet this had to be 100% offline. Thankfully the [mdi](https://materialdesignicons.com/) icon library provides thousands of icons in a web font or an svg/xaml (canvas)/xaml (DrawImage), I went for XAML/Canvas since I could just do the canvas icons I would need you can find those definitions [here](https://github.com/AvaloniaCommunity/Avalonia.FuncUI/blob/master/src/Examples/Examples.MusicPlayer/Icons.fs) let's see a quick sample  

```
let play = 
  Canvas.create [
    Canvas.width 24.0
    Canvas.height 24.0
    Canvas.children [ 
      Path.create [ 
        Path.fill "black"
        Path.data "M8,5.14V19.14L19,12.14L8,5.14Z" 
      ] 
    ] 
  ]
```

Enter fullscreen mode Exit fullscreen mode

that is the icon that represents the "Play" button  
with that in mind it seems I already have most of my needs done, let's build something meaningful this time!

### [](https://dev.to/tunaxor/creating-web-components-with-fable-lit-2m11#file-structure)File Structure

Spoilers, it's flat (shocker! 😱), yeah the way F# handles files it's in a top/down manner, so even if you add folders the files have to be in order in the \[PROJECT\_NAME\].fsproj file  

```
Icons.fs
Types.fs
Songs.fs
PlayerLib.fs
Dialogs.fs
Extensions.fs
Playlist.fs
Player.fs
Shell.fs
Program.fs
```

Enter fullscreen mode Exit fullscreen mode

the most important files here are Shell, Player, Playlist and PlayerLib.

Shell is the main module and the one that handles external messages from internal controls. If we'd need to add a different "page"/view it would be in this place we also have a subscriptions module inside of Shell which will help us to handle important messages from our media player like if it's playing, pausing, stopped if the media is ending and things like those.  

```
type ShellWindow() as this =
        inherit HostWindow()
        do
            let player = PlayerLib.getEmptyPlayer
            let programInit (window, player) = init window player, Cmd.none
#if DEBUG
            this.AttachDevTools(KeyGesture(Key.F12))
#endif
            /// we use this function because sometimes we dispatch messages
            /// from another thread
            let syncDispatch (dispatch: Dispatch<'msg>): Dispatch<'msg> =
                match Dispatcher.UIThread.CheckAccess() with
                | true -> fun msg -> Dispatcher.UIThread.Post(fun () -> dispatch msg)
                | false -> fun msg -> dispatch msg

            Program.mkProgram programInit update view
            |> Program.withHost this
            |> Program.withSyncDispatch syncDispatch
            |> Program.withSubscription (fun _ -> Subs.playing player)
            |> Program.withSubscription (fun _ -> Subs.paused player)
            |> Program.withSubscription (fun _ -> Subs.stoped player)
            |> Program.withSubscription (fun _ -> Subs.ended player)
            |> Program.withSubscription (fun _ -> Subs.timechanged player)
            |> Program.withSubscription (fun _ -> Subs.lengthchanged player)
            |> Program.withSubscription (fun _ -> Subs.chapterchanged player)
#if DEBUG
            |> Program.withConsoleTrace
#endif
            |> Program.runWith (this, player)
```

Enter fullscreen mode Exit fullscreen mode

that is our main window, usually, you will see this one called MainWindow. We are registering out subscriptions here so we're able to handle these messages within our Elmish module

A good relevant part here is that there are a few messages that belong only to the Shell, and then the rest of the messages are specific to handle media player elements these messages get called by the subscriptions we used when using our `Program.mkProgram` function call.  

```
type Msg =
    | PlayerMsg of Player.Msg
    | PlaylistMsg of Playlist.Msg
    | SetTitle of string
    | OpenFiles
    | OpenFolder
    | AfterSelectFolder of string
    | AfterSelectFiles of string array
    (* Handle Media Player Events *)
    | Playing
    | Paused
    | Stopped
    | Ended
    | TimeChanged of int64
    | ChapterChanged of int
    | LengthChanged of int64
```

Enter fullscreen mode Exit fullscreen mode

One of the most useful ones here is `Stopped`, `TimeChanged` and `LengthChanged` because that allows us to present a more accurate representation of the media bar. These may or may not trigger a special update within the children's controls.

The update function looks like this  

```
let update (msg: Msg) (state: State) =
  match msg with
    // omitted code...      
    (* The following messages are fired from the player's subscriptions
       I feel these are can help to handle updates accross the whole application
       There are a lot more of events the Player Emits, but for the moment
       we'll work with these *)
        | Playing -> state, Cmd.none
        | Paused -> state, Cmd.none
        | Stopped -> state, Cmd.none
        | Ended -> state, Cmd.map PlaylistMsg (Cmd.ofMsg (Playlist.Msg.GetNext))
        | TimeChanged time -> state, Cmd.map PlayerMsg (Cmd.ofMsg (Player.Msg.SetPos time))
        | ChapterChanged chapter -> state, Cmd.none
        | LengthChanged length -> state, Cmd.none
```

Enter fullscreen mode Exit fullscreen mode

Some of these events don't actually do anything... why?  
They are here for demonstration purposes, let's say you want to save to your persistent storage when a song has been played but only after the user waited for it to finish, then you'll need to handle the `Ended` message perhaps you want to send an update yo your API to what was the last action the user did, you may need to send a request from any of these options. Most of these are just Commands that you'll be passing around to children controls like `Ended` and `TimeChanged` where we tell Playlist and Player that they need to do something (GetNext and SetPos (time) respectively) and you can always remove what you don't need.

Player is the control that contains our play/pause/previous/next/shuffle/repeat buttons it communicates to other controls via external messages. Here we show the two sets of messages this control can dispatch  

```
type ExternalMsg =
    | Next
    | Previous
    | Play
    | Shuffle
    | SetLoopState of Types.LoopState

type Msg =
    | Play of Types.SongRecord
    | Seek of double
    | SetPos of int64
    | SetLength of int64
    | SetLoopState of Types.LoopState
    | Previous
    | Pause
    | Stop
    | PlayInternal
    | Next
    | Shuffle
```

Enter fullscreen mode Exit fullscreen mode

Why do we use external messages? The external messages is a way for us to leverage the Elmish Top/Down communication seamlessly as we return an ExternalMsg Option type in our update function for this module.  
As an example:  

```
/// Player.fs
let update msg state =
    match msg with
    /// ... omitted code ...
    | Shuffle -> state, Cmd.none, Some ExternalMsg.Shuffle
    /// ... omitted code ...


/// Shell.fs
let private handlePlayerExternal (msg: Player.ExternalMsg option) =
    match msg with
    | None -> Cmd.none
    | Some msg ->
        match msg with
        /// ... omitted code ...
        | Player.ExternalMsg.Shuffle -> Cmd.ofMsg (PlaylistMsg(Playlist.Msg.Shuffle))
        /// ... omitted code ...

let update msg state =
    match msg with
    /// ... omitted code...
    | PlayerMsg playermsg ->
        let s, cmd, external = Player.update playermsg state.playerState
        let handled = handlePlayerExternal external
        let mapped = Cmd.map PlayerMsg cmd
        let batch = Cmd.batch [ mapped; handled ]
        { state with playerState = s }, batch
    /// ... omitted code ...

/// Playlist.fs
type Msg =
    /// ... omitted cases ...
    | Shuffle
    /// ... omitted cases ...
let update msg state = 
    match msg with 
    /// ... omitted code ...
    | Shuffle ->
        match state.songList with
        | Some songs ->
            let shuffled = shuffle songs
                { state with
                  songList = Some shuffled
                  currentIndex = 0 }, Cmd.none, None
        | None -> state, Cmd.none, None
```

Enter fullscreen mode Exit fullscreen mode

The chain of messages would be like this

1.  Click on Shuffle on Player.fs
2.  dispatch the Shuffle event that returns a state, a command and an external message
3.  the `Shell.fs` grabs the general `PlayerMsg` (defined in Shell.Msg) and applies the `update` function from the `Player` module `let s, cmd, external = Player.update playermsg state.playerState`
4.  We call `handlePlayerExternal external` which in turns matches the correct message type to the correct command, in this case `Cmd.ofMsg (PlaylistMsg(Playlist.Msg.Shuffle))`
5.  Inside Playlist.fs we capture the Shuffle message and do the corresponding shuffle logic

To play a song it's a similar flow

1.  double click a song in the playlist and dispatch the `PlaySong of Types.SongRecord` message.
2.  find the index of the song in the list and then update the state with the currentIndex. Return the updated state, an empty command and the external command `PlaySong of index: int * song: Types.SongRecord`
3.  The Shell module grabs the general `PlaylistMsg` and calls the update function from the PlaylistModule
4.  call `handlePlaylistExternal external` to match the correct command
5.  Inside `Player.fs` we handle the `Play of Types.SongRecord` message and play the newly assigned media

Now, you might think well this is a lot of code for a simple event, I'd just register an event listener for the event from the children component in javascript but I'll also say that it's not that simple... In Vue, you might need to use an [event bus](https://alligator.io/vuejs/global-event-bus/) in aurelia you might need a [Event Aggregator](https://ilikekillnerds.com/2016/02/working-with-the-aurelia-event-aggregator/) in react I think you would be using something like [Redux](https://redux.js.org/) in which case it's already quite similar ([read "Prior Art"](https://redux.js.org/introduction/prior-art#elm)).

And those same 5 steps are shared with almost any other external message it's a predictable way to handle external updates to other controls.  
PlayerLib is a module with two special functions that allow us to do the whole player thing working  

```
module PlayerLib =
    open LibVLCSharp.Shared

    let getMediaFromlocal (source: string) =
        use libvlc = new LibVLC()
        new Media(libvlc, source, FromType.FromPath)

    let getEmptyPlayer =
        use libvlc = new LibVLC()
        new MediaPlayer(libvlc)
```

Enter fullscreen mode Exit fullscreen mode

LibVlCSharp has quite a lot of features including playing media from the network, finding devices like Chromecast and a ton of other media playing information in this case we just need a media player and a media object to play within our media player.

What does that look like?

[![Raznor App](https://res.cloudinary.com/practicaldev/image/fetch/s--fKuRkqHv--/c_limit%2Cf_auto%2Cfl_progressive%2Cq_auto%2Cw_880/https://dev-to-uploads.s3.amazonaws.com/i/4u348d33vx8pwz84e0fs.png)](https://res.cloudinary.com/practicaldev/image/fetch/s--fKuRkqHv--/c_limit%2Cf_auto%2Cfl_progressive%2Cq_auto%2Cw_880/https://dev-to-uploads.s3.amazonaws.com/i/4u348d33vx8pwz84e0fs.png)

As I said before, it's just dead simple only pick files and play them... No more, no less.

On my last post also I mentioned the following

> When I want to do things for myself, I don't want to have a lot of  
> resources being consumed by a note-taking app

And I did verify that  
[![Not that much](https://res.cloudinary.com/practicaldev/image/fetch/s--pRVTRZOy--/c_limit%2Cf_auto%2Cfl_progressive%2Cq_auto%2Cw_880/https://dev-to-uploads.s3.amazonaws.com/i/6f6zsmwzu11mntg0vihh.png)](https://res.cloudinary.com/practicaldev/image/fetch/s--pRVTRZOy--/c_limit%2Cf_auto%2Cfl_progressive%2Cq_auto%2Cw_880/https://dev-to-uploads.s3.amazonaws.com/i/6f6zsmwzu11mntg0vihh.png)

As you can see the app is playing a song and it is using almost 60MB of RAM and nearly 2% CPU usage. I'd say this is the Note Taking App version of the Music Players. I can safely say that I wouldn't mind running the same amount of Avalonia Apps as the Electron ones I currently run (on my main pc) on my old 4gb desktop pc that's somewhere in here.

I won't say anything related to Spotify there because it's not fair comparison Spotify does do a lot more than what my dead simple player does.

Where can I find this?  
Two places:

[![Avalonia FuncUI](https://res.cloudinary.com/practicaldev/image/fetch/s--aEHEw4ox--/c_limit%2Cf_auto%2Cfl_progressive%2Cq_auto%2Cw_880/https://github.com/fsprojects/Avalonia.FuncUIgithub/img/logo/FuncUI.png)](https://github.com/fsprojects/Avalonia.FuncUIgithub/img/logo/FuncUI.png)

Develop cross-platform MVU GUI Applications using F# and Avalonia!

[![](https://camo.githubusercontent.com/99dfab5153657033f4e3bb29c82f2cf117aee5f33e4bacb19af29905d0b28ae0/68747470733a2f2f766f796f6e69632d6c6162732e76697375616c73747564696f2e636f6d2f4176616c6f6e69612e46756e6355492f5f617069732f6275696c642f7374617475732f4176616c6f6e6961436f6d6d756e6974792e4176616c6f6e69612e46756e6355493f6272616e63684e616d653d6d6173746572)](https://voyonic-labs.visualstudio.com/Avalonia.FuncUI/_apis/build/status/AvaloniaCommunity.Avalonia.FuncUI?branchName=master) [![GitHub top language](https://camo.githubusercontent.com/4980f0f4c9ba5e75fa572d91820bb67ebdc8b46d64bf3e0205e9be6a87b61c0a/68747470733a2f2f696d672e736869656c64732e696f2f6769746875622f6c616e6775616765732f746f702f4a61676765724a6f2f4176616c6f6e69612e46756e635549)](https://camo.githubusercontent.com/4980f0f4c9ba5e75fa572d91820bb67ebdc8b46d64bf3e0205e9be6a87b61c0a/68747470733a2f2f696d672e736869656c64732e696f2f6769746875622f6c616e6775616765732f746f702f4a61676765724a6f2f4176616c6f6e69612e46756e635549) [![GitHub repo size](https://camo.githubusercontent.com/1ff79515a13637f6331ea68132ee72fa964379506c6c46f0bfa24d01462863f1/68747470733a2f2f696d672e736869656c64732e696f2f6769746875622f7265706f2d73697a652f4a61676765724a6f2f4176616c6f6e69612e46756e635549)](https://camo.githubusercontent.com/1ff79515a13637f6331ea68132ee72fa964379506c6c46f0bfa24d01462863f1/68747470733a2f2f696d672e736869656c64732e696f2f6769746875622f7265706f2d73697a652f4a61676765724a6f2f4176616c6f6e69612e46756e635549) [![](https://camo.githubusercontent.com/9076ef95e836cd28733995218faaba6515082405ccee6a8c5234d44db43ab27c/68747470733a2f2f696d672e736869656c64732e696f2f6769746875622f6c6963656e73652f4a61676765724a6f2f4176616c6f6e69612e46756e635549)](https://camo.githubusercontent.com/9076ef95e836cd28733995218faaba6515082405ccee6a8c5234d44db43ab27c/68747470733a2f2f696d672e736869656c64732e696f2f6769746875622f6c6963656e73652f4a61676765724a6f2f4176616c6f6e69612e46756e635549) [![](https://camo.githubusercontent.com/d80d0ae5e0e90c1bedb6c992971d24031ef6b9c125281d3f1e48a3cddb7db81c/68747470733a2f2f6261646765732e6769747465722e696d2f4176616c6f6e69612d46756e6355492f636f6d6d756e6974792e737667)](https://gitter.im/Avalonia-FuncUI/community?utm_source=badge&utm_medium=badge&utm_campaign=pr-badge)

[![](https://res.cloudinary.com/practicaldev/image/fetch/s--KKKw6DZG--/c_limit%2Cf_auto%2Cfl_progressive%2Cq_auto%2Cw_880/https://github.com/fsprojects/Avalonia.FuncUIgithub/img/hero.png)](https://github.com/fsprojects/Avalonia.FuncUIgithub/img/hero.png) _(Application was created using Avalonia.FuncUI!)_

## About

This library allows you to write cross-platform GUI Applications entirely in F# - No XAML, but a declarative Elm-like DSL. MVU (Model-View-Update) architecture support is built in, and bindings to use it with Elmish are also ready to use.

## Getting started

-   🧱[Templates](https://github.com/AvaloniaCommunity/Avalonia.FuncUI.ProjectTemplates)
    
-   📓[Examples](https://github.com/AvaloniaCommunity/Avalonia.FuncUI/tree/master/src/Examples)
    
-   📚[Documentation](https://avaloniacommunity.github.io/Avalonia.FuncUI.Docs/)
    

## Contributing

Please contribute to this library through issue reports, pull requests, code reviews, documentation, and discussion.

## Example

Below is the code of a simple counter app (using the Avalonia.FuncUI.Elmish package).

module Counter \=
    type CounterState \= {
        count : int
    }

    let init \= {
        count \= 0
    }

    type Msg \=
    | Increment
    | Decrement

    let update (msg: Msg) (state: CounterState) : CounterState \=
        match msg with
        | Increment \-> { state with count \=  state.count + 1 }

…

Enter fullscreen mode Exit fullscreen mode

in the `src/Examples/MusicPlayer` section which I just PR'd yesterday (It's an awesome feeling to contribute something to an open-source project 🐱💻)

or the unpolished and unorganized here  

This is a sample Avalonia + Avalonia.FuncUI desktop application

you can check some thoughts on a similar app at this [blog post](https://dev.to/tunaxor/desktop-apps-with-avalonia-and-fsharp-4n21)

[![Raznor App Gif](https://res.cloudinary.com/practicaldev/image/fetch/s--DRvGUHp2--/c_limit%2Cf_auto%2Cfl_progressive%2Cq_66%2Cw_880/https://github.com/AngelMunoz/Raznor./playerzz.gif)](https://github.com/AngelMunoz/Raznor./playerzz.gif)

  
  

Either way, do you have questions or comments? Leave them below or reach me in twitter :) Have a nice week!