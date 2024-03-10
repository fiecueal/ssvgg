<script>
  import { dev } from "$app/environment";

  let // bound to the window size
    innerWidth,
    innerHeight,
    // canvas html tag properties
    /** @type {HTMLCanvasElement} */ canvas,
    /** @type {CanvasRenderingContext2D} */ context,
    /** @type {DOMRect} */ rect,
    width = "80vw",
    height = "80vh",
    background = "#eee",
    // settings & state
    grid = { x: 0, y: 0, shown: true },
    scale = { val: 1, min: 1, max: 5 },
    spacing = { base: 15, scaled: 15 * scale.val },
    mouse = { x: 0, y: 0 },
    cursor = { x: 1, y: 1 },
    // offset to make mouse pointer accurate to the position of the grid cursor circle
    offset = {
      x: mouse.x + spacing.scaled / 2,
      y: mouse.y + spacing.scaled / 2,
    },
    lastAction,
    points = [],
    segments = [],
    // exported svg properties
    fill = "none",
    stroke = "#000",
    viewBox = "";

  $: spacing.scaled = spacing.base * scale.val;
  $: offset = {
    x: mouse.x + spacing.scaled / 2,
    y: mouse.y + spacing.scaled / 2,
  };

  /**
   * runs when canvas var is binded to <canvas>
   * or if anything messes with:
   * window size, canvas size, grid marker count/visibility/scale
   */
  $: if (
    canvas &&
    (innerWidth || innerHeight || width || height || grid || scale || mouse)
  ) {
    context = canvas.getContext("2d");
    rect = canvas.getBoundingClientRect();
    draw();
  }

  /**
   * sets the canvas dimensions and draws everything
   * grid is drawn unless `grid.shown` is false
   */
  function draw() {
    canvas.width = rect.width
    canvas.height = rect.height
    grid.x = Math.trunc(canvas.width / spacing.base);
    grid.y = Math.trunc(canvas.height / spacing.base);

    // draw grid markers
    if (grid.shown) {
      context.beginPath();
      for (let x = 1; x < grid.x / scale.val; x++) {
        for (let y = 1; y < grid.y / scale.val; y++) {
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
    // end - draw grid markers

    // draw cursor
    context.beginPath();
    context.lineWidth = scale.val;
    // better aligns the drawn cursor to the mouse pointer
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
    // end - draw cursor
  }

  /**
   * moves the canvas cursor to the closest marker
   */
  function mousemove(event) {
    mouse.x = Math.trunc(event.clientX - rect.left);
    mouse.y = Math.trunc(event.clientY - rect.top);

    cursor.x = (offset.x - (offset.x % spacing.scaled)) / spacing.scaled;
    cursor.y = (offset.y - (offset.y % spacing.scaled)) / spacing.scaled;
  }

  /**
   * primary   - hold an existing point if on top of one
   * middle    - drag grid around if zoomed in(maybe)
   * secondary - select point to delete if on top of one
   */
  function mousedown(event) {
    if (dev) {
      switch (event.button) {
        case 0:
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
  }

  /**
   * primary   - place a new point or drop held point
   * middle    - stop dragging grid(maybe)
   * secondary - delete point
   */
  function mouseup(event) {
    if (dev) {
      switch (event.button) {
        case 0:
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
  function keydown(event) {}
</script>

<svelte:window bind:innerHeight bind:innerWidth />

<!--
width and height style (actual dimensions on client screen)
differ from canvas internal width and height(set in draw() function)
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
  on:keydown={keydown}
  on:contextmenu|preventDefault
/>

<button
  on:click={() => {
    grid.shown = !grid.shown;
  }}
>
  show grid: {grid.shown}
</button>

{#if dev}
  <p style:margin="0">mouse pos: x: {mouse.x}, y: {mouse.y}</p>
  <p style:margin="0">cursor pos: x: {cursor.x}, y: {cursor.y}</p>
  <p style:margin="0">last action: {lastAction}</p>
  <p style:margin="0">scale: {scale.val}</p>

  <svg
    {stroke}
    {width}
    {height}
    viewBox="0 0 {grid.x} {grid.y}"
    xmlns="http://www.w3.org/2000/svg"
  >
    <path d="M3,3L{grid.x - 3},{grid.y - 3}" />
  </svg>
{/if}
