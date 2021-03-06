.. _full_screen_applications:

Building full screen applications
=================================

`prompt_toolkit` can be used to create complex full screen terminal
applications. Typically, an application consists of a layout (to describe the
graphical part) and a set of key bindings.

The sections below describe the components required for full screen
applications (or custom, non full screen applications), and how to assemble
them together.

Before going through this page, it could be helpful to go through :ref:`asking
for input <asking_for_input>` (prompts) first. Many things that apply to an
input prompt also apply to full screen applications.

.. note::

    Also remember that the ``examples`` directory of the prompt_toolkit
    application contains plenty of examples. Each example is supposed to
    explain one idea. So, this as well should help you get started.

    Don't hesitate to open a GitHub issue if you feel that a certain example is
    missing.


A simple application
--------------------

Every prompt_toolkit application is an instance of an
:class:`~prompt_toolkit.application.Application` object. The simplest full
screen example would look like this:

.. code:: python

    from prompt_toolkit import Application

    app = Application(full_screen=True)
    app.run()

This will display a dummy application that says "No layout specified. Press
ENTER to quit.".

.. note::

        If we wouldn't set the ``full_screen`` option, the application would
        not run in the alternate screen buffer, and only consume the least
        amount of space required for the layout.

An application consists of several components. The most important are:

- I/O objects: the input and output device.
- The layout: this defines the graphical structure of the application. For
  instance, a text box on the left side, and a button on the right side.
  You can also think of the layout as a collection of 'widgets'.
- A style: this defines what colors and underline/bold/italic styles are used
  everywhere.
- A set of key bindings.

We will discuss all of these in more detail them below.


I/O objects
-----------

Every :class:`~prompt_toolkit.application.Application` instance requires an I/O
objects for input and output:

    - An :class:`~prompt_toolkit.input.base.Input` instance, which is an
      abstraction of the input stream (stdin).
    - An :class:`~prompt_toolkit.output.base.Output` instance, which is an
      abstraction of the output stream, and is called by the renderer.

Both are optional and normally not needed to pass explicitely. Usually, the
default works fine.

There is a third I/O object which is also required by the application, but not
passed inside. This is the event loop, an
:class:`~prompt_toolkit.eventloop.base.EventLoop` instance. This is basically a
while-true loop that waits for user input, and when it receives something (like
a key press), it will send that to the the appropriate handler, like for
instance, a key binding.

When :func:`~prompt_toolkit.application.Application.run()` is called, the event
loop will run until the application is done.


The layout
----------

A layered layout architecture
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

There are several ways to create a prompt_toolkit layout, depending on how
customizable you want things to be. In fact, there are several layers of
abstraction.

- The most low-level way of creating a layout is by combining
  :class:`~prompt_toolkit.layout.containers.Container` and
  :class:`~prompt_toolkit.layout.controls.UIControl` objects.

  Examples of :class:`~prompt_toolkit.layout.containers.Container` objects are
  :class:`~prompt_toolkit.layout.containers.VSplit` (vertical split),
  :class:`~prompt_toolkit.layout.containers.HSplit` (horizontal split) and
  :class:`~prompt_toolkit.layout.containers.FloatContainer`. These containers
  arrange the layout and can split it in multiple regions. Each container can
  recursively contain multiple other containers. They can be combined in any
  way to define the "shape" of the layout.

  The :class:`~prompt_toolkit.layout.containers.Window` object is a special
  kind of container that can contain
  :class:`~prompt_toolkit.layout.controls.UIControl` object. The
  :class:`~prompt_toolkit.layout.controls.UIControl` object is responsible for
  the generation of the actual content. The
  :class:`~prompt_toolkit.layout.containers.Window` object acts as an adaptor
  between the :class:`~prompt_toolkit.layout.controls.UIControl` and other
  containers, but it's also responsible for the scrolling and line wrapping of
  the content.

  Examples of :class:`~prompt_toolkit.layout.controls.UIControl` objects are
  :class:`~prompt_toolkit.layout.controls.BufferControl` for showing the
  content of an editable/scrollable buffer, and
  :class:`~prompt_toolkit.layout.controls.FormattedTextControl` for displaying
  (:ref:`formatted <formatted_text>`) text.

- A higher level abstraction of building a layout is by using "widgets". A
  widget is a reusable layout component that can contain multiple containers
  and controls. It should have a ``__pt__container__`` function, which is
  supposed to return the root container for this widget. Prompt_toolkit
  contains a couple of widgets like
  :class:`~prompt_toolkit.layout.widgets.base.TextArea`,
  :class:`~prompt_toolkit.layout.widgets.base.Button`,
  :class:`~prompt_toolkit.layout.widgets.base.Frame`,
  :class:`~prompt_toolkit.layout.widgets.base.VerticalLine` and so on.

- The highest level abstractions can be found in the ``shortcuts`` module.
  There we don't have to think about the layout, controls and containers at
  all. This is the simplest way to use prompt_toolkit, but is only meant for
  specific use cases, like a prompt or a simple dialog window.

Containers and controls
^^^^^^^^^^^^^^^^^^^^^^^

The biggest difference between containers and controls is that containers
arrange the layout by splitting the screen in many regions, while controls are
responsible for generating the actual content.

.. note::

   Under the hood, the difference is:

   - containers use *absolute coordinates*, and paint on a
     :class:`~prompt_toolkit.layout.screen.Screen` instance.
   - user controls create a :class:`~prompt_toolkit.layout.controls.UIContent`
     instance. This is a collection of lines that represent the actual
     content. A :class:`~prompt_toolkit.layout.controls.UIControl` is not aware
     of the screen.

+------------------------------------------------------+---------------------------------------------------------------+
| Abstract base class                                  | Examples                                                      |
+======================================================+===============================================================+
| :class:`~prompt_toolkit.layout.containers.Container` | :class:`~prompt_toolkit.layout.containers.HSplit`             |
|                                                      | :class:`~prompt_toolkit.layout.containers.VSplit`             |
|                                                      | :class:`~prompt_toolkit.layout.containers.FloatContainer`     |
|                                                      | :class:`~prompt_toolkit.layout.containers.Window`             |
+------------------------------------------------------+---------------------------------------------------------------+
| :class:`~prompt_toolkit.layout.controls.UIControl`   | :class:`~prompt_toolkit.layout.controls.BufferControl`        |
|                                                      | :class:`~prompt_toolkit.layout.controls.FormattedTextControl` |
+------------------------------------------------------+---------------------------------------------------------------+

The :class:`~prompt_toolkit.layout.containers.Window` class itself is
particular: it is a :class:`~prompt_toolkit.layout.containers.Container` that
can contain a :class:`~prompt_toolkit.layout.controls.UIControl`. Thus, it's
the adaptor between the two. The
:class:`~prompt_toolkit.layout.containers.Window` class also takes care of
scrolling the content and wrapping the lines if needed.

Finally, there is the :class:`~prompt_toolkit.layout.Layout` class which wraps
the whole layout. This is responsible for keeping track of which window has the
focus.

Here is an example of a layout that displays the content of the default buffer
on the left, and displays ``"Hello world"`` on the right. In between it shows a
vertical line:

.. code:: python

    from prompt_toolkit import Application
    from prompt_toolkit.buffer import Buffer
    from prompt_toolkit.layout.containers import VSplit, Window
    from prompt_toolkit.layout.controls import BufferControl, FormattedTextControl

    buffer1 = Buffer()  # Editable buffer.

    root_container = VSplit([
        # One window that holds the BufferControl with the default buffer on
        # the left.
        Window(content=BufferControl(buffer=buffer1)),

        # A vertical line in the middle. We explicitely specify the width, to
        # make sure that the layout engine will not try to divide the whole
        # width by three for all these windows. The window will simply fill its
        # content by repeating this character.
        Window(width=1, char='|'),

        # Display the text 'Hello world' on the right.
        Window(content=FormattedTextControl(text='Hello world')),
    ])

    layout = Layout(root_container)

    app = Application(layout=layout, full_screen=True)
    app.run()


More complex layouts can be achieved by nesting multiple 
:class:`~prompt_toolkit.layout.containers.VSplit`,
:class:`~prompt_toolkit.layout.containers.HSplit` and 
:class:`~prompt_toolkit.layout.containers.FloatContainer` objects.

If you want to make some part of the layout only visible when a certain
condition is satisfied, use a
:class:`~prompt_toolkit.layout.containers.ConditionalContainer`.


Focussing windows
^^^^^^^^^^^^^^^^^

TODO

Key bindings
------------

In order to react to user actions, we need to create a
:class:`~prompt_toolkit.key_binding.key_bindings.KeyBindings` object and pass
that to our :class:`~prompt_toolkit.application.Application`.

There are two kinds of key bindings:

- Global key bindings, which are always active.
- Key bindings that belong to a certain
  :class:`~prompt_toolkit.layout.controls.UIControl` and are only active when
  this control is focussed. Both
  :class:`~prompt_toolkit.layout.controls.BufferControl`
  :class:`~prompt_toolkit.layout.controls.FormattedTextControl` take a
  ``key_bindings`` argument.


Global key bindings
^^^^^^^^^^^^^^^^^^^

.. code:: python

    from prompt_toolkit import Application
    from prompt_toolkit.key_binding.key_bindings import KeyBindings

    kb = KeyBindings()
    app = Application(key_bindings=kb)
    app.run()

To register a new keyboard shortcut, we can use the
:meth:`~prompt_toolkit.key_binding.key_bindings.KeyBindings.add` method as a
decorator of the key handler:

.. code:: python

    from prompt_toolkit import Application
    from prompt_toolkit.key_binding.key_bindings import KeyBindings

    kb = KeyBindings()

    @kb.add('c-q')
    def exit_(event):
        """
        Pressing Ctrl-Q will exit the user interface.

        Setting a return value means: quit the event loop that drives the user
        interface and return this value from the `CommandLineInterface.run()` call.
        """
        event.app.set_return_value(None)

    app = Application(key_bindings=kb, full_screen=True)
    app.run()

The callback function is named ``exit_`` for clarity, but it could have been
named ``_`` (underscore) as well, because the we won't refer to this name.


Modal containers
^^^^^^^^^^^^^^^^

All container objects, like :class:`~prompt_toolkit.layout.containers.VSplit`
and :class:`~prompt_toolkit.layout.containers.HSplit` take a ``modal`` argument.

If this flag has been set, then key bindings from the parent account are not
taken into account if one of the childen windows has the focus.

This is useful in a complex layout, where many controls have their own key
bindings, but you only want to enable the key bindings for a certain region of
the layout.

The global key bindings are always active.


More about the Window class
---------------------------

As said earlier, a :class:`~prompt_toolkit.layout.containers.Window` is a
:class:`~prompt_toolkit.layout.containers.Container` that wraps a 
:class:`~prompt_toolkit.layout.controls.UIControl`, like a
:class:`~prompt_toolkit.layout.controls.BufferControl` or
:class:`~prompt_toolkit.layout.controls.FormattedTextControl`.

.. note::

    Basically, windows are the leafs in the tree structure that represent the UI.

A :class:`~prompt_toolkit.layout.containers.Window` provides a "view" on the
:class:`~prompt_toolkit.layout.controls.UIControl`, which provides lines of
content. The window is in the first place responsible for the line wrapping and
scrolling of the content, but there are much more options.

- Adding left or right margins. These are used for displaying scroll bars or
  line numbers.
- There are the `cursorline` and `cursorcolumn` options. These allow
  highlighting the line or column of the cursor position.
- Alignment of the content. The content can be left aligned, right aligned or
  centered.
- Finally, the background can be filled with a default character.


More about buffers and :class:`~prompt_toolkit.layout.controls.BufferControl`
-----------------------------------------------------------------------------



Input processors
^^^^^^^^^^^^^^^^

A :class:`~prompt_toolkit.layout.processors.Processor` is used to postprocess
the content of a :class:`~prompt_toolkit.layout.controls.BufferControl` before
it's displayed. It can for instance highlight matching brackets or change
the visualisation of tabs and so on.

A :class:`~prompt_toolkit.layout.processors.Processor` operates on individual
lines. Basically, it takes a (formatted) line and produces a new (formatted)
line.

Some build-in processors:

+----------------------------------------------------------------------------+-----------------------------------------------------------+
| Processor                                                                  | Usage:                                                    |
+============================================================================+===========================================================+
| :class:`~prompt_toolkit.layout.processors.HighlightSearchProcessor`        | Highlight the current search results.                     |
+----------------------------------------------------------------------------+-----------------------------------------------------------+
| :class:`~prompt_toolkit.layout.processors.HighlightSelectionProcessor`     | Highlight the selection.                                  |
+----------------------------------------------------------------------------+-----------------------------------------------------------+
| :class:`~prompt_toolkit.layout.processors.PasswordProcessor`               | Display input as asterisks. (``*`` characters).           |
+----------------------------------------------------------------------------+-----------------------------------------------------------+
| :class:`~prompt_toolkit.layout.processors.BracketsMismatchProcessor`       | Highlight open/close mismatches for brackets.             |
+----------------------------------------------------------------------------+-----------------------------------------------------------+
| :class:`~prompt_toolkit.layout.processors.BeforeInput`                     | Insert some text before.                                  |
+----------------------------------------------------------------------------+-----------------------------------------------------------+
| :class:`~prompt_toolkit.layout.processors.AfterInput`                      | Insert some text after.                                   |
+----------------------------------------------------------------------------+-----------------------------------------------------------+
| :class:`~prompt_toolkit.layout.processors.AppendAutoSuggestion`            | Append auto suggestion text.                              |
+----------------------------------------------------------------------------+-----------------------------------------------------------+
| :class:`~prompt_toolkit.layout.processors.ShowLeadingWhiteSpaceProcessor`  | Visualise leading whitespace.                             |
+----------------------------------------------------------------------------+-----------------------------------------------------------+
| :class:`~prompt_toolkit.layout.processors.ShowTrailingWhiteSpaceProcessor` | Visualise trailing whitespace.                            |
+----------------------------------------------------------------------------+-----------------------------------------------------------+
| :class:`~prompt_toolkit.layout.processors.TabsProcessor`                   | Visualise tabs as `n` spaces, or some symbols.            |
+----------------------------------------------------------------------------+-----------------------------------------------------------+

A :class:`~prompt_toolkit.layout.controls.BufferControl` takes only one
processor as input, but it is possible to "merge" multiple processors into one
with the :func:`~prompt_toolkit.layout.processors.merge_processors` function.
