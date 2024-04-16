# SSVGG
The **S**nappy **SVG** **G**UI!

Inspired by [Dotgrid](https://100r.co/site/dotgrid.html) and [GodSVG](https://github.com/MewPurPur/GodSVG)

Think of it as a Dotgrid++

## TODO
- [ ] feature parity with Dotgrid
  - [x] grid
    - [x] visibility toggle
  - [ ] draw paths in canvas - **IN PROGRESS**
    - [x] render on canvas
    - [ ] render as SVG image instead of canvas commands
    - [ ] render drag points
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
  - [ ] modifiable spacing between grid markers
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
