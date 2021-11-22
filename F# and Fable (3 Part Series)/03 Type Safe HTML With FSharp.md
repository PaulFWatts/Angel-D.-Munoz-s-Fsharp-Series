And the more F# the better! We're back with yet another way of doing Web Standards HTML...

While I've [talked about sutil](https://dev.to/tunaxor/taking-advantage-of-the-platform-with-sutil-and-web-components-5hm0) in the past I was slightly wrong saying that [sutil](https://sutil.dev/) was using Svelte under the hood, it turns out that **_Sutil_** is actually a pure F# implementation of a web framework! That means that yes! there's no VDOM, there's no dependency other than the FSharp.Core\* library.

> \* The F# Core library get's tree-shaken by tools like webpack/vite/snowpack, feel confident that the end build is just what you use, nothing more nothing less.

### [](https://dev.to/tunaxor/creating-web-components-with-fable-lit-2m11#wait-what-fablelit-sutil-feliz)Wait what? Fable.Lit, Sutil, Feliz?

> OH My GOD we're starting to get into the endless javascript frontend frameworks now in F# we don't need a thousand frameworks in F#

Sadly we don't have the numbers in the F# community to have an endless framework showdown as the JS community has. Alternatives mean innovation, alternatives mean freedom to chose, I would like to kindly remind you that the choice is yours, stick to what you feel confident using, there's nothing absolutely wrong of selecting a tool and work with it.

## [](https://dev.to/tunaxor/creating-web-components-with-fable-lit-2m11#fablelit-sutil)Fable.Lit + Sutil

One of the advantages of using Web Standards API's like the browser's DOM is that you get to play with the rest of the javascript ecoystem without major changes.

-   Fable.Lit

Focuses on **Rendering HTML** to the browser with amazing performance and interoperability, the string you use to write the templates for Lit are basically just HTML, this includes web component libraries like [shoelace](https://shoelace.style/), or [ionic framework](https://ionicframework.com/docs/components)  

```
    <button @click={() => console.log('click')}>
        Click me
    </button>

    <ion-button @click={() => console.log('click')}>
        Click me
    </ion-button>

    <sl-button @click={() => console.log('click')}>
        Click me
    <sl-button>
```

Enter fullscreen mode Exit fullscreen mode

-   Sutil

Focuses on **_Type Safe HTML_** leveraging observables, inspired by [svelte](https://svelte.dev/) and powered by \[feliz.engine\], Sutil works with DOM Nodes rather than having an intermediary (like Lit's string templates) meaning that Sutil can be used in cases where complete frameworks are hard to use, like enhancing an HTML node that was rendered from the server. That also means it can also use Web Components just as Lit does.  

```
    Html.button [
        onClick (fun _ -> printfn "click") []
    ]

    Html.custom("ion-button", [
        onClick (fun _ -> printfn "click") []
    ])

    Html.custom("sl-button", [
        onClick (fun _ -> printfn "click") []
    ])
```

Enter fullscreen mode Exit fullscreen mode

Having that said, let's talk about how can you use them together to have the best of both worlds, amazing interoperability and type safety.

For the next parts I will be using this repository

[https://github.com/AngelMunoz/ItsHtml](https://github.com/AngelMunoz/ItsHtml)

In previous posts I talked about how you can create web components with Fable.Lit + Haunted and I even made a sample library that can be accessed from the browser with a script tag or from the npm package ecosystem we will use the same library once again just to remind you that when I say it can be used anywhere HTML is used... I mean it

Following our last post conventions, let's start with our `Main.fs` file  

```
module Main

open Fable.Core.JsInterop
open Pages
open Components

importSideEffects "./styles.css"
// import our fsharp-components library and register the elements
let registerAll: unit -> unit = importMember "fsharp-components"

registerAll ()

// register your custom elements here
Home.register ()
Counter.register ()
Icons.register ()

// this is not a custom element but rather
// a function that mounts a sutil node in an HTML node
App.start ()
```

Enter fullscreen mode Exit fullscreen mode

From from past posts we know that we can register our web components in the entry point of our application (or anywhere, but to be concise we do it there) to make them available everywhere. that inludes components comming from other libraries like `fsharp-components`, before moving to `App.fs` I'd like to check `Bindings.fs`.

To make **Type Safe HTML** from unknown (to sutil) elements we can make a binding layer which is arguably unnecesary, but it makes it more pleasant to work with in Sutil.  

```
[<AutoOpen>]
module Bindings

open Sutil
open Types

module FsAttrs =

    let inline isOpen (isOpen: bool) =
        Attr.custom ("is-open", (if isOpen then "true" else ""))

    (* Other declarations omited for brevity *)

    // request a type safe positon
    let inline fsOffCanvasPosition (position: OffcanvasPosition) =
        let pos =
            match position with
            | Left -> "left"
            | Right -> "right"
        // do the correct match and assign it to the attribute
        Attr.custom ("position", pos)

    let inline fsKind (value: Kind) =
        let kind =
            match value with
            | Primary -> "primary"
            | Info -> "info"
            | Link -> "link"
            | Success -> "success"
            | Warning -> "warning"
            | Danger -> "danger"
            | Default -> ""

        Attr.custom ("kind", kind)

module Fs =
    // to prevent Html.custom clutter everywhere, you can declare functions
    // that match the Feliz API, mark them inline to
    // remove un-needed code in the compiled code
    let inline fsOffCanvas nodes = Html.custom ("fs-off-canvas", nodes)

    let inline fsMessage nodes = Html.custom ("fs-message", nodes)
    let inline fsTabHost nodes = Html.custom ("fs-tab-host", nodes)
    let inline fsTabItem nodes = Html.custom ("fs-tab-item", nodes)
```

Enter fullscreen mode Exit fullscreen mode

As you can see writing bindings is not that complex, and as I said, arguably unnecesary since they are just for comodity, you can use both `Attr.custom` and `Html.custom` when you need it.

Moving to `App.fs` you'll see there are more things  

```
[<RequireQualifiedAccess>]
module App

// a template for the fs-tab-item element
let private tabItemTemplate item =
    fsTabItem [
        fsTabItemLabel item.label
        fsTabItemTabName item.id
        fsKind item.kind
    ]

// switch the page depending on the content
let private getPage page =
    match page with
    | Page.Home -> Html.custom ("flit-home", [])
    | Page.Notes -> Notes()



let private onTabSelected page (ev: CustomEvent<{| tabName: string |}>) =
    let tabName =
        ev.detail
        |> Option.map (fun i -> i.tabName)
        |> Option.defaultValue "home"

    match tabName with
    | "notes" -> page <~ Page.Notes
    | "home"
    | _ -> page <~ Page.Home

let private offCanvasItemTemplate page item =

    let onSelected newPage _ = page <~ newPage

    let getPage id =
        match id with
        | "home" -> Page.Home
        | "notes" -> Page.Notes
        | _ -> Page.Home

    Html.li [
        onClick (onSelected (getPage item.id)) []
        Html.a [ Html.text item.label ]
    ]

let private menuStyle =
    rule
        "app-icon[name=menu]"
        [ Css.floatRight
          Css.cursorPointer
          Css.height (32)
          Css.width (32) ]

let private app () =
    // dynamic parts of your site are tracked with
    // stores in Sutil
    let page = Store.make Page.Home
    let menuOpen = Store.make false

    let entries =
        Store.make (
            [ { label = "Home"
                id = "home"
                kind = Link }
              { label = "Notes"
                id = "notes"
                kind = Link } ]
        )

    Html.app [
        disposeOnUnmount [ page; entries; menuOpen ]
        Html.article [
            // use our type safe API we wrote on Bindings.fs
            fsOffCanvas [
                on "fs-close-off-canvas" (fun _ -> menuOpen <~ false) []
                closable true
                Html.h3 [
                    Attr.slot "header-text"
                    Html.text "Type Safe components and seamless interop!"
                ]
                // render a list of menu entries
                Bind.each (entries, offCanvasItemTemplate page)
                Bind.attr ("isOpen", menuOpen)
            ]
            Html.custom (
                "app-icon",
                [ Attr.name $"{Menu}"
                  onClick (fun _ -> menuOpen <~ true) [] ]
            )
            fsTabHost [
                fsKind Link
                // react to events that web components emit
                onCustomEvent "on-fs-tab-selected" (onTabSelected page) []
                Html.nav [
                    Attr.slot "tabs"
                    // render a list of menu entries
                    Bind.each (entries, tabItemTemplate)
                ]
                // bind the content of the website depending on what page we are
                Bind.el page getPage
            ]
        ]
    ]
    |> withStyle [
        rule "a" [ Css.cursorPointer ]
        menuStyle
       ]

let start () =
    // mount our sutil app in a DOM element with that id
    Program.mountElement "sutil-app" (app ())

```

Enter fullscreen mode Exit fullscreen mode

So far, we have done the following:

-   Add a Binding Layer
-   Use Sutil's API to render our website

Our binding Layer adds type safety to unknown elements but there will be some cases where the amount of unknown/external code to migrate/work with is a lot or you simply don't have the time to start binding everything here and there. That's a Fable.Lit is not only usefull for big apps, it can be used in segments of an existing App to leverage working with third party libraries.

Let's check `Components/Counter.fs`, while fs-message has bindings in our `Bindings.fs` file, I'd like you to think that this could be a library with 30+ components (Like FAST, Shoelace, and ionic libraries tend to be) you might not want to spend the time writing bindings, but rather writing your application and that's fine, the way to do that is isolate those parts creating a web componentthat you can easily import somewhere else in your application (like counter, like the home page, etc.)  

```
[<RequireQualifiedAccess>]
module Components.Counter

open Browser.Types
open Lit
open Haunted

let private counter (props: {| initial: int option |}) =
    let count, setCount =
        Haunted.useState (defaultArg props.initial 0)

    // no bindings required to start using our fs-message!
    let messageTpl =
        match count with
        | count when count > 100 ->
            html
                $"""
                <fs-message header="Danger high count!" kind="danger" is-open>
                    <p>I don't want to be that guy but... that's a high count!</p>
                </fs-message>
                """
        | count when count < 0 ->
            html
                $"""
                <fs-message header="Warning low count!" kind="warning" is-open>
                    <p>I don't want to be that guy but... that's a low count!</p>
                </fs-message>
                """
        | _ -> Lit.nothing


    html
        $"""
        <p>Home: {count}</p>
        <button @click={fun _ -> setCount (count + 1)}>Increment</button>
        <button @click={fun _ -> setCount (count - 1)}>Decrement</button>
        <button @click={fun _ -> setCount (defaultArg props.initial 0)}>Reset</button>

        <!-- no special binding for the event either -->
        <div @fs-close-message={fun (e: Event) -> (e.target :?> HTMLElement).remove ()}>
            {messageTpl}
        </div>
        """

let register () =
    defineComponent "flit-counter" (Haunted.Component counter)
```

Enter fullscreen mode Exit fullscreen mode

While there are tools like [html2feliz](https://thisfunctionaltom.github.io/Html2Feliz/) which help you convert parts of your code this process can be tedious if you have a large existing application, in those cases it's simply easier to just copy/paste the existing HTML without changes and focus on making the logic work rather than trying to provide type safety from day 0. Type safety can be an incremental effort from you and your team.

Lastly let's check `Notes.fs`, this file has an [elmish](https://elmish.github.io/elmish/) implementation, to handle a form submission. I'll skip the whole elmish implementation and focus on the view.  

```

let private noteTemplate (note: Note) = Html.li $"Id: {note.Id} - {note.Title}"

let Notes () =
    // create the observable state and the dispatch function
    let state, dispatch =
        Store.makeElmishSimple init update ignore ()

    // get an observable from the notes inside the state
    let notes = state .> (fun s -> s.Notes)


    let onTitleChange (evt: Event) =
        SetTitle (evt.target :?> HTMLInputElement).value
        |> dispatch

    let onBodyChange (evt: Event) =
        SetBody (evt.target :?> HTMLInputElement).value
        |> dispatch

    Html.div [
        // It is important to signal Sutil to dispose Stores once an element is removed from the DOM
        disposeOnUnmount [ state ]
        // use the type safety API + the Elmish you know and love
        Html.form [
            on "submit" (fun _ -> dispatch Save) [ PreventDefault ]
            Html.input [
                Attr.typeText
                Attr.name "title"
                Attr.placeholder "Title"
                onKeyDown onTitleChange []
                on "blur" onTitleChange []
            ]
            Html.input [
                Attr.typeText
                Attr.name "body"
                Attr.placeholder "Body"
                onKeyDown onBodyChange []
                on "blur" onBodyChange []
            ]
            Html.button [
                Attr.typeSubmit
                Html.text "Add"
            ]
        ]
        Html.ul [
            // bind the notes and add type safe css transitions!
            Bind.each (notes, noteTemplate, [ InOut fade ])
        ]
    ]
```

Enter fullscreen mode Exit fullscreen mode

One aspect that I'm not covering here but Sutil also covers pretty well is [styling](https://sutil.dev/#documentation-styling), not only you get type safe HTML, you also get type-safe, scoped CSS.

## [](https://dev.to/tunaxor/creating-web-components-with-fable-lit-2m11#closing-thoughts)Closing thoughts

The biggest downside I can see here is using two very very similar tools for the same purpose but, since neither uses a virtual DOM there are no performance penalties, since they use HTML and DOM API's they don't have weird rendering interop issues or anything like that.

Both tools have their use case very clear.

-   If you prefer to write HTML due to tooling and other things stick to **_Fable.Lit_**.
-   If you want type safety and native DOM elements, stick to **_Sutil_**.
-   If you need both Great, you can use them both!

Type Safe interoperable HTML!? Count me in!