# Web Components in a Real Project

![](https://habrastorage.org/webt/at/3x/wm/at3xwmorjkpfckmmrjqp1uncvzg.png)

Hello everyone! My name is Arthur and I work as a frontend developer at Exness. Not so long ago, we transferred one of our projects to web components. I'll tell you what problems I had to face, and how many of the concepts that we are used to when working with frameworks are easily transferred to web components.
 
Looking ahead, I will say that the implemented project has been successfully tested on our wide audience, and the bundle size and loading time have been significantly reduced.

<cut />

I assume that you already have a basic understanding of the technology, but even without it, it will be clear that it is quite convenient to work with web components and organize the architecture of the project.

## Why Web Components?
The project was small, but demanding in terms of bundle size; the main ui-components were the forms. Someone will say that everything could be done with native html + js, which would be as lightweight as possible. But when it comes to supporting and expanding the project, giving up all the benefits of component development would be like a leap into the past. Alternatively, one could use [Svelte](https://github.com/sveltejs/svelte) or, for example, [Preact](https://github.com/preactjs/preact). But trying to do everything in terms of native concepts was too tempting idea.


## Is the future already here?
Most modern browsers, including mobile ones, support web components out of the box. For the rest, there are [polyfills](https://www.npmjs.com/package/@webcomponents/webcomponentsjs). It is noteworthy that for polyfills there is a smart [loader](https://www.npmjs.com/package/@webcomponents/webcomponentsjs#using-webcomponents-loaderjs) (~ 2kB), which does not load the polyfill of a certain functionality, if there is native browser support - thus nothing is loaded into modern browsers. Polyfills claim support up to IE11 and older versions of Edge. Fortunately, we do not support IE in our projects, everything really works in Edge. Also, everything works in the Chinese UC Browser and QQ Browser, including their mobile versions.

*Small restrictions on the work of polyfills:*
- *Polyfill tag `<slot> </slot>` for IE11 & Edge does not participate in event bubbling;*
- *This set of polyfills does not provide functionality for extending inline html elements. More on this below.*


## LitElement
“So much boilerplate to build components and reactively work with props! We need to speed up the process and write a class that extends HTMLElement and implements this code ”- this was the first thought that came up when diving into web components. Of course, I didn't have to write anything - the [LitElement](https://lit-element.polymer-project.org/) class (~ 7kb) takes over this work. And the [lit-html](https://lit-html.polymer-project.org/) used in it provides a convenient and optimized rendering engine and simplifies data binding and event subscription.

In addition, LitElement extends the standard **web component lifecycle** with convenient [additions](https://lit-element.polymer-project.org/guide/lifecycle), making it much like the component lifecycle of popular frameworks.

LitElement is another dependency, some will even say "another framework". But I deliberately *decided on these plus 7kB,* because LitElement looks like one of the options for a natural add-on over the existing native API. One way or another, it would be present in the project, at least for the implementation of the boilerplate code.


## Functional Component Templates
This is indirectly related to the topic of the article, but lit-html in its ~ 3.5kB contains a very convenient ability to describe the interface using functions. Moreover, the update of the DOM structures of such component functions is optimized: only those blocks are rendered, the values of which have changed since the previous render. In some cases and with due resourcefulness, the entire interface can be described only with functions, decorators and directives (about them a little later):

```Javascript
import {html, render} from 'lit-html'

const ui = data => html` ... $ {data} ... `

render (ui ('Hello!'), document.body)
```

Moreover, in some templates, you can use others:
```Javascript
const myHeader = html` <h1> Header </h1> `
const myPage = html`
  $ {myHeader}
  <div> Here's my main page. </div>
`
```

In other cases, you can come up with a wrapper to create custom elements from such functions:
```Javascript
const defineFxComponent = (tagName, FxComponent, Parent = LitElement) => {
  const Component = class extends Parent {
    render() {
      return FxComponent (this.data)
    }
  }
  customElements.define (tagName, Component)
}

defineFxComponent ('custom-ui', ui)

render (html` <custom-ui .data = "Hello!"> </custom-ui> `, document.body)
```

I will not dwell on the conveniences of templating, styling, passing attributes, data binding and subscribing to events, conditional and looping when working with lit-html. All of this is detailed in the [documentation](https://lit-html.polymer-project.org/guide/writing-templates). I will focus on what can be missed by a cursory glance at the manual, but can be useful.

### svg
The most hidden of these functions, which is not mentioned in the manual (but is in the [API](https://lit-html.polymer-project.org/api/modules/lit_html.html#svg)) is the tag ```svg`` ```. As you might guess, it is used to work with vector graphics, when working with which through ```html`` ``` some problems may arise. I got them when I tried to pass a TemplateResult (this is the result of executing ```html`` ```) inside my icon component - unnecessary closing tags appeared and the graphics were not rendered. When using ```svg`` ``` and passing the SVGTemplateResult, everything fell into place.

### Directives
Directives are functions that describe how their content will be displayed. Lit-html uses classes that implement the Part interface to store and output values for DOM representation. It is the Part interface that provides smart rendering that only updates what has changed, and directives are the way to access and influence this mechanism.

Directives can be of one of five types, each of which has access to the corresponding Part implementation:
- To display content (NodePart);
- To transfer an attribute (AttributePart);
- To transfer a boolean attribute (BooleanAttributePart);
- For data binding or property transfer (PropertyPart);
- For subscription to events (EventPart).

Each type of directive can only be used in a suitable place and has no meaning elsewhere.

The directive stores the value `value` - this is what was displayed in its place during the last rendering. To set a new value, there is the `setValue()` function. To force the update of values ​​in the DOM after the render has finished, use the `commit()` function (useful for asynchronous actions).

An example of a custom directive (which has access to the NodePart class - for displaying content) that stores and displays the number of renders:
```Javascript
import {directive} from 'lit-html'

const renderCounter = directive (() => part =>
  part.setValue (part.value === undefined? 0: part.value + 1)
)
```

Lit-html has a useful [set of built-in directives](https://lit-html.polymer-project.org/guide/template-reference#built-in-directives). There are analogs of some React hooks, functions for working with styles and classes as with objects, functions for asynchronous content updating, optimization, insecure html installation, etc.

A more complex state can be stored next to the directive for use inside the directive. Example [here](https://lit-html.polymer-project.org/guide/creating-directives#maintaining-state).

Custom directives and decorators can also be used as analogs for higher-order components. With this approach, you need to take care of the reactivity within the directive yourself. Example [here](https://github.com/jmas/lit-redux/blob/master/index.js).

### shady-render
If you create a base class using lit-html and Shadow DOM, you will need polyfills for older browsers. Lit-html has a separate [shady-render module](https://lit-html.polymer-project.org/guide/styling-templates#polyfilled-shadow-dom:-shadydom-and-shadycss) that integrates these polyfills.



## Higher-order components
HOC is a powerful pattern often used when working with React or Vue. When using it, the composition of components becomes simple and concise, and I would like to use some kind of its analogue when working with web components. Since web components are classes, for myself, as an analogue of HOC, I decided to use functions that would return the extension of the class passed to them as a parameter.

### Expansion of functionality
In the project I needed redux, so let's take a look at [the connector for it](https://github.com/realiarthur/lite-redux) as an example. Below is the code for a decorator that takes a store and returns a standard redux connector. Inside the class, mapStateToProps are accumulated from the entire inheritance chain (for those cases if it contains a HOC that also communicates with redux), so that later, when the component is embedded in the DOM, one callback subscribes them all to change the redux state. When a component is removed from the DOM, this subscription is removed.

```Javascript
import {bindActionCreators} from 'redux'

export default store => (mapStateToProps, mapDispatchToProps) => Component =>
  class Connect extends Component {
    constructor (props) {
      super (props)
      this._getPropsFromStore = this._getPropsFromStore.bind (this)
      this._getInheritChainProps = this._getInheritChainProps.bind (this)

      // Accumulation mapStateToProps
      this._inheritChainProps = (this._inheritChainProps || []). concat (
        mapStateToProps
      )
    }

    // Function for getting data from store
    _getPropsFromStore (mapStateToProps) {
      if (!mapStateToProps) return
      const state = store.getState()
      const props = mapStateToProps (state)

      for (const prop in props) {
        this [prop] = props [prop]
      }
    }

    // Callback to subscribe to the store change, which will call all mapStateToProps from the inheritance chain
    _getInheritChainProps() {
      this._inheritChainProps.forEach (i => this._getPropsFromStore (i))
    }

    connectedCallback() {
      this._getPropsFromStore (mapStateToProps)

      this._unsubscriber = store.subscribe (this._getInheritChainProps)

      if (mapDispatchToProps) {
        const dispatchers =
          typeof mapDispatchToProps === 'function'
            ? mapDispatchToProps (store.dispatch)
            : mapDispatchToProps
        for (const dispatcher in dispatchers) {
          typeof mapDispatchToProps === 'function'
            ? (this [dispatcher] = dispatchers [dispatcher])
            : (this [dispatcher] = bindActionCreators (
                dispatchers [dispatcher],
                store.dispatch,
               () => store.getState()
              ))
        }
      }

      super.connectedCallback()
    }

    disconnectedCallback() {
      // Remove subscription to change store
      this._unsubscriber()
      super.disconnectedCallback()
    }
  }
```

The most convenient way to use this method when initializing store is to create and export a regular connector that can be used as a higher-order component:

```Javascript
// store.js
import {createStore} from 'redux'
import makeConnect from 'lite-redux'
import reducer from './reducer'

const store = createStore (reducer)

export default store

// Create a standard connector
export const connect = makeConnect (store)
```

```Javascript
// Component.js
import {connect} from './store'

class Component extends WhatEver {
  /* ... */
}

export default connect (mapStateToProps, mapDispatchToProps) (Component)
```

### Extending Display and Observable Properties
In many cases, in addition to extending the functionality of a component, it is required to wrap its display. It is convenient to use the render function of the extensible component for this. In addition, it can be useful to extend the list of observed properties to ensure reactivity: `get observedAttributes()` for web components, or `get properties()` for LitElement. To illustrate, I will give an example of a password input field that expands the input text field component passed to it:

```Javascript
const withPassword = Component =>
  class PasswordInput extends Component {
    static get properties() {
      return {
        // Assume super.properties already contains type property
        ... super.properties,
        addonIcon: {type: String}
      }
    }

    constructor (props) {
      super (props)
      this.type = 'password'
      this.addonIcon = 'invisible'
    }

    setType (e) {
      this.type = this.type === 'text'? 'password': 'text'
      this.addonIcon = this.type === 'password'? 'invisible': 'visible'
    }

    render() {
      return html`
        <div class = "with-addon">
          <! - Extendable class display ->
          $ {super.render()}
          <div @click = $ {this.setType}>
            <custom-icon icon = $ {this.addonIcon}> </custom-icon>
          </div>
        </div>
      `
    }
  }

customElements.define ('password-input', withPassword (TextInput))
```

Here I would like to draw your attention to the `... super.properties` line in the` get properties() `method, which allows you not to define the properties already described in the extensible component. And to the `super.render()` line in the `render` method, which displays the display of the extensible component at the specified place in the markup.


When using this approach to implement an analogue of HOC, it is worth taking some precautions:
- Be careful when naming properties and methods, as you can override them in an inherited component;
- Remember that when passing a class method as a callback to any event, there is a possibility that this method will be overridden somewhere in the inheritance chain, and only the last of the HOCs will be subscribed to the event, and not both;
- Try to subscribe as clearly as possible to changes in properties transferred from the HOC.


## How I Ditched Shadow DOM and Inline Element Extension
As I said, the main ui components of my project are forms and various input fields. For convenience, I would like to use a component that provides convenient tools for working with a form and implements the necessary functionality: storing and updating state (values, errors, touched), validation, submit processing, reset.

This functionality has nothing to do with the topic of the article, but I would like to talk about what to wrap it in. Looking ahead, before making a working version, I tried three packages: a standard web component, an inline `<form>` element extension, and a higher-order component. And this is a reason for a little story ...

### Version 1. Web Components and the Shadow DOM
The first thing that came to my mind was to make standard web components for input fields and forms using Shadow DOM as part of best practices, wrap input fields in a HOC to communicate with the form, insert the form into the page with a separate custom tag. Everything in order.

On the one hand, I needed the functionality of the `<form>` tag and its HTMLFormElement class (for example, calling submit when pressing Enter), but I didn't want to add it every time in addition to the custom form tag. On the other hand, I could not use the slot to wrap its contents in the `<form>` tag, because the form events stopped working - apparently the `<form>` is not yet fully ready for the slot inside itself.

The solution was to pass the template as a property for the custom form. This is a kind of analogue of passing a render function to React:

```Javascript
// Form component
import {LitElement, html} from 'lit-element'

class LiteForm extends LitElement {
  /* ... form functionality ... */

  render() {
    return html` <form @submit = $ {this.handleSubmit} method = $ {this.method}>
      $ {this.formTemplate (this)}
    </form> `
  }
}

customElements.define ('lite-form', LiteForm)
```

```Javascript
// Sample form
import {html, render} from 'lit-element'

const formTemplate = ({values, handleBlur, handleChange, ... props}) =>
  html` <input
      .value = $ {values.firstName}
      @input = $ {handleChange}
      @blur = $ {handleBlur}
    />
    <input
      .value = $ {values.lastName}
      @input = $ {handleChange}
      @blur = $ {handleBlur}
    />
    <button type = "submit"> Submit </button> `

const MyForm = html` <lite-form
  method = "POST"
  .formTemplate = $ {formTemplate}
  .onSubmit = $ {{/*...*/}}
  .initialValues ​​= $ {{/*...*/}}
  .validationSchema = $ {{/*...*/}}
> </lite-form> `

render (html` $ {MyForm} `, document.getElementById ('root'))
```

Of course, I didn't want each input field to pass properties and events on the form. I also wanted to work with custom input fields and make it easier to output errors. Therefore, I needed several higher-order components to work with the form, such as `withField` or` withError`.

This implementation required the HOCs to be able to independently find their form or communicate with it through events. After going through several options (communication via the event bus, general context - all of them fit if there is only one form on the page), I settled on a very simple, but working one:

```Javascript
// here the IS_LITE_FORM constant is the name of the boolean attribute that each element of the custom form has
const getFormClass = element => {
  const form = element.closest (`[$ {IS_LITE_FORM}]`)
  if (form) return form

  const host = element.getRootNode(). host
  if (! host) throw new Error ('Lite-form not found')
  return host [IS_LITE_FORM]? host: getFormClass (host)
}
```
Everything is trivial here: recursive search for an element with an attribute indicating that this is the desired form. I would like to note the function [getRootNode](https://developer.mozilla.org/en-US/docs/Web/API/Node/getRootNode), thanks to which the search passes through the tree of nested Shadow DOMs - a necessary function when solving such specific problems.

Using `withField`, I could greatly simplify the form template:
```Javascript
const formTemplate = props =>
  html` <custom-input name = "firstName"> </custom-input>
    <custom-input name = "lastName"> </custom-input>
    <button type = "submit"> Submit </button> `
```

In general, everything worked fine, but ... Before I tell you why I abandoned Shadow DOM, a few more words about it.

### Styling Shadow DOM externally and pseudo-classes: host and: host-context
You can use CSS variables to customize component styles from the main document. They go through the Shadow DOM, are visible everywhere, and that's quite handy. But situations arise when this is not enough, for example, when conditional styling is necessary:
```css
:host ([rtl]) {
  text-align: right;
}

:host-context (body [dir = 'rtl']) .text {
  text-align: right;
}
```
The `:host` pseudo-class allows you to style the root element of a component only if it matches the given selector. In the example, the text will be right-aligned only if the root element has the `rtl` attribute.

The `:host-context` pseudo-class allows you to understand what context the Shadow DOM is in and react to it. In the example, a block with class `.text` inside the component will align the text appropriately depending on the` dir` attribute set for `body`.

### Bubbling events through the Shadow DOM
Events when bubbling through the Shadow DOM behave in a somewhat unusual way: they replace the target element, which becomes the web component itself. This is done in order to maintain DOM encapsulation, and this must be taken into account when developing, because the final `target` may not contain all the properties that the original target element contained.

Bubbling through the Shadow DOM can be controlled by the `composed` event property. To make the event bubble through the Shadow DOM, both its `composed` and `bubbles` must be set to `true`. Most of the standard events float through the Shadow DOM successfully.

### Why I gave up on Shadow DOM
First, due to the impossibility of auto-filling forms with stored user data (login-password, address, credit card data) if the form or input fields are hidden behind the Shadow DOM. Also, the browser does not offer to save user data from such forms. In my project, this flaw (or a feature, whichever is closer to you) has become critical.

*Of course, if only slots are used to submit a form template, and this template is described in the main document to which it will belong, then autocompletion will occur even when using the Shadow DOM in the form component. But other than that, all components in which the form is nested and all input fields must not use the Shadow DOM. These restrictions are so strong that it is easier to drop Shadow DOM.*

Secondly, due to the lack of a querySelector analog that would work through the Shadow DOM. For metrics and analytics, such a tool would be useful so that, for example, in the Google Tag Manager, you do not write long constructs like `document.querySelector (...) .shadowRoot.querySelector (...) .shadowRoot.querySelector (...). shadowRoot.querySelector (...) `

With the abandonment of Shadow DOM, there were no particular difficulties. In LitElement, the following code is enough for this:
```Javascript
createRenderRoot() {
  return this
}
```
Along with the Shadow DOM, I lost the `blur` event bubble from input fields. I used it inside withField so that I don't have to manually pass it out of the component. But I signed up for it during the dive stage and it worked.

### Version 2. Inline element extension
After abandoning the Shadow DOM, it seemed to me a good idea to extend the built-in HTMLFormElement class and the `<form>` tag - the layout would look like native, and at the same time, access to all form events would be preserved, and this required minimal code changes:
```Javascript
// Form component
class LiteForm extends HTMLFormElement {
  connectedCallback() {
    this.addEventListener ('submit', this.handleSubmit)
  }

  disconnectedCallback() {
    this.removeEventListener ('submit', this.handleSubmit)
  }

  /* ... form functionality ... */
}

customElements.define ('lite-form', LiteForm, {extends: 'form'})
```
Everything worked as in the usual form, only with additional functionality:
```Javascript
// Sample form
const MyForm = html` <form
  method = "POST"
  is = "lite-form"
  .onSubmit = $ {{...}}
  .initialValues ​​= $ {{...}}
  .validationSchema = $ {{...}}
>
  <custom-input name = "firstName"> </custom-input>
  <custom-input name = "lastName"> </custom-input>
  <button type = "submit"> Submit </button>
</form> `

render (html` $ {MyForm} `, document.getElementById ('root'))
```
Here I would like to draw attention to the first argument in the `customElements.define` function and the` is` attribute in the form element, which indicate what "type" the tag will be. And also on the third argument in `customElements.define`, which specifies which tag will be expanded.

Everything worked great and looked great. But not in all browsers: Safari does not support inline element expansion because it considers it unsafe. This also applies to browsers for iOS (including Chrome, Firefox, etc.). Rumor has it that Apple may revise this behavior, but in Safari 13 and iOS, things remain the same. Although there are [polyfills](https://www.npmjs.com/package/@ungap/custom-elements-builtin), I decided not to use this concept until Safari and iOS started supporting inline element extensions.

### Version 3. Higher-order component
After a couple of unsuccessful attempts to wrap the functionality of the form, I decided to leave it outside of it and make a higher-order component to wrap everything I needed. This required some small code changes:
```Javascript
// Higher-order component of the form
export const withForm = ({
  onSubmit,
  initialValues,
  validationSchema,
  ... config
} = {}) => Component =>
  class LiteForm extends Component {
    connectedCallback() {
      this._onSubmit = (onSubmit || this.onSubmit || function() {}). bind (this)
      this._initialValues ​​= initialValues ​​|| this.initialValues ​​|| {}
      this._validationSchema = validationSchema || this.validationSchema || {}
      /* ... */
      super.connectedCallback && super.connectedCallback()
    }

    /* ... form functionality ... */
  }
```

Here, in the `connectedCallback()` function, the form accepts a config (onSubmit, initialValues, validationSchema, etc.) either from the arguments passed to `withForm()` or from the extensible component itself. This allows you to wrap any classes, as well as build base classes that can be used in the layout by passing the config in it. By the way, in this way you can build both base classes from the first implementations of the form:

```Javascript
// An example of a base class from the first implementation of the form:
// Standard web component or LitElement
import {withForm} from 'lite-form'

class LiteForm extends LitElement {
  render() {
    return html` <form @submit = $ {this.handleSubmit} method = $ {this.method}>
      $ {this.formRender (this)}
    </form> `
  }
}

customElements.define ('lite-form', withForm (LiteForm))
```

```Javascript
// An example of a base class from the second implementation of the form:
// Extend the inline element
import {withForm} from 'lite-form'

class LiteForm extends HTMLFormElement {
  connectedCallback() {
    this.addEventListener ('submit', this.handleSubmit)
  }

  disconnectedCallback() {
    this.removeEventListener ('submit', this.handleSubmit)
  }
}

customElements.define ('lite-form', withForm (LiteForm), {extends: 'form'})
```

On the other hand, you can not create the base class of the form, but wrap the final components containing form templates in `withForm()` and pass the config to the HOC:

```Javascript
// Sample form
import {withForm} from 'lite-form'

class UserForm extends LitElement {
  render() {
    return html`
      <form method = "POST" @submit = $ {this.handleSubmit}>
        <custom-input name = "firstName"> </custom-input>
        <custom-input name = "lastName"> </custom-input>
        <button type = "submit"> Submit </button>
      </form>
    `
  }
}

const enhance = withForm ({
  initialValues: {/*...*/},
  onSubmit: {/*...*/},
  validationSchema: {/*...*/}
})

customElements.define ('user-form', enhance (UserForm))
```
Extensible classes can both use the Shadow DOM and remain public and better respond to the specifics of the project. The complete code of the form component with examples can be found [see here](https://github.com/realiarthur/lite-form).


## Conclusion
In the title of the article, “web components” could have been replaced by “custom elements” - in the end, this became the only API that I used in the project, and the most sophisticated of all the technology. But I would like to believe that it will evolve in terms of creating elements, passing properties and subscribing to their changes, to become more concise. I used LitElement mainly for this, and its lifecycle was not useful to me - the standard was enough.

Slots are also a promising technology, especially if they could be used outside of the Shadow DOM concept ([no hacks](https://stackoverflow.com/questions/48726904/is-there-a-way-workaround-to-have-the-slot-principle-in-hyperhtml-without-using)). This would provide ample opportunities for component composition.

Shadow DOM will definitely be useful in a number of cases, but its widespread use is questionable. The use of Shadow DOM comes with some limitations and inconveniences. At the same time, more efficient technologies can be used to encapsulate CSS, and DOM encapsulation is not always needed.

Although there are still a number of open questions when using web components, in general, the technology looks promising, and its use can already justify itself on small projects. But in its current state, it seems premature to use it as a replacement for frameworks on a large project that a group of developers is working on.



### Useful links
[webcomponents.org,](https://www.webcomponents.org/) [Polymer,](https://www.polymer-project.org/) [LitElement,](https://lit-element.polymer-project.org/) [lit-html,](https://lit-html.polymer-project.org/) [Vaadin Router,](https://vaadin.com/router) [Vaadin Components,](https://vaadin.com/components) [lite-redux,](https://www.npmjs.com/package/lite-redux) [lite-form,](https://www.npmjs.com/package/lite-form) [awesome-lit-html,](https://github.com/web-padawan/awesome-lit-html) [Polyfills,](https://github.com/webcomponents/polyfills/tree/master/packages/webcomponentsjs) [custom-elements-builtin polyfill](https://www.npmjs.com/package/@ungap/custom-elements-builtin)
