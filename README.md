# SSVGG
The **S**nappy **SVG** **G**UI!

Inspired by [Dotgrid](https://100r.co/site/dotgrid.html) and [GodSVG](https://github.com/MewPurPur/GodSVG)

## TODO
- [ ] feature parity with Dotgrid
  - [x] grid
    - [x] visibility toggle
  - [x] grid-snapped cursor
  - [ ] left click
    - [x] place a new point
    - [ ] drag to reposition point
    - [ ] ctrl+drag to copy a path
    - [ ] shift+drag to move a path
  - [ ] right click to delete point
  - [ ] strokes
    - [x] lines
    - [ ] arc
    - [ ] arc rev
    - [ ] bezier
    - [ ] arc(full)
    - [ ] arc rev(full)
    - [ ] clear selection
    - [ ] erase segment
  - [ ] draw paths in canvas - **IN PROGRESS**
  - [ ] layers
  - [ ] style
    - [ ] close
    - [ ] linecap
    - [ ] linejoin
    - [ ] thickness
    - [ ] fill
    - [ ] mirror
  - [ ] color picker
  - [ ] keybinds
  - [ ] undo/redo
  - [ ] export svg/png/custom `.grid` format
  - [ ] import `.grid`
  - [ ] canvas theming & palettes
- [ ] features beyond Dotgrid
  - [ ] opacity for paths
  - [ ] zooming
    - [x] zoomable grid(without zooming the browser window)
    - [ ] draggable grid when zoomed
    - [ ] zoom centered on current cursor position
  - [ ] resizable canvas(without resizing the browser window)
  - [ ] better handling of overlapping points
  - [ ] remove layer cap
  - [ ] edit existing paths individually
  - [ ] un/group paths
  - [ ] more stroke/path options
    - [ ] use other tags like `<rect>` and `<circle>`
    - [ ] `H` and `V` for cardinal lines in exported svg
  - [ ] language-agnostic icons instead of words in menu(maybe)
  - [ ] better `.grid` format(maybe compatibility with Dotgrid's `.grid` too)

## Why?
Dotgrid is an amazing tool for making simple SVG icons but the exports are also bigger than they need to be, there are a few bothersome bugs, and there are features in convenience and functionality that I'm looking for. I feel that a lot of these additions go against Hundred Rabbit's philosophy so I'm starting from scratch instead. GodSVG is another spin on a simple SVG editor that gives more control and optimizations to the exported SVG files. The latest version(latest commit, no release yet) feels nice to use and is capable of so much more but doesn't have the simplistic feel of Dotgrid which I like more. It also doesn't have the convenience of just placing points and hitting a keybind to quickly draw a path. GodSVG is a great tool but I'm looking for an extension of Dotgrid's functionality, not an overhaul.

Another reason why I'm making this is because I don't know how to work with Canvas API and want to learn so this seemed like a fun undertaking. But I hope I can make this at least better than Dotgrid.
