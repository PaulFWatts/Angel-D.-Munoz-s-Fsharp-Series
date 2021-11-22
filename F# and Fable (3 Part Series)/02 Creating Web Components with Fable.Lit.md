And We're back! with more F# goodness!

Last time we [saw how to write components with lit-html and F#](https://blog.tunaxor.me/blog/2021-08-28-using-lit-html-with-fsharp.html) that's good! but apart from having

-   A different way to write applications
-   easy library/web component interop without a lot of bindings
-   Native HTML DOM elements and web standards

We're still using and consuming these components ourselves. This time we'll take a step back and think about leveraging the safety and conciseness of F# at compile time to produce web components that can be used in any modern browser and in almost any modern framework.

> The source code for this post is here: [https://github.com/AngelMunoz/fs-components](https://github.com/AngelMunoz/fs-components)

## [](https://dev.to/tunaxor/creating-web-components-with-fable-lit-2m11#prelude)Prelude

When you talk about web components you need to speak about a number o things...

-   attributes & properties
-   behavior
-   styling

and also remember that you are in a javascript environment, your runtime is not F# nor .NET you have to program defensively which you might already be used to do it because F# teaches you a few things here and there, specially around immutability and when to expect empty/non-existent values.

### [](https://dev.to/tunaxor/creating-web-components-with-fable-lit-2m11#attributes-ltgt-properties)Attributes <> Properties

Attributes are strings and `Attributes <> Properties` which might be confusing at the beginning but it kind of makes sense when you see it. Attributes are part of the HTML markup, it allows the author of the HTML markup to provide some information for the element in case it's needed, but it's only for the HTML markup, hence why it's a string. to ilustrate the point let's see an example  

```
<div last-name="my-name" age="10"></div>

<script>
  const div = document.querySelector("[name=last-name]");
  div.lastName = "I'm not the attribute!";
  div.age = 60;
  console.log(div["last-name"], div.lastName);
  console.log(div.getAttribute("last-name"));
  console.log(div.age, div.getAttribute("age"));
</script>
```

Enter fullscreen mode Exit fullscreen mode

If you inspect the div in the dev-tools you'll see that attributes are stored differently than properties.

Having that said, don't expect that changing an attribute will change a property!

### [](https://dev.to/tunaxor/creating-web-components-with-fable-lit-2m11#behavior)Behavior

Components by themselves most of the time have a particular use case, they don't exist for the sake of existing, if a component has multiple responsabilities it is expected that each of these responsabilities could be handled by another component inside, components are modular they can be used by themselves or conjunction with other components.  

```
<tabs-host>
  <tabs-nav>
    <tab-item content="videos">
      Videos <my-icon slot="tab-icon" name="camera"></my-icon>
    </tab-item>
    <tab-item content="books">
      Books <my-icon slot="tab-icon" name="library"></my-icon>
    </tab-item>
    <tab-item content="about">
      About <my-icon slot="tab-icon" name="info"></my-icon>
    </tab-item>
  </tabs-nav>
  <tab-content name="videos">...</tab-content>
  <tab-content name="books">...</tab-content>
  <tab-content name="about">...</tab-content>
</tabs-host>
```

Enter fullscreen mode Exit fullscreen mode

Think of that example as if it was a web page in a website, yes it has to show videos, books, and an about section, but it does so by delegating part of the behavior to different parts of itself the component may even expose parts of itself to be filled by the user, like the `tab-icon` slot where the component author let's the consumer know that they can replace completely that part of the component with their own content.

It is also worth noting that every web component is backed by a javascript class which inherits from `HTMLElement` so your component is actually a first class citizen in the web.

### [](https://dev.to/tunaxor/creating-web-components-with-fable-lit-2m11#styling)Styling

I think this might be the hardest part because style encapsulation in web components is aggresive which as an author is fine because you have defined how your component should look like and no-one will betray your vision ever! ...or so you think because in real life components should be extensible enough for consumer so they can adapt them in their own design systems without making the component feel foreign to the application.

For this we have a couple of tools that allow us to provide sane defaults yet allowing consumers to customize things.

-   [shadow dom](https://developer.mozilla.org/en-US/docs/Web/Web_Components/Using_shadow_DOM)
-   [css variables](https://developer.mozilla.org/en-US/docs/Web/CSS/Using_CSS_custom_properties)
-   [parts](https://developer.mozilla.org/en-US/docs/Web/CSS/::part)
-   [constructable stylesheets](https://developers.google.com/web/updates/2019/02/constructable-stylesheets)
-   [css modules](https://css-tricks.com/css-modules-the-native-ones/)

the last one is still getting into browsers but it should eventually make it there, constructable stylesheets along with css modules will allow you to write styles and use them as you would normally do it, right now you either need to have a complicated setup, import them inside a style tag of your component, or do inline styles which are not great options for performance thankfully chromium browsers implement them already and the feature is pollyfillable so you will see how we make it work.

Hopefully that prelude will help you a little bit to get the idea of what we're looking for and some of the things we need to have in mind.

## [](https://dev.to/tunaxor/creating-web-components-with-fable-lit-2m11#messagefs)Message.fs

`Message.fs` is our main piece for the day it is somewhat a copy of [bulma's message component](https://bulma.io/documentation/components/message/), the only extra thing is that we're making it in a single, reusable, with defined behavior component for ease of use.  

```
[<RequireQualifiedAccess>]
module Message

open Fable.Core
open Lit
open Haunted
open Browser.Types
open ShadowStyles
open ShadowStyles.Types
open Types
// import the "global" stylesheet
[<ImportDefault("./fs-components.css")>]
let private fsComponentsStyles: string = jsNative

// impor the particular stylesheet for this component
[<ImportDefault("./fs-message.css")>]
let private fsmessageClases: string = jsNative

// from the string contents create a constructable stylesheet
// with the help of ShadowStyles
let private styles =
    [ CSSStyleSheet.FromString(fsmessageClases)
      CSSStyleSheet.FromString(fsComponentsStyles) ]
```

Enter fullscreen mode Exit fullscreen mode

When it comes to css modules thankfully tools like [vite](https://vitejs.dev/) or \[webpack\] allow us to import the content of the files without making a fuzz about it. the styles if you inspect their content isn't a really big deal it's a common css that defines our color scheme in the root of the website.  

```
/* fs-components.css */
:root {
  --primary-color: #00d1b2;
  --primary-color-light: #c0fff6;
  --primary-color-dark: #00927d;
  /* more colors */
}
```

Enter fullscreen mode Exit fullscreen mode

```
/* fs-message.css */
/* notice we're not using a class here */
article {
  display: flex;
  flex-direction: column;
  border-radius: 50px;
  padding: 0.1em;
}

.message {
  /* define new variables under the scope of .message */
  --header-bg: var(--dark-color);
  --header-color: var(--white-color);
  --body-bg: #dbdbdb;
  --body-color: var(--black-color);
}

.message-body {
  /* override the variables if necessary */
  background-color: var(--body-bg);
  color: var(--body-color);
  /* these styles will not be changed by anyone outside our component */
  border-bottom-left-radius: 4px;
  border-bottom-right-radius: 4px;
  padding: 0 1em;
}

.message-header {
  background-color: var(--header-bg);
  color: var(--header-color);
  border-top-left-radius: 4px;
  border-top-right-radius: 4px;
  padding: 0 1em;
  display: flex;
  justify-content: space-between;
  align-items: center;
  height: 2em;
}

/* override the styles when a particular selector is active */

.message.is-primary .message-header {
  --header-bg: var(--primary-color);
}
.message.is-primary .message-body {
  --body-bg: var(--primary-color-light);
  --body-color: var(--primary-color-dark);
}
```

Enter fullscreen mode Exit fullscreen mode

in this case we're leveraging CSS custom properties to allow consumers to define their own color scheme by overriding these variables themselves.  

```
[<AllowNullLiteral>]
type FsMessageElement =
    inherit HTMLElement

    abstract kind : Kind option with get, set
    abstract header : string option with get, set
```

Enter fullscreen mode Exit fullscreen mode

This is an importan aspect to understand about web components, even if you are defining a function, you're still using a class underneath, and your users might delete properties or set them to `null` or `undefined` I would advice you to always mark these properties as optional to ensure you are handling nullability concerns and not just fail at runtime.  

```
// here's our function
// the host passed is actually the instance of your component at runtime
// that's why defined that interface
let private Message (host: FsMessageElement) =
    // by passing an empty array at the end, we ansure this hook
    // only runs once which is useful for initializing things
    Haunted.useEffect (
        (fun _ ->
            // Initialize properties you want to have a default value
            // to add to the confusion, Haunted will track
            // attributes as properties if you put them
            /// inside the `observedAttributes` array when defining
            // the component
            host.kind <- host.kind |> Option.orElse (Kind.Default |> Some)),
        [||]
    )

    // add our styles to our component host
    ShadowStyles.adoptStyleSheets (host, styles)

    let messageClases =
        let kind = defaultArg host.kind Default
        // we're using classMap underneath so
        // if we can pass a tuple of a string and a boolean
        // to ensure a class gets applied or not
        seq {
            "message", true

            match kind with
            | Primary -> "is-primary", true
            | Link -> "is-link", true
            | Info -> "is-info", true
            | Success -> "is-success", true
            | Warning -> "is-warning", true
            | Danger -> "is-danger", true
            | Default -> "", false
        }
    // define an event handler
    let tryCloseMessage _ =
        // composed = true allows the event go through the shadow DOM
        let evt =
            Haunted.createEvent (
                "fs-close-message",
                {| composed = true
                   bubbles = true
                   cancelable = true |}
            )
        // dispatch an event so the parents of this component
        // know something happened inside and they should do
        // something about it
        host |> dispatchEvent evt

    let header () =
        // hide the header if it wasn't set
        if Option.isSome host.header then
            html
                $"""
                <div part="message-header" class="message-header">
                    <p>{host.header}</p>
                    <slot name="close-button">
                        <button class="delete" aria-label="delete" @click={tryCloseMessage}>&times;</button>
                    </slot>
                </div>
                """
        else
            Lit.nothing

    html
        // add our dynamic classes
        // also notice the parts here and in the header
        $"""
        <article class={Lit.classes messageClases}>
            {header ()}
            <div part="message-body" class="message-body">
            <slot></slot>
            </div>
        </article>
        """
```

Enter fullscreen mode Exit fullscreen mode

It's a lengthy function but most of it is really about setting up styles we could definetely factor those parts out. Speaking of parts, there are a couple of extensibility features here:

-   slots
-   part's

**slots** are like a window for your component to the external world, when you put contents into a slot these contents are part of the _Light DOM_ or what we know as the normal DOM, they are affected by consumer styles and as authors we don't have controls on them, we are actually allowing the consumers to put content as they see fit.

**parts** are similar windows, but these windows only allow styles to be changed rather than the content. so if the consumer wants they will be able to re-stylize those aspects if they want but that's an area I have not gone deeply and I can't share experences on it.

Lastly, let's make a single public function in this module `register`  

```
// ensure we the reference to the function, otherwise fable might pass a curried function
// and not the function reference itself and that doesn't work
[<Emit("Message")>]
let private MessageRef: obj = jsNative

let register () =
    // for your own apps it's fine if you define the component automatically
    // but for libraries it's best to let the user decide if they want to use
    // the component or not.
    defineComponent
        "fs-message"
        (Haunted.Component(MessageRef, {| observedAttributes = [| "kind"; "header"; "isOpen" |] |}))
```

Enter fullscreen mode Exit fullscreen mode

Haunted provides a couple of settings out of the box, like `observedAttributes` which allow Haunted to observe said attributes and set them as properties in the host which might be confusing if you don't know how that works.

There's also `useShadowDOM` which can be set to false, but when you do that you have to give up things like `Slots`, `Parts`, `Style Encapsulation` (which is arguably the reason you'd like to do it)

If you are authoring a website for yourself/company then feel free to skip the shadow DOM since you will be in control of the application and styling, otherwise please try to use the shadow DOM.

If you want to take a better look at that check the stackblitz samples!

## [](https://dev.to/tunaxor/creating-web-components-with-fable-lit-2m11#distribution)Distribution

I'm not a webpack expert so I'll leave it out of the picture for now, the repository uses [vite](https://vitejs.dev/) which is a dev tool for javascript modules (ESModules) unbundled development and for build and distribution uses rollup from the vue team, [snowpack](https://www.snowpack.dev/) does does a similar job.

The only particular reason I chose vite for this example is that it allows you to import the raw css strings from the modules which then I used to create the constructable stylesheets. It is very likely that I'll update the Fable.Lit.Templates to vite.

Our vite config is pretty simple  

```
const { defineConfig } = require("vite");
const path = require("path");

module.exports = defineConfig({
  build: {
    // use the lib options
    // otherwise it will bundle the scripts
    // with the index file as if it was a webapp
    lib: {
      entry: path.resolve(__dirname, "src/Main.js"),
      name: "fs-components",
      fileName: (format) => `fs-components.${format}.js`,
      formats: ["es", "umd"],
    },
    rollupOptions: {
      treeshake: true,
    },
  },
});
```

Enter fullscreen mode Exit fullscreen mode

We will tree-shake our bundle and export two formats `es` and `umd`, umd is like the general catch all, but modern tools work better with ESModules you may wonder why do we have a `Main.js` file and a `Library.fs.js` file.

For development purposes something like the following is completely acceptable  

```
<script type="module">
  import { registerAll } from "https://unpkg.com/fsharp-components?module";
  registerAll();
  // we're free to use our components now
</script>
```

Enter fullscreen mode Exit fullscreen mode

that's why in the `Library.fs` file we defined a `registerAll` function, the more components we create, the more registrations we will do inside that function. But... Imagine our library had twenty or forty even fifty components... that would make it probably impractical for a consumer who only requires two or perhaps ten of our components, in those cases we re-export via the `Main.js` file all of the registrations with different names to prevent name collisions. By using named exports we are also allowing the consumers to tree-shake their builds themselves and prevent heavy bundles.

for `npm` we just need to login in the npm registry `npm adduser` and then `npm publish` it will take the data from our `package.json` and publish the next version.

## [](https://dev.to/tunaxor/creating-web-components-with-fable-lit-2m11#conclusion)Conclusion

Feel free to try this sample component on whatever web setup you may have! be it in Node, Ruby, F#, Vue, Angular, Razor Views, you name it it's a platform feature so it should work almost everywhere!

Try [Lit.Fable](https://github.com/alfonsogarciacaro/Fable.Lit) today!

> Please note that I'm using `Fable.Haunted` here to enable web components, we're still trying to iron out the design for a lit based web component or even a vanilla web component backed up by similar concepts we have here. it would be great if you could chime up in the discussion in the Fable.Lit repository and help us bridge F# and the modern and standard web