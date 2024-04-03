<script>
  import { onMount } from "svelte";
  import { dev } from "$app/environment";
  import { keybinds } from "$lib/keybinds.js";

  let // bound to the window size
    innerWidth,
    innerHeight,
    // canvas html tag properties
    /** @type {HTMLCanvasElement} */ canvas,
    /** @type {CanvasRenderingContext2D} */ context,
    /** @type {DOMRect} */ rect,
    // canvas html tag width and height style, not attribute
    width = "80vw",
    height = "80vh",
    background = "#eee",
    // settings & state
    grid = { x: 0, y: 0, shown: true },
    scale = { val: 1, min: 1, max: 5 },
    spacing = { base: 15, scaled: 15 * scale.val },
    mouse = { x: 0, y: 0 },
    cursor = { x: 1, y: 1 },
    click = { x: null, y: null, button: null },
    // offset to make mouse pointer accurate to the position of the grid cursor circle
    offset = {
      x: mouse.x + spacing.scaled / 2,
      y: mouse.y + spacing.scaled / 2,
    },
    lastAction,
    points = [],
    paths = { lines: [] },
    // exported svg properties
    svg,
    fill = "none",
    stroke = "#000",
    viewBox = "",
    d = "";

  onMount(() => {
    svg = document.createElementNS("http://www.w3.org/2000/svg", "svg");
    context = canvas.getContext("2d");
  });

  /**
   * runs when canvas var is binded to <canvas>
   * or if anything messes with:
   * window size, canvas size, grid marker count/visibility/scale
   */
  $: if (
    canvas &&
    (innerWidth || innerHeight || width || height || grid || scale)
  ) {
    rect = canvas.getBoundingClientRect();
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

    drawGrid();
    drawRender();
    drawPoints();
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
    context.beginPath();
    for (const point of paths.lines) {
      if (point.m) {
        context.moveTo(point.x * spacing.scaled, point.y * spacing.scaled);
      } else {
        context.lineTo(point.x * spacing.scaled, point.y * spacing.scaled);
      }
    }
    context.lineWidth = spacing.scaled;
    context.fillStyle = fill;
    context.strokeStyle = stroke;
    context.stroke();
    context.closePath();
    // context.fill()
  }

  function drawPoints() {
    context.beginPath();
    for (const point of points) {
      const pos = { x: point.x * spacing.scaled, y: point.y * spacing.scaled };
      context.moveTo(pos.x, pos.y);
      context.arc(pos.x, pos.y, 4 * scale.val, 0, 2 * Math.PI, false);
    }
    context.fillStyle = "dimgrey";
    context.fill();
    context.closePath();
  }

  function drawCursor() {
    context.beginPath();
    context.lineWidth = scale.val;
    // offset aligns the grid cursor to the mouse pointer
    context.arc(
      offset.x - (offset.x % spacing.scaled),
      offset.y - (offset.y % spacing.scaled),
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
        click.button = 0;
        click.x = cursor.x;
        click.y = cursor.y;

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
        points.push({ x: cursor.x, y: cursor.y });

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
  }

  /**
   * runs other methods based on which key pressed
   * https://100r.co/site/dotgrid.html#shortcut_quick_list
   */
  function keydown(event) {
    if (event.repeat) return;

    switch (event.key) {
      case keybinds.line:
        if (points.length < 2) break;
        drawLines();
        break;
      case keybinds.fill:
        console.log("fill");
        break;
    }

    draw();
  }

  function drawLines() {
    let new_d = `M${points[0].x},${points[0].y}`;

    for (let i = 1; i < points.length; i++) {
      new_d += `L${points[i].x},${points[i].y}`;
    }

    points[0].m = true;
    paths.lines.push(...points);
    points.length = 0;
    d += new_d;
  }
</script>

<svelte:window
  bind:innerHeight
  bind:innerWidth
  on:keydown|preventDefault={keydown}
/>

<!--
width and height style (actual dimensions on client screen)
differ from width and height attribute(set in draw() function)
which is used to determine the number of grid markers to show on screen
-->
<canvas
  style="display: block; margin: auto"
  style:width
  style:height
  style:background
  bind:this={canvas}
  on:mousemove={mousemove}
  on:mousedown={mousedown}
  on:mouseup={mouseup}
  on:wheel|preventDefault={wheel}
  on:contextmenu|preventDefault
/>

{#each Object.entries(keybinds) as [key, val]}
  <button
    on:click={() => {
      keydown({ key: val });
    }}
  >
    {key} ({val.toUpperCase()})
  </button>
{/each}

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

  <svg
    {stroke}
    {fill}
    {width}
    {height}
    viewBox="0 0 {grid.x} {grid.y}"
    xmlns="http://www.w3.org/2000/svg"
  >
    <path {d} />
  </svg>
{/if}
