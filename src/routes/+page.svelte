<script>
  import { dev } from "$app/environment";

  let // bound to the window size
    innerWidth,
    innerHeight,
    // canvas html tag properties
    /** @type {HTMLCanvasElement} */
    canvas,
    /** @type {CanvasRenderingContext2D} */
    context,
    /** @type {DOMRect} */
    rect,
    width = "80vw",
    height = "80vh",
    background = "#eee",
    // settings & state
    grid = { x: 0, y: 0, shown: true, spacing: 15 },
    scale = { val: 1, min: 1, max: 5 },
    mouse = { x: 0, y: 0 },
    lastAction,
    points = [],
    segments = [],
    // exported svg properties
    fill = "none",
    stroke = "#000",
    viewBox = "";

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
   * sets the canvas dimensions and draws the grid
   * unless `grid.shown` is false
   */
  function draw() {
    // make the canvas dimensions divisible by 3
    let w = Math.trunc(rect.width);
    let h = Math.trunc(rect.height);
    canvas.width = w - (w % 3);
    canvas.height = h - (h % 3);
    grid.x = Math.trunc(canvas.width / grid.spacing);
    grid.y = Math.trunc(canvas.height / grid.spacing);

    // draw grid markers
    if (grid.shown) {
      context.beginPath();
      for (let x = 1; x < grid.x / scale.val; x++) {
        for (let y = 1; y < grid.y / scale.val; y++) {
          const thick_dot = x % 4 === 0 && y % 4 === 0;
          const pos = { x: x * grid.spacing, y: y * grid.spacing };
          const radius = thick_dot ? 2 : 1;
          context.moveTo(pos.x * scale.val, pos.y * scale.val);
          context.arc(
            pos.x * scale.val,
            pos.y * scale.val,
            radius * scale.val,
            0,
            2 * Math.PI,
            false,
          );
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
    let offset = { x: mouse.x + 15, y: mouse.y + 15 };
    context.arc(
      offset.x - (offset.x % (grid.spacing * scale.val)),
      offset.y - (offset.y % (grid.spacing * scale.val)),
      7.5 * scale.val,
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
      scale.val *= 10;
      scale.val += 1;
      scale.val = Math.trunc(scale.val);
      scale.val /= 10;
      // console.log(scale.val)
    } else if (event.deltaY > 0 && scale.val > scale.min) {
      scale.val *= 10;
      scale.val -= 1;
      scale.val = Math.trunc(scale.val);
      scale.val /= 10;
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
  <p style:margin="0">
    marker pos: x: {(mouse.x - (mouse.x % grid.spacing)) / grid.spacing}, y: {(mouse.y -
      (mouse.y % grid.spacing)) /
      grid.spacing}
  </p>
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
