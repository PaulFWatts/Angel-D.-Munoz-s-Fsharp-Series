Hello, there!

Today I'll bring you some F# goodness this time for the frontend developers who may have scrapped F# due to the lack of frontend options.

## [](https://dev.to/tunaxor/series/14356#lit-html)Lit HTML

Lit HTML is a rendering library that doesn't use Virtual DOM and uses javascript tagged template literals to render the HTML you know and love. Also it's highly worth noting that the HTML you will render is standard HTML meaning that you are free to use any kind of [web components](https://developer.mozilla.org/en-US/docs/Web/Web_Components) in the market like [ionic framework](https://ionicframework.com/), [shoelace](https://shoelace.style/) or Microsoft's [FAST](https://www.fast.design/) elements [among others](https://open-wc.org/guides/community/component-libraries/).

## [](https://dev.to/tunaxor/series/14356#lets-begin)Let's begin

As life is with frontend development you will need the wollowing requirements

-   A node LTS distribution (I'm using v14.17.1 but any LTS or current release should be good)
-   A dotnet SDK (preferably net5.0+)

That being said you should be able to install the [dotnet templates](https://www.nuget.org/packages/Fable.Lit.Templates) for Fable + Lit  

```
dotnet new --install Fable.Lit.Templates
```

Enter fullscreen mode Exit fullscreen mode

After running that you will see something like this  

```
Success: Fable.Lit.Templates::1.0.0-beta-003 installed the following templates:
Template Name        Short Name         Language
-------------------  -----------------  --------
Elmish.Lit           elmish-lit         F#
Feliz.Lit + Haunted  feliz-lit-haunted  F#
Lit + Haunted        lit-html-haunted   F#
```

Enter fullscreen mode Exit fullscreen mode

> Brief digression, [Haunted](https://hauntedhooks.netlify.app/) is a library that allows you to produce custom elements and also use hooks like you would if you were using [react](https://reactjs.org/).

let's get started with the simplest one, type the following

`dotnet new elmish-lit -o elmish-sample`

This will create a directory with an elmish based counter application.

> VSCode is highly encouraged to use as the editor since it has support for both F# and interpolated strings highlight as html, the workspace includes the recommended extensions.

once you have opened that directory with your editor go ahead and type the following

> For performance and disk friendliness it is recommended that you use [pnpm](https://pnpm.io/) although, it's not needed.

-   `npm install`

the postinstall step will install the .NET tools (fable) required for the project to run

-   `npm start`

This will build your F# sources and start your project in localhost:8080

> Note: these templates are in beta stage please kindly report any bugs you might find along the way

If everything goes well you should see this in your browser

[![hello world with a counter](https://res.cloudinary.com/practicaldev/image/fetch/s--6Zt1FXJV--/c_limit%2Cf_auto%2Cfl_progressive%2Cq_auto%2Cw_880/https://i.imgur.com/Iv0wYO8.png)](https://res.cloudinary.com/practicaldev/image/fetch/s--6Zt1FXJV--/c_limit%2Cf_auto%2Cfl_progressive%2Cq_auto%2Cw_880/https://i.imgur.com/Iv0wYO8.png)

Let's check how that works, open the `src/Main.fs` file and go little by little  

```
module Main

// let's import our styles
Fable.Core.JsInterop.importSideEffects "./styles.css"

open Elmish

open Lit
open Lit.Elmish

// define what is the shape of our state for our elmish program
type State = { counter: int; name: string }

// in the Msg add the kind of events we will handle
type Msg =
    | Increment
    | Decrement
    | Reset

let private init _ = { counter = 0; name = "World" }

let private update msg state =
    // depending on what message we receive we will return our updated state
    match msg with
    | Increment ->
        { state with
              counter = state.counter + 1 }
    | Decrement ->
        { state with
              counter = state.counter - 1 }
    | Reset -> init state

// counter here is a simple function
let private counter
    (props: {| counter: int
               decrement: unit -> unit
               reset: unit -> unit
               increment: unit -> unit |})
    =
    // using lit-html we can create "holes" where
    // we can bind functions and values depending on where we put them
    html
        $"""
        <button @click={fun _ -> props.decrement ()}>-</button>
        <button @click={fun _ -> props.reset ()}>Reset</button>
        <button @click={fun _ -> props.increment ()}>+</button>
        <div>{props.counter}</div>
        """

let view state dispatch =
    let counterEl =
        // let's create our counter and pass the properties in
        counter
            {| counter = state.counter
               // check the dispatch for the functions
               // these will tell elmish what to do when a particular function
               // is called
               decrement = fun _ -> dispatch Decrement
               increment = fun _ -> dispatch Increment
               reset = fun _ -> dispatch Reset |}

    // put our main view with any other content we need
    // also ntoe that counterEl is put inside the hole and lit will
    // also render that
    html
        $"""
        <div>Hello {state.name}!</div>
        {counterEl}
        """

// start your program and call it a day!
Program.mkSimple init update view
|> Program.withLit "elmish-lit"
|> Program.run
```

Enter fullscreen mode Exit fullscreen mode

With propper editor tooling it should look like this

[![code with propper editor support](https://res.cloudinary.com/practicaldev/image/fetch/s--da86TgcI--/c_limit%2Cf_auto%2Cfl_progressive%2Cq_auto%2Cw_880/https://i.imgur.com/o6C0Tyt.png)](https://res.cloudinary.com/practicaldev/image/fetch/s--da86TgcI--/c_limit%2Cf_auto%2Cfl_progressive%2Cq_auto%2Cw_880/https://i.imgur.com/o6C0Tyt.png)

That should get you started with Fable + Lit, feel free to experiment with these templates and also share with the community what did you do with them!

## [](https://dev.to/tunaxor/series/14356#a-more-complex-example)A more complex example

Let's now check this repository

-   [https://github.dev/AngelMunoz/fable-lit-sample](https://github.dev/AngelMunoz/fable-lit-sample)

This repository is a somewhat simple "RSS feed reader" we will not check every part of it but just some code

> This repository is based on this template
> 
> -   `dotnet new lit-html-haunted -o lit-haunted-sample`

Let's check `src/FeedManaget.fs`, but before we jump in directly I'll mention that the programming model here is more centered about the `component` concept, meaning that a component has behavior/state and is responsible of a section of your UI, components can be as small as a single div or as big as a whole page, the ideal size will depend on the "responsability" of that component, each component can and should have child components that should handle a particular behavior. In F# you can think about each component as a function. Each atribute/property of a component is going to be an argument for that function.

> although each property and attribute (given how haunted works) will end up being a property of an object which will be a single parameter.  

```
[<RequireQualifiedAccess>]
module Pages.FeedManager

open Browser.Types
open Lit
open Haunted
open Types

// a simple function that takes a feed object and renders it inside an ion-label
let private feedTpl feed =
    html
        $"""
         <ion-item>
            <ion-label>{feed.Name}</ion-label>
         </ion-item>
        """
// feedManager is our function which handle
// all of the things related with managing the user's feeds
let private feedManager () =
    // set a state for the current feed item in the form
    let currentFeed, setCurrentFeed =
        Haunted.useState ({ Name = ""; Url = "" })

    // set the state for the feeds the user has
    let feeds, setFeeds = Haunted.useState (Feed.LoadFeeds())

    // handle the events of the feed's name input
    let nameChanged (ev: CustomEvent<{| value: string |}>) =
        // as you can see we are handling CustomEvents
        // these are native browser CustomEvents!
        let name =
            ev.detail
            // since custom events do not always pass details
            // they define the detail as an option<'T>
            // we simply map over the value or set a default in case of detail being None
            |> Option.map (fun detail -> detail.value)
            |> Option.defaultValue ""
        // call our hook to set the name in our feed
        setCurrentFeed { currentFeed with Name = name }

    let urlChanged (ev: CustomEvent<{| value: string |}>) =
        let url =
            ev.detail
            |> Option.map (fun detail -> detail.value)
            |> Option.defaultValue ""

        setCurrentFeed { currentFeed with Url = url }

    let saveFeed _ =
        if currentFeed.Name.Length > 0
           && currentFeed.Url.Length > 0 then
            let feeds = [| yield! feeds; currentFeed |]
            // we update both our state and the local storage
            // (for the next time we load the page)
            setFeeds feeds
            Feed.SaveFeeds feeds

    html
        $"""
        <ion-content>
            <h1>Feed List</h1>
            <section>
                <ion-item>
                    <ion-label>Add Feed</ion-label>
                </ion-item>
                <!--
                    ionic usually exposes properties rather than attributes for the components
                    to use internally, we're using lit-html and we can do one-way binding using
                    `.` before the property name.
                    in the case of events we need to put `@` before the event name
                -->
                <ion-input .debounce="{750}" placeholder="Name" @ionChange={nameChanged}></ion-input>
                <ion-input .debounce="{750}" type="url" placeholder="Url" @ionChange={urlChanged}></ion-input>
                <ion-button @click={saveFeed}>Save</ion-button>
            </section>
            <ion-list>
                {feeds |> Array.map feedTpl}
            </ion-list>
        </ion-content>
        """

let register () =
    // define our component globally so it can be used it anywhere
    defineComponent "x-feed-manager" (Haunted.Component feedManager)
```

Enter fullscreen mode Exit fullscreen mode

If you are asking youself _Why would I register a tag globally_? it's mainly because these tags work as any other browser tag like `div`, `table`, `span` you can use them in any context where you thing they should be used.

let's check our `src/App.fs` file  

```
[<RequireQualifiedAccess>]
module App

open Lit

open Haunted

let private app () =

    html
        $"""
        <ion-app>
            <ion-tabs>
                <!--
                    Here we're letting ionic know that we want to use <x-home></x-home>
                    and <x-feed-manager></x-feed-manager> as the tags for the content of the tabs
                -->
                <ion-tab tab="home" component="x-home"></ion-tab>
                <ion-tab tab="feeds" component="x-feed-manager"></ion-tab>
                <ion-tab-bar slot="bottom">
                    <ion-tab-button tab="home">
                        <ion-icon name="home"></ion-icon>
                        <ion-label>Home</ion-label>
                    </ion-tab-button>
                    <ion-tab-button tab="feeds">
                        <ion-icon name="bookmark"></ion-icon>
                        <ion-label>Feeds</ion-label>
                    </ion-tab-button>
                </ion-tab-bar>
            </ion-tabs>
        </ion-app>
        """

let register () =
    // register the app function as the `flit-app` component in the browser
    defineComponent "flit-app" (Haunted.Component app)
```

Enter fullscreen mode Exit fullscreen mode

once again, we register our component at the end as a global tag rather than just exposing the function and if we take a look at our `src/Main.fs` it looks like this  

```
module Main

open Fable.Core.JsInterop
open Pages

importSideEffects "./styles.css"

// register your custom elements here
FeedManager.register ()
FeedViewer.register ()
Home.register ()
App.register ()
```

Enter fullscreen mode Exit fullscreen mode

we're registering our components and then just freely using those tags where they may be required. Lastly, to make all of this make sense, take a look at the body tag inside `public/index.html`...  

```
<body class="dark">
  <flit-app></flit-app>
  <script type="module" src="/dist/Main.fs.js"></script>
</body>
```

Enter fullscreen mode Exit fullscreen mode

Yup! we're using our tag! the browser simply knows what to do when it sees the tags we define.  
Feel free to check the whole sample and let me know what you think, you can check this [project live here](https://fable-lit-sample.web.app/) and if you need a sample RSS feed you can use `https://blog.tunaxor.me/feed.rss`

## [](https://dev.to/tunaxor/series/14356#server-people)Server people

If you are thinking, what about I write server side code that also exposes tags will they work?  
well as long as you have the corresponding javascript code... Yes! these will run as usual when the page starts! so if you or your company have a custom set of components living in some cdn either private or public, you can even ditch out SPA applications and just render things from the server you can simply develop your components as individual pieces of reusable components and put a script tag in your browser remember that with HTTP 2 + you don't pay any penalty for including multiple small scripts so bundling is not even required.

## [](https://dev.to/tunaxor/series/14356#y-tho)y tho?

If you think I'm asking you to drop every single react project you have right now and migrate to Lit well NO, Why Would You Do That? it doesn't make sense, it would be silly to migrate everything to Lit! this is more about choice, the react ecosystem is nice and has good options and that's fine but as the day of this writing, [it's still](https://custom-elements-everywhere.com/) somewhat annoying to interact with new frameworks and libraries that are developing in the wild. Particularly from big names like Microsoft, Adobe, and even Google, of course they will provide compatibility layers but those layers have performance penalties as well as using a Virtual DOM, being fast enough is not the same of being sanely fast.

Main reasons to try lit-html

-   Performance
-   Browser Standards
-   Full Framework/Library compatibility with web components/custom elements
-   You like to use HTML
-   You want type safety with all of the above

Woah woah, wait didn't you say you **_were using strings_**? how is that type safety?

Well let me tell you that [@alfonsogcnunez](https://twitter.com/alfonsogcnunez) is aware of the people who love type safety!

check this template as well  

```
dotnet new feliz-lit-haunted -o feliz-flavored-lit
```

Enter fullscreen mode Exit fullscreen mode

Inside `src/Home.fs` you'll find something like this  

```
[<RequireQualifiedAccess>]
module Pages.Home

open Lit.Feliz
open Haunted


let private counter (props: {| initial: int option |}) =
  // same hooks as if in lit based templates
  let count, setCount =
    Haunted.useState (props.initial |> Option.defaultValue 0)
  // Type Safe HTML DSL!
  Html.div [
    Html.p $"Home: {count}"
    Html.button [
      Ev.onClick (fun _ -> setCount (count + 1))
      Html.text "Increment"
    ]
    Html.button [
      Ev.onClick (fun _ -> setCount (count - 1))
      Html.text "Decrement"
    ]
    Html.button [
      Ev.onClick (fun _ -> setCount (0))
      Html.text "Reset"
    ]
  ]
  // Yup as simple as that
  |> Feliz.toLit

let register () =
  // Register your tags as well
  defineComponent
    "flit-home"
    // also if you don't want to deal with Shadow DOM and use something
    // like bulma or boostrap you can turn it off :)
    (Haunted.Component(counter, {| useShadowDOM = false |}))
```

Enter fullscreen mode Exit fullscreen mode

Check the [fable.lit](https://github.com/alfonsogarciacaro/Fable.Lit) github repository to see also ways to interact with inter-operate Lit + React within Fable!

Another good reason is that you can produce your own set of custom elements for distribution to your clients or within your organization's teams and they don't even need to know what F# is! Both you and your clients can simply enjoy the safety of F# without even mentioning it

As an example you don't need to know that [FAST](https://www.fast.design/) elements from microsoft and a new set of components from adobe are written in Lit, neither you need to know that [ionic framework](https://ionicframework.com/) is written with Stencil! yet hundreds of thousands (I'd even say millions with ionic numbers in the play) of people use them to either build apps or consume them! actually we just saw an example!

So if you are looking for a broader integration with the ecosystem, and trying to use browser standard means to grow build your applications rather than doing them the **_"React way"_** then jump in!

### [](https://dev.to/tunaxor/series/14356#some-history-behind-this-on-fable-land)Some history behind this on fable land

In [fable](https://fable.io/)\-land we have been using [react](https://reactjs.org/) historically by a few reasons either using [fable-react](https://github.com/fable-compiler/fable-react) or [feliz](https://github.com/Zaid-Ajaj/Feliz) the main one is that react's programming model (i.e. functional like style) is an awesome fit for F#

UI elements as data and functions?  
I'm sold right?

The other reason is that other frameworks have a programming model that isn't as friendly as react for F# (i.e. tons of mutability) these frameworks either need a particular file structure, interact with a particular kind of file and simply becomes a tooling nightmare that to be honest is hard to keep up with. if you didn't know it there have been bindings for things like Vue, Svelte, Snabdom and similar, that's kind of interesting so **_why haven't they been picked up?_**

Part of the F# culture here applies from .NET, people need to run businesses and need to pay bills, so why should they be looking around and playingwith things when there are other needs at hand?

The F# people (and .NET as well) tend to simply settle down for a single solution for most problems.

This has pros/cons as everything from one hand you'll have resources for a specific set of tools and you're very likely to simply follow the road some have paved already, on the other hand if you need something outside that road... well you will need to pave it yourself and sometimes that takes A TON of effort and time which if you run a business and pay bills... it simply doesn't pay.

With that being said! why another tool then? it turns out that Google and other browser vendors had a vision years ago. A single standard way to define something in the browser that runs regardless of the framework the users might be using this vision turned out as what we now know as [web components](https://dev.tobasically%20a%20custom%20html%20tag%20you%20can%20use%20anywhere/) there are several libraries and frameworks to define web components and custom elements. One of those was [polymer](https://polymer-library.polymer-project.org/3.0/docs/devguide/feature-overview) which got rebranded to [lit-element](https://lit-element.polymer-project.org/guide) and later as [lit](https://lit.dev/) but suffice to say that their programming model has been pretty clear from the beginning, use a class to keep state and use html as your UI language.

In the second rebranding, lit-element got split in two separating the web component framework from the rendering engine, this rendering engine is called [lit-html](https://lit.dev/docs/libraries/standalone-templates/) and focuses on using ES2015 tagged literals (functions that take interpolated strings) to render html which recently fable got support for and that changes things a little, the most critical thing is that interpolated strings fit nicely within F# no need for extra tooling or things that we don't know about, returning strings (which technically is not a string is a `TemplateResult`) is not too different from returning `ReactElement` so you can use your usual functional belt to also do UI's in a functional first mode and while you lose some type safety (using the string based templates not the Feliz flavor) you gain access to a vast ecosystem and libraries that otherwise you wouldn't be able to use because [react doesn't play well with web components](https://custom-elements-everywhere.com/).

## [](https://dev.to/tunaxor/series/14356#conclusion)Conclusion

This was a long one! but one that I care about I feel this has more potential than previous attempts to be part of the F# ecosystem and I'll do my best to provide you with samples and documentation related to this :)

And that's it! I'll see you in the next one.