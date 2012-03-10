# org.timmc/cascade

Manage state dirtiness dependency cascades.

## Get it

1. Leiningen dependency: `[org.timmc/cascade "0.1.0"]`
2. Add this to your ns block: `(:require [org.timmc.cascade :as cascade])`

## Purpose

Let's say you have a GUI with an interactive canvas and a collection
of related stateful control widgets, such as a rotation slider and a
zoom slider.  The user can drag objects around on the canvas, which
requires redrawing the canvas image, but a redraw will also be
required when the canvas is dragged or the rotate/zoom sliders are
altered. The sliders may be altered by a "best fit" button, and the
user's data objects (shown on the canvas) may be manipulated by the
undo/redo menu items. Resizing the window and hovering over canvas
elements will have further effects. Perhaps your dependency lists
look like this:

* :pose [] - zoom / rotate / translate values (based on controls)
* :dim [] - window dimensions
* :xform [:pose :dim] - graphics transform matrix
* :udata [] - user data (objects on canvas)
* :hover [] - what the user is hovering over on the canvas
* :painting [:udata :xform :hover] - image data buffer
* :mode [:udata] - interaction mode based on what existing data
* et cetera...

Chaining the various dependent state updates is a bug-prone exercise
in frustration. org.timmc/cascade allows you to define your directed
acyclic graph of dependencies as a persistent "cascade" data structure,
storing a "dirty" or "clean" indicator for each node. The example below
is taken from the Bézier curve editor that originally spawned this library.

    (cascade/create
      :canvas-shape nil false ;; shape and size of canvas component
      :center update-canvas-shape! [:canvas-shape]
      :pose-spinners nil false ;; data in JSpinners
      :pose update-pose! [:pose-spinners]
      :mouse-pos nil false ;; mouse-pos in @state
      :udata nil false ;; pretty much anything in @udata
      :hover update-hover! [:mouse-pos :udata]
      :xform update-xform! [:pose :center]
      :mode update-mode! [:udata]
      :menustate update-menus! [:mode]
      :history-gui update-history-gui! true ;; undo/redo was committed
      ;; top-level states
      :painting ask-redraw [:udata :xform :hover]
      :gui nil [:menustate :history-gui]
      ;; collector
      :all nil [:painting :gui])

Each node declares a side-effecting cleaner thunk (or nil) and either
a list of dependency nodes or a boolean indicating whether the node is
already clean. If the resulting data structure is stored in an atom `c`,
relying code can call `(swap! c cascade/dirty :center)` to indicate that
the canvas has been dragged, and then `(swap! c cascade/clean :gui)` to
recalculate all (and only) the relying nodes from :gui to :center.

(Note that if a cleaner thunk causes an error, the swap! will fail,
and the cascade value will remain the same.)

See the main namespace docstring for more info: `org.timmc.cascade`

## License

Copyright Tim McCormack 2011-2012.

Free-licensed under the Eclipse Public License v1.0 (see ./epl-v10.html).
