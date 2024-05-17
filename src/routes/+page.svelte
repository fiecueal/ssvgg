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
    // TODO: the setup with offset var sucks, change later
    // offset to make mouse pointer accurate to the position of the grid cursor circle
    offset = {
      x: mouse.x + spacing.scaled / 2,
      y: mouse.y + spacing.scaled / 2,
    },
    lastAction,
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
    // offset aligns the grid cursor to the mouse pointer
    context.arc(
      offset.x - (offset.x % spacing.scaled) + 0.5,
      offset.y - (offset.y % spacing.scaled) + 0.5,
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
    offset = {
      x: mouse.x + spacing.scaled / 2,
      y: mouse.y + spacing.scaled / 2,
    };

    const x = (offset.x - (offset.x % spacing.scaled)) / spacing.scaled;
    const y = (offset.y - (offset.y % spacing.scaled)) / spacing.scaled;

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
    switch (event.button) {
      case 0:
        click.x = cursor.x;
        click.y = cursor.y;

        for (const point of layers[current_layer].points) {
          if (point.x === cursor.x && point.y === cursor.y) {
            click.held = point;
          }
        }

        lastAction = "primary down";
        break;
      case 1:
        lastAction = "middle down";
        break;
      case 2:
        lastAction = "secondary down";
        break;
    }
  }

  /**
   * primary   - place a new point or drop held point
   * middle    - stop dragging grid(maybe)
   * secondary - delete point
   */
  function mouseup(event) {
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
        click.held = null;
        lastAction = "primary up";
        break;
      case 1:
        lastAction = "middle up";
        break;
      case 2:
        lastAction = "secondary up";
        break;
    }
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
   * https://100r.co/site/dotgrid.html#shortcut_quick_list
   */
  function keydown(event) {
    if (event.repeat) return;
    if (event.key === "Control") return (ctrl_down = true);

    if (event.ctrlKey) calcCtrlLayerKeys(event);
    else calcBaseLayerKeys(event);
  }

  function calcBaseLayerKeys(event) {
    switch (event.key) {
      case keybinds.base_layer.line:
        if (preview_points.length < 2) break;
        setPathType("L");
        break;
      case keybinds.base_layer.arc:
        if (preview_points.length < 2) break;
        setPathType("A1");
        break;
      case keybinds.base_layer.arc_rev:
        if (preview_points.length < 2) break;
        setPathType("A0");
        break;
      case keybinds.base_layer.bezier_quad:
        if (preview_points.length < 3 || preview_points.length % 2 === 0) break;
        setPathType("Q");
        break;
      case keybinds.base_layer.bezier_cube:
        if (preview_points.length < 4 || (preview_points.length - 1) % 3 !== 0)
          break;
        setPathType("C");
        break;
    }

    setPath();
    draw();
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
          preview_points[i].type = "Q" + (i % 2);
        }
        break;
      case "C":
        for (let i = 1; i < preview_points.length; i++) {
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
    switch (event.key) {
      case keybinds.ctrl_layer.png:
        downloadImage("png");
        break;
      case keybinds.ctrl_layer.svg:
        downloadImage("svg");
        break;
      case keybinds.ctrl_layer.webp:
        downloadImage("webp");
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
    if (event.key === "Control") ctrl_down = false;
  }
</script>

<svelte:window
  bind:innerHeight
  bind:innerWidth
  on:keydown|preventDefault={keydown}
  on:keyup|preventDefault={keyup}
/>

<!--
width and height style (actual dimensions on client screen)
differ from width and height attribute(set in draw() function)
which is used to determine the number of grid markers to show on screen
-->
<div id="container">
  <div style="grid-area:layers" style:background>add stuff here later</div>

  <canvas
    style="grid-area:canvas;width:100%;height:100%"
    style:background
    bind:this={canvas}
    on:mousemove={mousemove}
    on:mousedown={mousedown}
    on:mouseup={mouseup}
    on:wheel|preventDefault={wheel}
    on:contextmenu|preventDefault
  />

  <div style="grid-area:options" style:background>
    {#if ctrl_down}
      {#each Object.entries(keybinds.ctrl_layer) as [key, val]}
        <button
          on:click={() => {
            keydown({ key: val, ctrlKey: true });
          }}
        >
          {key} ({val.toUpperCase()})
        </button>
      {/each}
    {:else}
      {#each Object.entries(keybinds.base_layer) as [key, val]}
        <button
          on:click={() => {
            // runs the same function as a keydown event so uses the same method
            keydown({ key: val });
          }}
        >
          {key} ({val.toUpperCase()})
        </button>
      {/each}
    {/if}
    <button
      on:click={() => {
        grid.shown = !grid.shown;
      }}
    >
      show grid: {grid.shown}
    </button>

    {#if dev}
      <p style:margin="0">grid size: x: {grid.x}, y: {grid.y}</p>
      <p style:margin="0">mouse pos: x: {mouse.x}, y: {mouse.y}</p>
      <p style:margin="0">cursor pos: x: {cursor.x}, y: {cursor.y}</p>
      <p style:margin="0">last action: {lastAction}</p>
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
    grid-template-areas:
      "layers canvas"
      "options options";
    grid-template-columns: 15rem 1fr;
    grid-template-rows: 1fr 25rem;
  }
</style>
