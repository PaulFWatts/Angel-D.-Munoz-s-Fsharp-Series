I'm a pretty much the average person on different languages Javascript, C#, Python, Java some Scala and some F#  
I mostly work with Javascript and if you're like me perhaps you have tried to do some .net but didn't find C#/XAML that much interested in the mid-run.  
Perhaps you tried F# liked it but didn't find a reason to use it that much because you are already using another back-end stack.  
Well this time I tried out doing Cross-Platform Desktop Apps with F# let's see how it turns out

### [](https://dev.to/tunaxor/creating-web-components-with-fable-lit-2m11#mvu-architecture)MVU Architecture

The MVU architecture also known as the Elm architecture is rather a simple one and born in the browser (as far as I know),  
every time I see a language showing the MVU architecture is always the same counter sample

I did try many many many times to grasp this concept and use it but I failed several times and I want to believe the reason many of us that don't get it at first glance is due to having to deal with browser only things:

-   Promises
-   HTML5 Routing
-   DOM Elements
-   Javascript Interoperability

So, your usual counter app works nice but once you need to deal with these other things it can get messy real quick if you're not used to the architecture yet.

## [](https://dev.to/tunaxor/creating-web-components-with-fable-lit-2m11#enter-avalonia-and-avaloniafuncui)Enter Avalonia and Avalonia.FuncUI

This is a sample Avalonia + Avalonia.FuncUI desktop application

you can check some thoughts and a brief explanation at this [blog post](https://dev.to/tunaxor/desktop-apps-with-avalonia-and-fsharp-4n21)

[![AvaFunc Gif](https://res.cloudinary.com/practicaldev/image/fetch/s--SacGQEEs--/c_limit%2Cf_auto%2Cfl_progressive%2Cq_66%2Cw_880/https://github.com/AngelMunoz/AvaFunc./avafunc.gif)](https://github.com/AngelMunoz/AvaFunc./avafunc.gif)

  
  

This is a small project which aims to test some aspects of Avalonia and Avalonia.FuncUI  
In particular the following

1.  "**_MultiPage_**" apps
2.  Reusability
3.  Crossplatform Electron Replacement

## [](https://dev.to/tunaxor/creating-web-components-with-fable-lit-2m11#1-multipage-apps)1\. "**_MultiPage_**" Apps

First of all, we're in the desktop **_Pages_** don't really exist so you have multiple modules that look almost the same as your usual counter sample let's see one  

```
namespace AvaFunc
module Shell =

    type State =
        { (* ... *) }

    let init =
        { (* ... *) }

    type Msg =
        // omitted code
    let update (msg: Msg) (state: State) =
        // omitted code
    let view (state: State) dispatch =
        // omitted code
```

Enter fullscreen mode Exit fullscreen mode

Every time you need to create a different **_Page_** you can create a different file that includes these same elements:

-   State (also known as Model)
-   init
-   Msg
-   update
-   view

let's add some code to have a "_routing_" like thing here  

```
namespace AvaFunc
open Avalonia.Controls
open Avalonia.Layout
open Avalonia.Media
open Avalonia.FuncUI.DSL
module Shell =
    type Page = 
      | Home
      | About

    type State =
        { currentPage: Page }

    let init =
        { currentPage = Page.Home }

    type Msg =
      | NavigateTo of Page

    let update (msg: Msg) (state: State) =
      match msg with
      | NavigateTo page ->
        { state with currentPage = page }

    let viewMenu state dispatch = 
        Menu.create [
          Menu.viewItems [ 
            MenuItem.create [ 
              MenuItem.onClick (fun _ -> dispatch (NavigateTo Home))
              MenuItem.header "Home" 
            ]
            MenuItem.create [ 
              MenuItem.onClick (fun _ -> dispatch (NavigateTo About))
              MenuItem.header "About" 
            ]
        ]

    let view (state: State) dispatch =
        DockPanel.create [
          DockPanel.children [
            yield viewMenu state dispatch
            match state.currentPage with 
            | Home -> 
              yield TextBox.create [ TextBox.dock Dock.Bottom TextBox.text "Hello Home" ]
            | About -> 
              yield TextBox.create [ TextBox.dock Dock.Bottom TextBox.text "Hello About" ]
          ]
        ]
```

Enter fullscreen mode Exit fullscreen mode

Quite a length right? let's go piece by piece, First, declare your page types  
and save it in your state  

```
type Page = 
  | Home
  | About

type State =
    { currentPage: Page }

let init =
    { currentPage = Page.Home }
```

Enter fullscreen mode Exit fullscreen mode

then, declare your "**_events_**" and what will you do when they get called  
here we have a single "**_event_**" which is `NavigateTo of Page`  
the reason I call it "**_event_**" is because it is not an event like any other js event, in reality, it's just named (message) that identifies an action in kind of this way

> hey! this action has been performed what are you going to do about it?  

```
type Msg =
  | NavigateTo of Page

let update (msg: Msg) (state: State) =
  match msg with
  | NavigateTo page ->
    { state with currentPage = page }
```

Enter fullscreen mode Exit fullscreen mode

in response to the NavigateTo and the page it brings with it, we simply update the state and set the actual page.

Now the most verbose part of it, the one that concerns to the views (the actual UI stuff)  

```
let viewMenu state dispatch = 
    // omited code

let view (state: State) dispatch =
    DockPanel.create [
      DockPanel.children [
        yield viewMenu state dispatch
        match state.currentPage with 
        | Home -> 
          yield TextBox.create [ TextBox.dock Dock.Bottom TextBox.text "Hello Home" ]
        | About -> 
          yield TextBox.create [ TextBox.dock Dock.Bottom TextBox.text "Hello About" ]
      ]
    ]
```

Enter fullscreen mode Exit fullscreen mode

This view is as simple as you are guessing it, it has a dock panel that has two children a Menu and a TextBox. The first child is a `viewMenu` function creates a Menu control for us and puts it where we want it (spoilers, you can put these functions somewhere else and make then shareable) the second child is a TextBox control, but depending on which page we are, it will be a different TextBox

-   Home for the "Hello Home"
-   About for the "Hello About"

You can use that to render/call a complete module, that means... Yes multiple pages being rendered! It could look something like this  

```
let view state dispatch =
  DockPanel.create [
    DockPanel.children [ 
      match state.CurrentView with
      | Home -> yield HomeModule.view state.homeState (HomeMsg >> dispatch)
      | About -> yield AboutModule.view state.aboutState (AboutMsg >> dispatch) 
    ]
  ]
```

Enter fullscreen mode Exit fullscreen mode

Whereas you guessed it, HomeModule has the same structure  

```
namespace AvaFunc
module HomeModule =
    type State =
        { (* ... *) }

    let init =
        { (* ... *) }

    type Msg =
        // omitted code
    let update (msg: Msg) (state: State) =
        // omitted code
    let view (state: State) dispatch =
        // omitted code
```

Enter fullscreen mode Exit fullscreen mode

and the same applies to the About module.

At this point, I should mention to you that there's a [TabControl](https://github.com/AvaloniaCommunity/Avalonia.FuncUI/blob/master/src/Avalonia.FuncUI.ControlCatalog/Views/MainView.fs), Control in Avalonia which actually can be used to navigate your whole application instead of something DIY.  
That didn't stop me though since I didn't know this and went DIY we're also trying to learn after all. That's why I said I'm the most average guy you'll ever see so even your most average teammates can archive fairly complex stuff with surprisingly fewer bugs than expected (Even I was surprised)

## [](https://dev.to/tunaxor/creating-web-components-with-fable-lit-2m11#2-reusability)2\. Re-usability

I'm not an expert in MVU so perhaps there's a way to do it better but you should be able to some of your view  
functions, you can take a look at this [module](https://github.com/AngelMunoz/AvaFunc/blob/d4e706b671c398ce5387d7959a00628b800bbbe3/AvaFunc.App/SharedViews.fs) and see it's being used [here](https://github.com/AngelMunoz/AvaFunc/blob/master/AvaFunc.App/QuickNotes.fs#L203) and [here](https://github.com/AngelMunoz/AvaFunc/blob/master/AvaFunc.App/QuickNoteDetail.fs#L101) so... if the most average guy could find a way to  
share some code, there could be a better way to do it and it most certainly will fit the architecture

## [](https://dev.to/tunaxor/creating-web-components-with-fable-lit-2m11#3-crossplatform-electron-replacement)3\. Cross-platform Electron Replacement

This is a very important point for me, when I want to do things for myself I don't want to have a lot of  
resources being consumed by a note-taking app, I used to do my personal stuff with Javascript + UWP  
I have a couple of posts on that but... sometimes it seems Microsoft is looking over your happiness because decided to remove Javascript + UWP in VS2019.

Back to square 0... Avalonia does run in Linux/MacOS/Windows thanks to .net core  
and thanks to the [Avalonia.FuncUI](https://github.com/AvaloniaCommunity/Avalonia.FuncUI) project I can use F#  
in a simple way that feels really good, I've always wanted todo desktop software with F# but I felt that you always had to do  
some workaround or magic stuff to get into the official .net Microsoft GUI solutions like WPF which were still Windows only.

With Avalonia and Avalonia.FuncUI this is not the case you are writing F#, you can add F#/C# .netstandard Libraries  
and all are at the reach of two simple commands  

```
dotnet new --install JaggerJo.Avalonia.FuncUI.Templates
dotnet new funcUI.full -o MyCrossPlatformApp
```

Enter fullscreen mode Exit fullscreen mode

I see myself writing more F# from now on, in some sense F# is pretty similar to Javascript so if you like things like  
React or Functional Programming in Javascript, you will feel at home using F#.

Avalonia itself is an interesting project, it's not handled by Microsoft it's just something that was born from the community for the community.

Now if you head to the Avalonia [Website](https://avaloniaui.net/) and then head to the docs you'll see it's pretty empty  
that might get you to consider if this is actually usable, and here's the [official word](https://avaloniaui.net/blog/#production-ready) of that  
there are still a couple of things missing out there like tray menu for windows, some fancy controls but  
I think you can start working on some really cool stuff here it's powered by .net core so anything that is a .net standard library should be at your fingertips

### [](https://dev.to/tunaxor/creating-web-components-with-fable-lit-2m11#bonus-styling)Bonus: Styling

You can do styling in somewhat a css like style take the following example  

```
<Styles
    xmlns="https://github.com/avaloniaui"
    xmlns:x="http://schemas.microsoft.com/winfx/2006/xaml">
    <Style Selector=".detailbtn /template/ ContentPresenter">
        <Setter Property="Background" Value="#1da1db"/>
        <Setter Property="BorderBrush" Value="#a4d9f0"/>
        <Setter Property="Height" Value="28"/>
    </Style>
    <Style Selector=".detailbtn:pointerover /template/ ContentPresenter">
        <Setter Property="Background" Value="#116083"/>
    </Style>
    <Style Selector=".tobottom /template/ ContentPresenter">
        <Setter Property="VerticalAlignment" Value="Bottom"/>
    </Style>
</Styles>
```

Enter fullscreen mode Exit fullscreen mode

those styles are applied to the following button (using the classes)  

```
Button.create
  [ Button.row 2
    Button.column 0
    Button.content "📃"
    Button.classes [ "detailbtn"; "tobottom" ]
    Button.onClick (fun _ -> dispatch (NavigateToNote note)) ]
```

Enter fullscreen mode Exit fullscreen mode

So if you have better skills at UI design than I (most certainly you have)  
I bet you will be able to do pretty nice stuff.

## [](https://dev.to/tunaxor/creating-web-components-with-fable-lit-2m11#closing-thoughts)Closing Thoughts

Avalonia + F# is A-W-E-S-O-M-E  
when I saw the project I just went into and do random stuff trying to see if it would work, then I tried stuff that I'm used to in web applications, create a list of items then click on it and see the details of it, navigation between pages, CRUD like operations and it might not seem like a "Big App" but I tested basic concepts that I've used to build "Big Apps" at work, there's also input validation in the project, I just check for the title being `length > 3` but if you can do that, then you can do the wildest validation you need.  
Some projects when you test these things, they just don't feel right or feel clunky, I don't feel that this time.

I can't want to see what people write with this :)  
Please feel free to share your thoughts below