---
aliases: [dotnet]
tags: [fsharp]
---
-----
[Generating HTML in F# - DEV Community 👩‍💻👨‍💻](https://dev.to/tunaxor/generating-html-in-f-4h5b)

## [](https://dev.to/tunaxor/doing-some-io-in-f-4agg#simple-things-in-fsharp)Simple things in FSharp

Hello there, this is the 4th entry in Simple Things F#.

Today we'll be talking about producing HTML strings (useful to create reports, pdfs and things like those). Producing HTML in F# is quite simple to be honest, there are a bunch of libraries that allow that we'll be looking at some I've used at some point

-   [Giraffe.ViewEngine](https://giraffe.wiki/view-engine)
-   [Feliz.ViewEngine](https://github.com/dbrattli/Feliz.ViewEngine)
-   [Scriban](https://github.com/scriban/scriban)

The first two are F# DSL's to build HTML, the last one is an HTML scripting language for .NET (there's also [Giraffe.Razor](https://github.com/giraffe-fsharp/Giraffe.Razor) but I'll skip that one for today)

### [](https://dev.to/tunaxor/doing-some-io-in-f-4agg#giraffe)Giraffe

When it comes to HTML DSL's this is the most "traditional" one since it's not a flavor liked my many these days but that doesn't take away it's usefulness. Giraffe View Engine uses XmlNode as it's building block so you can produce XML as well with Giraffe View Engine! (points for that)

the structure of a node function in Giraffe is as follows

-   `tagName [(* attribute list *)] [(* node list *)]`

for example creating `<div class="my-class"></div>` would be something like this

-   `div [ _class "my-class" ] []`

> attributes are prefixed with an underscore `_` to prevent clashing with reserved words in F#

Let's take a look at a simple page with a header  

```
#r "nuget: Giraffe.ViewEngine"

open Giraffe.ViewEngine


let view =
    html [] [
        head [] [ title [] [ str "Giraffe" ] ]
        body [] [ header [] [ str "Giraffe" ] ]
    ]
let document = RenderView.AsString.htmlDocument view

printfn "%s" document
```

Enter fullscreen mode Exit fullscreen mode

> To Run this, copy this content into a file named `script.fsx` (or whatever name you prefer) and type:
> 
> -   `dotnet fsi run script.fsx`

we assign the result of the html function to view, then we just render the document as a string, we can then follow up and create a file with the IO API's we saw earlier in this series.

> If you're using Saturn/Giraffe as your web framework you don't need to do the manual render they provide helpers that take care of this (as you'll see on the next entry in this series)

you can also create functions to pre-define aspects of your views and override values if you deem it necessary  

```
#r "nuget: Giraffe.ViewEngine"

open Giraffe.ViewEngine


let card attributes = 
    article [ yield! attributes; _class "card is-green"]

let cardFooter attributes =
    footer [ yield! attributes; _class "card-footer is-rounded"]

let cardHeader attributes =
    header [ yield! attributes; _class "card-header no-icons"]

let mySection = 
    div [] [
        card [ _id "my-card" ] [
            cardHeader [] [
                h1 [] [ str "This is my custom card"]
                img [ _src "https://some-image.com"; _class "card-header-image" ]
            ]

            p [] [ str "this is the body of the card" ]

            cardFooter [ _data "my-attr" "extra attributes" ] [
                p [] [ str "This is my footer"]
            ]
        ]
    ]

// note that we used htmlNode this time instead of htmlDocument
let document = RenderView.AsString.htmlNode mySection

printfn "%s" document
```

Enter fullscreen mode Exit fullscreen mode

> To Run this, copy this content into a file named `script.fsx` (or whatever name you prefer) and type:
> 
> -   `dotnet fsi run script.fsx`

so creating new "tags" is basically just creating a new function that accepts attributes and contents.

### [](https://dev.to/tunaxor/doing-some-io-in-f-4agg#feliz)Feliz

The original Feliz DSL was made by [@zaidajaj](https://dev.to/zaidajaj) (he produces AMAZING OSS work for F# you should check his github profile) to use in [Fable](https://fable.io/docs/) applications which are built most of the time on top of React.js so if you take a look at the following content you'll see that `view` is a type of `ReactElement` but don't worry it's just a type, there's no javascript running here it's just a way to make use of a really good DSL (which reduces typing and improves readability) to produce HTML as well in the backend.

> I'm not ultra well versed in composition techniques with Feliz so take the following examples with a grain of salt (you can also check the Elmish book [https://zaid-ajaj.github.io/the-elmish-book/](https://zaid-ajaj.github.io/the-elmish-book/)) for more information about Fable and Feliz  

```
#r "nuget: Feliz.ViewEngine"

open Feliz.ViewEngine

let view = 
    Html.html [
        Html.head [ Html.title "Feliz" ]
        Html.body [
            Html.header [ prop.text "Feliz" ]
        ]
    ]

let document = Render.htmlDocument view

printfn "%s" document
```

Enter fullscreen mode Exit fullscreen mode

> To Run this, copy this content into a file named `script.fsx` (or whatever name you prefer) and type:
> 
> -   `dotnet fsi run script.fsx`

when you open the `Feliz.ViewEngine` namespace you have access to the `Html` and `prop` types these have all the tags and attributes you may need to construct HTML content. if you are feeling tired of writing `prop` and `Html` you can even `open type` those classes and access all of their static methods  

```
#r "nuget: Feliz.ViewEngine"

open Feliz.ViewEngine
open type Html
open type prop

let view = 
    html [
        head [ title "Feliz" ]
        body [
            header [ text "Feliz" ]
        ]
    ]

let document = Render.htmlDocument view

printfn "%s" document
```

Enter fullscreen mode Exit fullscreen mode

> To Run this, copy this content into a file named `script.fsx` (or whatever name you prefer) and type:
> 
> -   `dotnet fsi run script.fsx`

that will save you a few keystrokes as well! you might run into some naming clashes but you can also just use qualified `prop` and `Html` if you need it.

let's continue with the card component to see how you can compose different view elements. There are a few differences when you use the Feliz DSL, primarily these are because how the React API is defined, you can't mix properties with children, so you have to pass `prop.children` to be able to define child elements for your components  

```
#r "nuget: Feliz.ViewEngine"

open Feliz.ViewEngine
open type Html
open type prop

// custom card, but you can't customize it's classes only the children elements
let card (content: ReactElement seq) = 
    article [
        className "card is-green"
        children content
    ]
// this card footer will allou you to define any property  of the footer element
let cardFooter content =
    footer [
        className "card-footer is-rounded"
        yield! content
    ]

let slotedHeader (content: ReactElement seq) = 
    header [
        className "card-header"
        // pass the contents directly to the children property
        children content
    ]

let customizableHeader content = 
    header [
        className "card-header"
        // allow any property to be set
        yield! content
    ]

let card1 = 
    div [
        card [
            // our slottedHeader only allows to pass children not props
            slotedHeader [
                h1 [ text "This is my custom card"]
                // className "" <- can't do this
            ]
            p [ text "this is the body of the card" ]
            cardFooter [
                custom("data-my-attr", "extra attributes")
                children (p [text "This is my footer"])
            ]
        ]
    ]

let card2 = 
    div [
        card [
            /// our customizable header allows us
            /// to pass properties as well as children elements
            customizableHeader [
                children (h1 [ text "This is my custom card"])
                className "custom class" 
            ]
            p [ text "this is the body of the card" ]
            cardFooter [
                custom("data-my-attr", "extra attributes")
                children (p [text "This is my footer"])
            ]
        ]
    ]

let r1 = Render.htmlView card1
let r2 = Render.htmlView card2

printfn "%s\n\n%s" r1 r2
```

Enter fullscreen mode Exit fullscreen mode

> To Run this, copy this content into a file named `script.fsx` (or whatever name you prefer) and type:
> 
> -   `dotnet fsi run script.fsx`

But there's a thing that will happen here, unlike Giraffe.ViewEngine Feliz doesn't strip existing props so our header will end up like this  

```
<header class="card-header" class="custom class">
    <h1>This is my custom card</h1>
</header>
```

Enter fullscreen mode Exit fullscreen mode

In HTML the last defined property always wins, so keep in mind that depending on your intent you might need properties or children ant that might be the deciding factor between using one way or the other.

### [](https://dev.to/tunaxor/doing-some-io-in-f-4agg#scriban)Scriban

If you like me can't simply just leave HTML because of _reasons_ there are also Text based alternatives like [Scriban](https://github.com/scriban/scriban) which allow you to just write an html file and just fill it with data  

```
#r "nuget: Scriban"

open Scriban

type Product = { name: string; price: float; description: string }

let renderProducts products = 
    let html = 
        """
        <ul id='products'>
        {{ for product in products }}
          <li>
            <h2>{{ product.name }}</h2>
                 Price: {{ product.price }}
                 {{ product.description | string.truncate 15 }}
          </li>
        {{ end }}
        </ul>
        """
    let result = Template.Parse(html)
    result.Render({| products = products |})

let result =
    renderProducts [
        { name = "Shoes"; price = 20.50; description = "The most shoes you'll ever see"}
        { name = "Potatoes"; price = 1.50; description = "The most potato you'll ever see" }
        { name = "Cars"; price = 10.3; description = "The most car you'll ever see" }
    ]

printfn "%s" result
```

Enter fullscreen mode Exit fullscreen mode

> To Run this, copy this content into a file named `script.fsx` (or whatever name you prefer) and type:
> 
> -   `dotnet fsi run script.fsx`

If you have ever used jinja, moustache, handlebars, liquid and similar template engines it will look familiar to you basically here we define an HTML string (which can be read from an HTML file in disk) we parse it, then render it with a data source (in case we need one)

> scriban has a TON of helpers in it's scripting language (like those pipes that are truncating a string to 15 characters)

If you want to compose components in the scriban templates the approach is way way different  

```
#r "nuget: Scriban"

open System
open Scriban

type Product = 
    { name: string;
      price: float; 
      details : {| description: string |} }
// create a fragment/component html string
let detailDiv = 
    """
    <details>
        <summary> {{ product.details.description | string.truncate 15 }} <summary>
        {{ product.details.description }}
    </details>
    """

let renderProducts products = 
    let html = 
        // here with the help of sprintf
        // and {{ "%s" | object.eval_template }}
        // we use F# to pre-process the template
        sprintf
            """
            <ul id='products'>
            {{ for product in products }}
              <li>
                <h2>{{ product.name }}</h2>
                     Price: {{ product.price }}
                     {{ "%s" | object.eval_template }}
              </li>
            {{ end }}
            </ul>
            """
            detailDiv
    let result = Template.Parse(html)
    result.Render({| products = products |})

let result =
    renderProducts [
        { name = "Shoes"
          price = 20.50
          details = 
            {| description = "The most shoes you'll ever see" |} }
        { name = "Potatoes"
          price = 1.50
          details =
            {| description = "The most potato you'll ever see"  |} }
        { name = "Cars"
          price = 10.3
          details =
            {| description = "The most car you'll ever see"  |} }
    ]

printfn "%s" result
```

Enter fullscreen mode Exit fullscreen mode

> To Run this, copy this content into a file named `script.fsx` (or whatever name you prefer) and type:
> 
> -   `dotnet fsi run script.fsx`

now, keep in mind that you're using F# and this you can modify the strings before passing them to the final template before parsing it but this leads to handling strings here and there and possibly getting into regex territory and to be honest I don't like that. I think this approach is best suited to templates you already set and know what the model for them is and that they are not super dynamic, you can still perform a lot of dynamic operations with the scriban scripting capabilities inside the template.

also keep in mind that if you really need it, you can parse and compile multiple HTML templates and then pass them together to a layout and just do a final render but I'm not aware of how performant/useful that is in practice

## [](https://dev.to/tunaxor/doing-some-io-in-f-4agg#closing-thoughts)Closing Thoughts

So that's it generating HTML isn't complex and it might be useful for you, perhaps you want to build a resume from JSON to HTML, perhaps you need to build a report from the last year sales or another similar use case.

As always feel free to ping me on Twitter or the comments below 😁