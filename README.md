Pyxl is an open source package that extends Python to support inline HTML. It converts HTML fragments into valid Python expressions, and is meant as a replacement for traditional python templating systems like [Mako](http://www.makotemplates.org/) or [Cheetah](http://www.cheetahtemplate.org/). It automatically escapes data, enforces correct markup and makes it easier to write reusable and well structured UI code. Pyxl was inspired by the [XHP](https://github.com/facebook/xhp/wiki) project at Facebook.

## Motivation

At Cove, where Pyxl was developed, we found that using templates was getting in the way of quickly building new features. There were the usual issues of remembering to escape data to prevent XSS holes, avoiding invalid markup and deciphering cryptic stack traces. More importantly, our templates were getting hard to manage and understand which made iterating on our product more work than should be necessary. 

Existing templating systems do support things like logic and reusable modules - but they are essentially like having a different programming language for writing UI which falls well short of python itself. The primary reason templating systems exist is because creating HTML in languages like python means writing crazy string manipulation code, or losing the niceness of writing actual HTML by doing something like this:

    import html
    print (
        html.head().appendChild(
            html.body().appendChild(
                    html.text("Hello World!"))))

To get around these limitations, we developed Pyxl which allowed us to treat HTML as a part of the python language itself. So, writing the above example with Pyxl would look like:

    # coding: pyxl
    from pyxl.html import *
    print <html><body>Hello World!</body></html>

This meant no longer dealing with a separate "templating" language, and a lot more control over how we wrote our front-end code. Also, since Pyxl maps HTML to structured python objects and expressions instead of arbitrary blobs of strings, adding support for things like automatically escaping data was trivial. Switching to Pyxl led to much cleaner and modularized UI code, and allowed us to write new features and pages a lot quicker.

## Installation

Download [pyxl-1.0.tar.gz](https://github.com/downloads/awable/pyxl/pyxl-1.0.tar.gz) and run the following commands from the directory you downloaded to.

    tar xvzf pyxl-1.0.tar.gz
    cd pyxl-1.0
    python setup.py build
    sudo python setup.py install
    sudo python finish_install.py

To confirm that Pyxl was correctly installed, run the following command from the same directory:

    python pyxl/examples/hello_world.py

You should see the string `<html><body>Hello World!</body></html>` printed out. Thats it! You're ready to use Pyxl.

## How it works

Pyxl converts HTML tags into python objects before the file is run through the interpreter, so the code that actually runs is regular python. For example, the `Hello World` example above is converted into:

    from pyxl.html import *
    print x_head().append_children(x_body().append_children("Hello World!"))

Pyxl's usefulness comes from being able to write HTML rather than unwieldy object instantiations and function calls. The `pyxl.html` module contains objects for all the basic HTML tags. The conversion is relatively straightforward: Opening tags are converted into object instantiations for the respective tag, nested tags are passed in as arguments to the `append_children` method, and closing tags close the bracket to the `append_children` call. As a result, a big advantage of this is that stack traces on errors map directly to what you've written. To learn more about how Pyxl does this, see the **Implementation Details** section below.

## Documentation

All python files with inline HTML must have the following first line:

    # coding: pyxl

With that, you can start using HTML in your python file.

### Inline Python Expressions

Anything wrapped with {}'s is evaluated as a python expression. Please note that attribute values must be wrapped inside quotes, regardless of whether it contains a python expression or not. When used in attribute values, the python expression must evaluate to something that can be cast to unicode. When used inside a tag, the expression can evaluate to anything that can be cast to unicode, an HTML tag, or a list containing those two types. This is demonstrated in the example below:

    image_name = "bolton.png"
    image = <img src="/static/images/{image_name}" />
    text = "Michael Bolton"
    print <div>{image}{text}</div>

### Dynamic Elements

Pyxl converts tags into python objects in the background, which inherit from a class called [`x_base`](https://github.com/awable/pyxl/blob/master/pyxl/pyxl/base.py). This means that tags have certain methods you can call on them. Here is an example snippet that uses the `append` function to dynamically create an unordered list.

    items = ['Puppies', 'Dragons']
    nav = <ul />
    for path, text in items:
        nav.append(<li>{text}</li>)

Another useful function is `children()`, which returns all the child nodes for an element. `children()` accepts an optional selector string as an argument to filter the children. Currently, there is only support for filtering the children by a class (format: ".class_name"), id (format: "#id_string") or tag name. Here is a snippet which adds all `input` elements from an existing form to a new form:

    new_form = <form action="/submit" method="POST">{old_form.children("input")}</form>

### Attributes

You can access any attribute of a tag as a member variable on the tag, or via the `attr(attr_name)` function. Setting attribute must happen via the `set_attr(attr_name, attr_value)` function i.e. do not set attrs by directly setting member variables. To access attributes that contain '-' (hypen) as a member variable, replace the hypen with '_' (underscore). For this reason, pyxl does not allow attributes with an underscore in their name. Here is an example that demonstrates all these principles:

    fruit = <div data-text="tangerine" />
    print fruit.data_text
    fruit.set_attr('data-text', 'clementine')
    print fruit.attr('data-text') # alternate method for accessing attributes

### Escaping

Pyxl automatically escapes all data and attribute values, therefore all your markup is XSS safe by default. One can explicitly avoid escaping by wrapping data in a call to `rawhtml`, but that only applies to data inside a tag. Everything in attribute values is always escaped. Note that static text inside tags (i.e. anything not inside {}'s) is considered regular HTML and is not escaped.

    safe_value = "<b>Puppies!</b>"
    unsafe_value = "<script>bad();</script>"
    unsafe_attr = '">'
    print (<div class="{unsafe_attr}">
               {unsafe_value}
               {rawhtml(safe_value)}
           </div>)

The above script will print out:

    <div class="&quot;&gt;">
        &lt;script&gt;bad();&lt;/script&gt;
        <b>Puppies!</b>
    </div>

### UI Modules

UI Modules are especially useful for creating re-usable building blocks in your application, making it quicker to implement new features, and keeping the UI consistent. Pyxl thinks of UI modules as user defined HTML tags, and so they are used just like you would use a `<div>` or any other tag.

Creating UI modules in Pyxl simply means creating a class that inherits from [`x_element`](https://github.com/awable/pyxl/blob/master/pyxl/pyxl/element.py) and implements the `render()` method. Modules must be prefixed with `x_`. This is an arbitrary requirement, but is useful in separating out pyxl modules from other things. To demonstrate, a useful UI module is a content box, which is a box with a linked title and some arbitrary content:

    # coding: pyxl
    from pyxl.html import *
    from pyxl.element import x_element

    class x_content_box(x_element):
        __attrs__ = {
            'title': unicode,
            'href': unicode,
        }
        def render(self):
            return (
                <div class="content_box">
                    <a href="{self.href}"><h3>{self.title}</h3></a>
                    {self.children()}
                </div>)

This makes the tag `<content_box>` available to us which accepts `title` and `href` as attributes. Here is an example of this new UI module being used.

    # coding: pyxl
    from pyx.xhtml import *
    from some_module import x_content_box

    content = <div>Any arbitrary content...</div>
    print <content_box title="Content Box Title" href="/content_link">{content}</content_box>

Some things to note about UI modules.

* Modules names must begin with `x_` and be an instance of `x_element`
* Modules must specify the attributes they accept via the `__attrs__` class variable. This is a dictionary where the key is the attribute name, and the value is the attribute type. Passing an attribute that is not listed in `__attrs__` will result in an error. The only exceptions are attributes accepted by all pyxl elements i.e. id, class, style, onclick, title and anything prefixed with "data-" or "aria-"
* Providing a `class` attribute for a UI module element will automatically append the class string to the underlying HTML element the UI module renders. This is useful when you want to style UI modules differently based on where it is being rendered.

### Fragments

The [`pyxl.html`](https://github.com/awable/pyxl/blob/master/pyxl/pyxl/html.py) module provides the `<frag>` tag, which allows one to group a set of HTML tags without a parent. Rendering the `<frag>` tag simply renders all the children, and doesn't add to the markup.

### Conditional HTML

Pyxl avoids support for logic within the HTML flow, except for one case where we found it especially useful: conditionally rendering HTML. That is why Pyxl provides the `<if>` tag, which takes an attr called `cond`. Children of an `<if>` are only rendered if `cond` evaluates to True.

## Implementation Details

### Parsing

Pyxl uses support for specifying source code encodings as described in [PEP 263](http://www.python.org/dev/peps/pep-0263/) to do what it does. The functionality was originally provided so that python developers could write code in non-ascii languages (eg. chinese variable names). Pyxl creates a custom encoding called pyxl which allows it to convert XML into regular python before the file is compiled. Once the pyxl codec is registered, any file starting with `# coding: pyxl` is run through the pyxl parser before compilation.

To register the pyxl codec, one must import the [`pyxl.codec.register`](https://github.com/awable/pyxl/blob/master/pyxl/pyxl/codec/register.py) module. The **Installation Process** makes it so that this always happens at python startup via the final `sudo python finish_install.py` step. What this step is doing is adding a file called `pyxl.pth` in your python site-packages directory, which imports the `pyxl.codec.register` module. Anything with a `.pth` extension in the site-packages directory is run automatically at python startup. Read more about that [here](http://docs.python.org/library/site.html).

Some people may prefer avoiding adding pyxl.pth to their site-packages directory, in which case they should skip the final step of the installation process and explicitly import `pyxl.codec.register` in the entry point of their application.

The pyxl encoding is a wrapper around utf-8, but every time it encounters a blob of HTML in the file, it runs it through python's [`HTMLParser`](http://docs.python.org/library/htmlparser.html) and replaces the HTML with python objects. As explained above, opening tags are converted into object instantiations for the respective tag, nested tags are passed in as arguments to the `append_children` method, and closing tags close the bracket to the `append_children` call. The code for these conversions can be seen [here](https://github.com/awable/pyxl/blob/master/pyxl/pyxl/codec/parser.py).

### HTML Objects

Though the syntactic sugar of being able to write HTML in python is pyxl's biggest usefulness, pyxl does also provide a basic framework for dealing with HTML tags as objects. This is not a full DOM implementation, but provides most of the necessary functionality. All the basic HTML tags are represented by objects defined in the [`pyxl.html`](https://github.com/awable/pyxl/blob/master/pyxl/pyxl/html.py) module, all of which inherit from the [`x_base`](https://github.com/awable/pyxl/blob/master/pyxl/pyxl/base.py) class.

An HTML tag is rendered by calling the `to_string()` method (called automatically when tags are cast to strings), which recursively calls `to_string()` on all its children. Therefore, it should be noted that almost all the work happens only once `to_string()` is called. It is also at this stage where attribute values and data is escaped. Most of the work consists of string concatenations, and performance based on applications we've written is equivalent to templating engines like Cheetah. Note that there is probably some low hanging fruit in performance improvements that we haven't looked in to (mostly because it hasn't been a problem).

## Editor Support

### Emacs

Grab pyxl-mode.el from the downloaded package under `pyxl/emacs/pyxl-mode.el` or copy it from [here](https://github.com/awable/pyxl/blob/master/emacs/pyxl-mode.el). To install, drop the file anywhere on your load path, and add the following to your ~/.emacs file (GNU Emacs) or ~/.xemacs/init.el file (XEmacs):

    (autoload 'pyxl-mode "pyxl-mode" "Major mode for editing pyxl" t)
    (setq auto-mode-alist
         (cons '("\\.py\\'" . pyxl-mode) auto-mode-alist))
