<script>
  import { onMount } from "svelte";
  import { dev } from "$app/environment";
  import * as keybinds from "$lib/keybinds.js";

  let // bound to the window size
    innerWidth,
    innerHeight,
    // canvas html tag properties
    /** @type {HTMLCanvasElement} */ canvas,
    /** @type {CanvasRenderingContext2D} */ context,
    /** @type {DOMRect} */ rect,
    background = "#eee",
    // settings & state
    grid = { x: 0, y: 0, shown: true },
    scale = { val: 1, min: 1, max: 5 },
    spacing = { base: 15, scaled: 15 * scale.val },
    mouse = { x: 0, y: 0 },
    cursor = { x: 1, y: 1 },
    click = { x: null, y: null, button: null, held: null },
    ctrl_down = false,
    preview_points = [],
    // exported svg properties
    // svg html element
    svg,
    // svg as a string
    serialized_svg,
    // svg as a URL to an image
    rendered_svg,
    layers = [{ path: null, points: [] }],
    current_layer = 0,
    // default layer properties
    fill = "none",
    stroke = "#000";

  $: active_keybinds = ctrl_down ? keybinds.ctrl_layer : keybinds.base_layer;

  onMount(() => {
    context = canvas.getContext("2d");
    // initialize first path for the svg
    layers[current_layer].path = document.createElementNS(
      "http://www.w3.org/2000/svg",
      "path",
    );
    layers[current_layer].path.setAttribute("fill", fill);
    layers[current_layer].path.setAttribute("stroke", stroke);
    layers[current_layer].path.setAttribute("d", "");
    // initialize svg
    svg = document.createElementNS("http://www.w3.org/2000/svg", "svg");
    svg.setAttribute("xmlns", "http://www.w3.org/2000/svg");
    svg.appendChild(layers[current_layer].path);
  });

  /**
   * runs when canvas var is binded to <canvas>
   * or if anything messes with:
   * window size, canvas size, grid marker count/visibility/scale
   */
  $: if (canvas && (innerWidth || innerHeight || grid || scale)) {
    rect = canvas.getBoundingClientRect();
    context.clearRect(0, 0, canvas.width, canvas.height);
    draw();
  }

  /**
   * sets the canvas dimensions and draws everything
   * grid is drawn unless `grid.shown` is false
   */
  function draw() {
    canvas.width = rect.width;
    canvas.height = rect.height;
    spacing.scaled = spacing.base * scale.val;
    grid.x = Math.trunc(canvas.width / spacing.scaled);
    grid.y = Math.trunc(canvas.height / spacing.scaled);
    svg.setAttribute("viewBox", `0 0 ${grid.x} ${grid.y}`);

    drawGrid();
    drawRender();
    drawPreviewPoints();
    drawPlacedPoints();
    drawCursor();
  }

  function drawGrid() {
    if (!grid.shown) return;

    context.beginPath();
    for (let x = 1; x < grid.x; x++) {
      for (let y = 1; y < grid.y; y++) {
        const thick_dot = x % 4 === 0 && y % 4 === 0;
        const pos = { x: x * spacing.scaled, y: y * spacing.scaled };
        const radius = thick_dot ? 2 : 1;
        context.moveTo(pos.x, pos.y);
        context.arc(pos.x, pos.y, radius * scale.val, 0, 2 * Math.PI, false);
      }
    }
    context.fillStyle = "darkgrey";
    context.fill();
    context.closePath();
  }

  function drawRender() {
    if (!rendered_svg) {
      serialized_svg = new XMLSerializer().serializeToString(svg);
      const url = URL.createObjectURL(
        new Blob([serialized_svg], {
          type: "image/svg+xml;charset=utf-8",
        }),
      );
      rendered_svg = new Image();
      rendered_svg.onload = function () {
        context.drawImage(
          rendered_svg,
          0,
          0,
          grid.x * spacing.scaled,
          grid.y * spacing.scaled,
        );
        URL.revokeObjectURL(url);
      };
      rendered_svg.src = url;
    } else {
      context.drawImage(
        rendered_svg,
        0,
        0,
        grid.x * spacing.scaled,
        grid.y * spacing.scaled,
      );
    }
  }

  function drawPreviewPoints() {
    context.beginPath();
    for (const point of preview_points) {
      const pos = { x: point.x * spacing.scaled, y: point.y * spacing.scaled };
      context.moveTo(pos.x, pos.y);
      context.arc(pos.x, pos.y, 4 * scale.val, 0, 2 * Math.PI, false);
    }
    context.fillStyle = "dimgrey";
    context.fill();
    context.closePath();
  }

  function drawPlacedPoints() {
    if (!grid.shown) return;

    for (const point of layers[current_layer].points) {
      const pos = { x: point.x * spacing.scaled, y: point.y * spacing.scaled };
      context.beginPath();
      context.arc(pos.x, pos.y, 6 * scale.val, 0, 2 * Math.PI, false);
      context.fillStyle = "black";
      context.fill();
      context.closePath();
      context.beginPath();
      context.arc(pos.x, pos.y, 2 * scale.val, 0, 2 * Math.PI, false);
      context.fillStyle = "white";
      context.fill();
      context.closePath();
    }
  }

  function drawCursor() {
    context.beginPath();
    context.lineWidth = scale.val;
    context.arc(
      cursor.x * spacing.scaled,
      cursor.y * spacing.scaled,
      spacing.scaled / 2,
      0,
      2 * Math.PI,
      false,
    );
    context.stroke();
    context.closePath();
  }

  /**
   * moves the canvas cursor to the closest marker
   */
  function mousemove(event) {
    mouse.x = Math.trunc(event.clientX - rect.left);
    mouse.y = Math.trunc(event.clientY - rect.top);

    let x = Math.round(mouse.x / spacing.scaled);
    let y = Math.round(mouse.y / spacing.scaled);

    if (x >= grid.x) x = grid.x - 1;
    else if (x <= 0) x = 1;
    if (y >= grid.y) y = grid.y - 1;
    else if (y <= 0) y = 1;

    if (cursor.x !== x || cursor.y !== y) {
      cursor.x = x;
      cursor.y = y;
      draw();
    }
  }

  /**
   * primary   - hold an existing point if on top of one
   * middle    - drag grid around if zoomed in(maybe)
   * secondary - select point to delete if on top of one
   */
  function mousedown(event) {
    if (click.button) return;
    click.button = event.button;

    switch (event.button) {
      case 0:
        setHeldPoint();
        break;
      case 1:
        break;
      case 2:
        setHeldPoint();
        break;
    }
  }

  function setHeldPoint() {
    click.x = cursor.x;
    click.y = cursor.y;

    for (const point of layers[current_layer].points) {
      if (point.x === cursor.x && point.y === cursor.y) {
        click.held = point;
      }
    }
  }

  /**
   * primary   - place a new point or drop held point
   * middle    - stop dragging grid(maybe)
   * secondary - delete point
   */
  function mouseup(event) {
    if (click.button !== event.button) return;
    click.button = null;

    switch (event.button) {
      case 0:
        if (
          click.held === null ||
          (click.held.x === cursor.x && click.held.y === cursor.y)
        ) {
          addPreviewPoint();
        } else {
          moveHeldPoint();
        }
        break;
      case 1:
        break;
      case 2:
        if (
          click.held &&
          click.held.x === cursor.x &&
          click.held.y === cursor.y
        ) {
          deleteHeldPoint();
        }
        break;
    }
    click.held = null;
  }

  function addPreviewPoint() {
    preview_points.push({ x: cursor.x, y: cursor.y });

    // draws the latest point without refreshing everything
    context.beginPath();
    context.moveTo(cursor.x * spacing.scaled, cursor.y * spacing.scaled);
    context.arc(
      cursor.x * spacing.scaled,
      cursor.y * spacing.scaled,
      4 * scale.val,
      0,
      2 * Math.PI,
      false,
    );
    context.fillStyle = "dimgrey";
    context.fill();
    context.closePath();
  }

  function moveHeldPoint() {
    click.held.x = cursor.x;
    click.held.y = cursor.y;
    setPath();
    draw();
  }

  function deleteHeldPoint() {
    const points = layers[current_layer].points;
    const i = points.indexOf(click.held);

    switch (points[i].type.charAt(0)) {
      case "M":
        if (points[i + 1]) {
          if (points[i + 1].type.charAt(0) === "Q") {
            points.splice(i, 2); // remove "M", "Q1"
          } else if (points[i + 1].type.charAt(0) === "C") {
            points.splice(i, 3); // remove "M", "C1", "C2"
          } else {
            points.splice(i, 1); // remove "M"
          }
          points[i].type = "M"; // make sure no path starts without an "M"
        } else {
          points.splice(i, 1); // remove "M"
        }
        break;
      case "Q":
      case "C":
        if (points[i].type.charAt(1) !== "0") return;
        if (points[i].type.charAt(0) === "Q") {
          points.splice(i - 1, 2);
        } else {
          points.splice(i - 2, 3);
        }
        break;
      default:
        points.splice(i, 1);
    }
    setPath();
    draw();
  }

  /**
   * zoom grid in and out
   * `scale.val` mul'd by 10 and +/- 1 before being div'd by 10
   * to avoid floating-point presicion issues from `[+-]= 0.1`
   */
  function wheel(event) {
    if (event.deltaY < 0 && scale.val < scale.max) {
      scale.val = Math.trunc(scale.val * 10 + 1) / 10;
    } else if (event.deltaY > 0 && scale.val > scale.min) {
      scale.val = Math.trunc(scale.val * 10 - 1) / 10;
    }

    rendered_svg = null;
  }

  /**
   * also used when buttons are clicked
   * runs other methods based on which key pressed
   */
  function keydown(event) {
    if (event.repeat) return;
    ctrl_down = event.ctrlKey;

    if (event.ctrlKey) calcCtrlLayerKeys(event);
    else calcBaseLayerKeys(event);
  }

  function calcBaseLayerKeys(event) {
    switch (keybinds.base_layer[event.key]) {
      case "linecap":
      case "linejoin":
        updateStrokeEnds(keybinds.base_layer[event.key]);
        break;
      case "line":
        if (preview_points.length < 2) break;
        setPathType("L");
        break;
      case "arc":
        if (preview_points.length < 2) break;
        setPathType("A1");
        break;
      case "arc rev":
        if (preview_points.length < 2) break;
        setPathType("A0");
        break;
      case "bezier quad":
        if (preview_points.length < 3 || preview_points.length % 2 === 0) break;
        setPathType("Q");
        break;
      case "bezier cube":
        if (preview_points.length < 4 || (preview_points.length - 1) % 3 !== 0)
          break;
        setPathType("C");
        break;
    }

    setPath();
    draw();
  }

  /**
   * cycles through different linejoin or linecap types:
   * linejoins: miter | round | bevel
   * linecaps: butt | round | square
   * default linejoin gets a stroke-miterlimit attribute
   */
  function updateStrokeEnds(attr) {
    const second_type = attr === "linecap" ? "square" : "bevel";

    switch (layers[current_layer].path.getAttribute(`stroke-${attr}`)) {
      case "square": // default is "butt" so remove attr when switching to butt
      case "bevel": // default is "miter" so remove attr when switching to miter
        layers[current_layer].path.removeAttribute(`stroke-${attr}`);
        break;
      case "round":
        layers[current_layer].path.setAttribute(`stroke-${attr}`, second_type);
        break;
      case "butt":
      case "miter":
      default:
        layers[current_layer].path.setAttribute(`stroke-${attr}`, "round");
        break;
    }
  }

  function setPathType(type) {
    preview_points[0].type = "M";

    switch (type) {
      case "L":
      case "A1":
      case "A0":
        for (let i = 1; i < preview_points.length; i++) {
          preview_points[i].type = type;
        }
        break;
      case "Q":
        for (let i = 1; i < preview_points.length; i++) {
          // Q0 = end point
          // Q1 = control point
          preview_points[i].type = "Q" + (i % 2);
        }
        break;
      case "C":
        for (let i = 1; i < preview_points.length; i++) {
          // C0 = end point
          // C1/C2 = first/second control point
          preview_points[i].type = "C" + (i % 3);
        }
        break;
    }

    layers[current_layer].points.push(...preview_points);
    preview_points.length = 0;
  }

  function setPath() {
    let new_d = "";
    for (let i = 0; i < layers[current_layer].points.length; i++) {
      const point = layers[current_layer].points[i];
      switch (point.type) {
        case "M":
        case "L":
          new_d += `${point.type}${point.x} ${point.y}`;
          break;
        case "A1":
        case "A0":
          new_d += `A${Math.abs(
            point.x - layers[current_layer].points[i - 1].x,
          )} ${Math.abs(
            point.y - layers[current_layer].points[i - 1].y,
          )} 0 0 ${point.type.charAt(1)} ${point.x} ${point.y}`;
          break;
        case "Q0":
        case "Q1":
        case "C0":
        case "C1":
        case "C2":
          // only put a path type on the first control point (C1 or Q1)
          new_d += `${
            Number(point.type.charAt(1)) === 1 ? point.type.charAt(0) : " "
          }${point.x} ${point.y}`;
          break;
      }
    }
    layers[current_layer].path.setAttribute("d", new_d);
    rendered_svg = null;
  }

  function calcCtrlLayerKeys(event) {
    switch (keybinds.ctrl_layer[event.key]) {
      case "png":
      case "svg":
      case "webp":
        downloadImage(keybinds.ctrl_layer[event.key]);
        break;
      case "preview render":
        grid.shown = !grid.shown;
        break;
      // case keybinds.ctrl_layer.undo:
      //   undo();
      //   break;
      // case keybinds.ctrl_layer.redo:
      //   redo();
      //   break;
    }
  }

  function downloadImage(filetype, width, height) {
    const d = new Date();
    let timestamp = `${d.getFullYear()}`;
    timestamp += `-${String(d.getMonth() + 1).padStart(2, "0")}`;
    timestamp += `-${String(d.getDate()).padStart(2, "0")}`;
    timestamp += `_${String(d.getHours()).padStart(2, "0")}`;
    timestamp += `-${String(d.getMinutes()).padStart(2, "0")}`;
    timestamp += `-${String(d.getSeconds()).padStart(2, "0")}`;

    let url;

    if (filetype === "svg") {
      url = URL.createObjectURL(
        new Blob([serialized_svg], { type: "image/svg+xml;charset=utf-8" }),
      );
    } else {
      url = rasterizeSVG(filetype, width, height);
    }

    const link = document.createElement("a");
    link.setAttribute("download", `ssvgg_${timestamp}.${filetype}`);
    link.setAttribute("href", url);
    link.click();
    URL.revokeObjectURL(url);
  }

  function rasterizeSVG(filetype, width, height) {
    const cnv = document.createElement("canvas");
    cnv.width = grid.x * spacing.scaled;
    cnv.height = grid.y * spacing.scaled;

    cnv
      .getContext("2d")
      .drawImage(
        rendered_svg,
        0,
        0,
        grid.x * spacing.scaled,
        grid.y * spacing.scaled,
      );

    return cnv.toDataURL(`image/${filetype}`);
  }

  function keyup(event) {
    ctrl_down = event.ctrlKey;
  }
</script>

<svelte:window
  bind:innerHeight
  bind:innerWidth
  on:keydown|preventDefault={keydown}
  on:keyup|preventDefault={keyup}
/>

<!-- TODO implement layers -->
<div id="container">
  <canvas
    id="canvas"
    style:background
    bind:this={canvas}
    on:mousemove={mousemove}
    on:mousedown={mousedown}
    on:mouseup={mouseup}
    on:wheel|preventDefault={wheel}
    on:contextmenu|preventDefault
  />

  <div id="actions" style:background>
    <div id="how_to">
      <h2>HOW-TO</h2>
      <ol>
        <li>place two or more points on the canvas with left click</li>
        <li>
          select one of the buttons or press the corresponding key to draw a
          path based on the points
        </li>
      </ol>
    </div>

    <div id="keyboard">
      <!-- string represent left side of keyboard where all keybinds are -->
      {#each "qwertasdfgzxcvb".split("") as bind}
        {#if active_keybinds[bind]}
          <button
            on:click={() => {
              keydown({ key: bind, ctrlKey: ctrl_down });
            }}
          >
            {active_keybinds[bind]} ({bind.toUpperCase()})
          </button>
        {:else}
          <button style="font-size:1.5rem" disabled>
            {bind.toUpperCase()}
          </button>
        {/if}
      {/each}

      <button
        class:ctrl_down
        on:click={() => {
          ctrl_down = !ctrl_down;
        }}
      >
        CTRL
      </button>
    </div>

    <div id="mouse">
      <button id="mouse_left" disabled>place point</button>
      <button id="mouse_right" disabled>delete point</button>
      <button id="mouse_body" disabled>MOUSE</button>
    </div>

    {#if dev}
      <p style:margin="0">grid size: x: {grid.x}, y: {grid.y}</p>
      <p style:margin="0">mouse pos: x: {mouse.x}, y: {mouse.y}</p>
      <p style:margin="0">cursor pos: x: {cursor.x}, y: {cursor.y}</p>
      <p style:margin="0">scale: {scale.val}</p>

      {@html serialized_svg}
    {/if}
  </div>
</div>

<style>
  #container {
    width: calc(100vw - 1rem);
    height: calc(100vh - 1rem);
    margin: 0.5rem;
    gap: 0.5rem;
    display: grid;
    grid-template-rows: 1fr max-content;
  }
  #canvas {
    width: 100%;
    height: 100%;
    overflow: hidden;
  }
  #actions {
    width: 100%;
    height: 100%;
    display: flex;
    align-items: center;
    justify-content: space-evenly;
  }
  #how_to {
    max-width: 25%;
    height: 100%;
    font-family: sans-serif;
  }
  #keyboard {
    margin: 0.25rem;
    padding-right: 3rem;
    gap: 0.25rem;
    display: grid;
    grid-template-columns: repeat(5, 1fr);

    & button:nth-child(n + 6):nth-child(-n + 10) {
      margin-left: 1rem;
      margin-right: -1rem;
    }
    & button:nth-child(n + 11):nth-child(-n + 15) {
      margin-left: 3rem;
      margin-right: -3rem;
    }
  }
  #mouse {
    margin: 0.25rem;
    gap: 0.25rem;
    display: grid;
    grid-template-columns: 1fr 1fr;
  }
  #mouse_left {
    border-top-left-radius: 2rem;
  }
  #mouse_right {
    border-top-right-radius: 2rem;
  }
  #mouse_body {
    width: 8.25rem;
    height: 8.25rem;
    grid-column: span 2;
    border-bottom-left-radius: 4rem;
    border-bottom-right-radius: 4rem;
  }
  button {
    width: 4rem;
    height: 4rem;
    border: black 2px solid;
    border-radius: 0.5rem;

    &:hover {
      filter: brightness(0.9);
    }
    &:active {
      filter: brightness(0.8);
    }
    &:disabled {
      filter: brightness(0.7);
      border: darkgrey 1px solid;
    }
  }
  .ctrl_down {
    filter: brightness(0.8);
  }
</style>
