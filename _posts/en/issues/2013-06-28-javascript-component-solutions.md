---

title: JavaScript components low basics

categories: en issues

layout: post

invisible: true

---
This article features common problems that developers often face (or maybe used
to face before the current open source era)
when writing JavaScript for the client side; possible solutions to these
problems and explanations of my choices.

## Client side magic
I am going to write about client-side JavaScript only, the code runs under a
browser and empowers an interface with interactivity.

A piece of interface is represented with some HTML (and obviously styled with
CSS). Also it can be equiped with JavaScript usually describing how to react on
user's actions.

So, when developing a piece of interface, we write:

1. HTML/CSS
2. JavaScript
3. Some code to link 2 to 1.<br/>
   This code says which part of HTML on a page corresponds to a JavaScript
   component.

## A few ways to matchmake
Thus, the so-called 'linking' joins designated piece of JavaScript to its
corresponding piece of HTML. This can be done in different ways.

### Linking with events attributes. The old school way.
Since, in most cases running JavaScript is a reaction to some events firing,
there is an obvious way to declare in HTML what the reaction should be. These are
the [intrinsic event
attributes](http://www.w3.org/TR/html401/interact/scripts.html#events).

For example, if you need a hidden `<div>` with an authorization form to be shown
when a user clicks on a "log in" link, this can be done with a JavaScript
function. Then, call it in an event `onclick` attribute.

```
<a href="/login-page" onclick="return showLogin()">log in</a>
<div id="login-form">
    <form name="login" ... >
    ...
    </form>
</div>
```

```
dom = document.getElementById ? true : false;

function showLogin() {
  if ( dom && document.forms['login'] ) {
    document.getElementById('login-form').style.display = 'block';
    return false;
  } else return true;
}
```

That works fine, except it uses global variables and demonstrates constraint with
choosing a script loading strategy, which will be detailed below. Also in this
case you cannot code any predefined action; everything happens after a user
does something.<br/>
I personally don't like this method because it leaves JavaScript implants in
HTML code, whose main purpose is to describe the document structure, not page
behavior.

### Moving JavaScript into JavaScript
Therefore, a better way would be to stack everything related to JavaScript
into the page `.js` file.

From here I will write jQuery code to save time and some letters. If you prefer
another library, just get the gist.

```
$.fn.myPlugin = function() {
  this.fadeIn(
    'normal',
    function() { ... }
  );
}

...

$('#element').myPlugin();
```

With plugins you can describe behavioural pattern and then apply it to the
elements needed. In this case your don't need to inject JavaScript into HTML any
more, which makes maintenance easier. But for every single element you still
have write a line of code to apply the plugin.

### Declaring behavior with CSS classes
In case of having dozens of controls per page it's not useful to unitialize each of them
separately. A better way would be to mark HTML block with a matching CSS
class.

```
<input class="js-myPlugin" type="text" name="login" />
```

```
$.fn.myPlugin = function() {
  this.fadeIn(
    'normal',
    function() { ... }
  );
}

$('.js-myPlugin').myPlugin();
```

With that you need your 'linking' code only once for a plugin; it can be
written right in the plugin file, so that the consistency is keeping.

### One CSS class to rule them all
However, when the variety of components grows, the 'linking' solution should be
even more declarative. As a result, reading the block HTML enables
to understand if there should be some JavaScript for it. You can mark all the
'alive' parts of your page with a special CSS class; let it be `js`.

```
<input
  class="inpt js"
  data-component="myPlugin"
  name="login"
  value=""/>
```

The step up of this method is the need of information which plugin is to apply to
a particular control and a little more code to initialize. This leads onto a
question about how to parametrize the component. This example suggests `data`
attributes although there is another idea (spoiler!).

With minor change there is the code initializing a proper JavaScript component
for all the blocks on a page:

```
$('.js').each(function() {
    var $this = $(this);
    $this[$this.data('component')]();
})
```

## Parametrizing
Not only a component name has to be provided when initializing by `js` CSS class.
Instances of the same component also may need to pass some information. The
input plugin can validate the field value. But depending on the input different
kinds of data can be expected.

```
$.fn.myInput = function() {
    // ...
}
```

```
<input name="login" value=""/>
<input name="zipcode" value=""/>
```

It is easy to provide per-instance parameters when initializing every instance
separately.

```
$('input[name="login"]').myInput({
  validator: 'login'
});
$('input[name="zipcode"]').myInput({
  validator: 'zipcode'
});
```

### data attributes
With more declarative manner you need to include such information into a
component HTML structure. Already mentioned data attributes can help with that.

```
<input
  class="inpt js"
  data-component="myInput"
  data-validator="login"
  name="login"
  value=""/>

<input
  class="inpt js"
  data-component="myInput"
  data-validator="zipcode"
  name="login"
  value=""/>
```

```
$.fn.myInput = function() {
    if (this.data('validator') === 'login') // Supported by many frameworks
}
```

Data attribute is a brainchild of HTML5, so carry a doctype when using them.

As a widespread solution with many libraries providing helpers for them, data
attributes make parametrizing not difficult. Although the limitation of being
able to use key-value pairs only can affect the plugin architecture.

### onclick
An elegant way is to provide data for a JavaScript component using natural
JavaScript format, such as hashes. Here the event attributes can serve us again.

```
<input
    class="myInput"
    onclick="return {
        validator: 'login'
    }"
    name="login"
    value=""/>
```

When a user normally clicks, the hash is booleanized into `true`, so the default
browser click reaction is exactly as expected.
From your plugin you can call the event-named method and get the data
in its native JavaScript format without any need to parse and convert.

```
$.fn.myInput = function() {
 this[0].onclick().validator === 'login'
};
```

As a JavaScript, this data can be of any type. Not only strings and number,
but large treeish hashes, arrays and even functions are possible. Such a
flexibility is extra salutary for organizing data in complex component and
customization.

```
<input
    class="inpt js"
    onclick="return {
        component: 'myInput',
        validator: function() { ... }
    }"
    name="login"
    value=""/>
```

With nested hashes an HTML block can be operated by 2 or more different
JavaScript components, each with its own bunch of parameters.

```
<input
    class="inpt js"
    onclick="return {
        'myInput' : {
            validator: function() { ... }
        },
        'anotherPlugin' : {}
    }"
    name="login"
    value=""/>
```

## Start fast
### Initializing components
Applying plugins to elements is only possible when the page is loaded and your
library can look for these elements in the document.

First, you can link the script file right after its related HTML code.

```
<input
    class="myInput"
    onclick="return {
        validator: 'login'
    }"
    name="login"
    value=""/>
<script src="js/plugins/myInput.js"></script>
```

Whilst it can work sometimes, the method is just unreliable. It's not certain
that the DOM will be ready by the time of the script loaded. Loading every plugin
as a separate file definitely slows down the page loading. In addition, this
is just ugly.

Then, as it's need for a page to be loaded when running scripts, logically an
`onload` event on the `window` object can help.

```
$(window).load(function(){
    $('.js').each(function() {
        var $this = $(this);
        $this[$this.data('component')]();
    })
})
```

The drawback is that `window.onload` occurs when all the content has been loaded.
Including images, which mostly are not necessary to run scripts.

The time you can safely initizlize your components is right after the document
has been loaded and parsed into DOM, which is accompanied by `document.ready`
event.

```
$(function() {
    $('.js').each(function() {
        var $this = $(this);
        $this[$this.data('component')]();
    })
})
```

With that all the plugins used can be bundled into one JavaScript file and linked
to a page within the `<head>` element as usual. Precisely loaded in advance,
the script will be run when the page is ready.

### Loading scripts
Although, putting `<script>` tag into `<head>` aslo delays page loading. The
better strategy is to draw a page as fast as possible and then introduce JavaScript
magic. Reach that with placing `<script>` before the closing `</body>` tag.

On the other hand, when a user sees a page, it is expected to work. By now, it's
hardly possible to respond user actions with plain HTML; people are used to rich
interfaces. If that's crucial for your page functioning, divide the script file
into 2. The first one linked in `<head>` provides major functionality. Then the
second placed before `</body>` imbues all the magic tricks.

## Common best practises
### Cover up
In this article, swiched from calling `showLogin` function to jQuery plugins
there were made some improvements in linking components to HTML code. But not
only. Giving JavaScript responsibility for the page behavior, it aslo enables to
step forward with avoiding global variables in the code.

This is true not only about JavaScript. The ids, form names and input names (all
the things that can be named in HTML) are also a kind of global object.

The usual advice is to avoid their usage.

> By reducing your global footprint to a single name, you significantly reduce
> the chance of bad interactions with other applications, widgets, or libraries.
> Douglas Crockford

With the [module
pattern](http://www.adequatelygood.com/JavaScript-Module-Pattern-In-Depth.html)
wrapping your component into self-invoking anonymous function all the global
variables turn into locals.

Using not `id` and `name` attributes to identify elements and access them from
JavaScript makes code more declarative, introduces more flexibility and make
code easier to maintain in the long run.

### Doublecheck you don't double-do
Initializing components can take time and resources. So, when first initialized,
the result can be saved and then reused. jQuery provides a nice `data` method
suitable for that.

```
$.fn.myPlugin = function() { return this.each(function() {
  var $this = $(this),
      data = $this.data('myPlugin');
  if (!data) {
    // init...
    $this.data('myPlugin', { /* ... */ });
  }
}};
```

### Beyond the element
If the corresponding block has been removed from a page, it's the component's
responsibility to clean up. Event listeners, constant background calculations
and even layout changes can happen. When being deactivated, a component has to
place things back with a `destroy` method.

```
$.fn.myPlugin = function() { return this.each(function() {
  var $this = $(this),
      data = $this.data('myPlugin');
  if (!data) {
    // init...
    $this.data('myPlugin', {
      destroy: function() { /* ... */ }
    });
  }
}};
```

### Initializing by request
With dinamically changed pages it's not enough to initialize components just ones after
`domReady`. There has to be some code to run every time you append new block
onto a page, which turns HTML fragment into a functioning component.

```
$.fn.myInit = function() {
    this.find('.js').each(function() {
        var $this = $(this);
        $this[$this.data('component')]();
    });
    return this;
})
```

A special method to destroy the components being removed has similar sense.

```
$.fn.myDestroy = function() {
    this.find('.js').each(function() {
        var $this = $(this),
            name = $this.data('component');
        $this.data(name).destroy();
    });
    return this.remove();
})
```

### Lazy initialization
One more trick is a [lazy
initialization](http://en.wikipedia.org/wiki/Lazy_initialization) of the
components. The idea is similar
to [lazy
loading](https://github.com/stevekwan/best-practices/blob/master/javascript/best-practices.md#lazy-load-assets-that-arent-immediately-required)
and merely means do as less as possible beforehand.

Instead of initializing all the components when `domReady` fires, you can do
this separately for every components at the time the particular components needs
that. Usually that's right after a user started to use
it and respective events triggered. So, all the necessary predefined actions
won't be run in advance and won't
slow down page rendering and the other components. Objects corresponding to
never-used components won't be created, which saves memory.

## Component core
Many code practises above would be repeated again and again when implementing
a lot of components. As you can guess, all the common things can be wrapped with a
framework representing the core code for the components.

One of the examples of this is jQuery UI. As many other frameworks it provides
API to create your own JavaScript components.

### Predefined component structure
With a component framework a developer uses API and describes a predefined
component structure. All the common things are in the core.

```
$.widget('my.component', {
    _create: function() { /* ... */ },
    destroy: function() { /* ... */ }
    myMethod: function() { /* ... */ },
});
```

### High-level patterns
Besides predefined structure, frameworks also provide architectural solutions to
different problems.<br/>
For example, with the releaving modules a core can provide an API to extend the
main component functionality.

```
$.widget('ui.dialog', $.ui.dialog, {
    close: function() {
        if(confirm('Close???')) {
            this._super('close')
        }
    }
})
```

Component frameworks define basics for reusing code and implementing reusable
JavaScript modules.

Another example of such a high-level component framework is JavaScript part of
[BEM](http://bem.info/). A developer can operate components with OOP common
patterns; create classes, their dynamic and static methods, as well as inherit
component and extend their functionality and reuse basics with super calls.

```
BEM.DOM.decl(
    { block: 'my-dialog', baseBlock: 'dialog' },
    {
        myMethod: function() {
            this.__base();
            this.__self.myStaticMethod();
        }
    },
    {
    myStaticMethod: function() { /*...*/ }
    }
);
```

## From now on
JavaScript component fundametals also cover problems like following.

* loading by request
* building page `.js` files
* dependency system
* complex component relations and data sharing
* data binding

## Credits
Before I wrapped up, let me 
thank to [Sergey Berezhnoy](https://github.com/veged) also known as
[@veged](https://twitter.com/veged) for his presentation about creating
JavaScript components. The Russian speaking readers, you can enjoy the video! [
Сергей Бережной, Яндекс
Разные способы создания клиентских
js-компонентов](http://events.yandex.ru/events/yasubbotnik/msk-jul-2012/talks/302/)