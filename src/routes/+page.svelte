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
    preview_points = [],
    // exported svg properties
    svg,
    layers = [{ path: null, points: [] }],
    current_layer = 0,
    serializer,
    serialized_svg,
    fill = "none",
    stroke = "#000";

  onMount(() => {
    context = canvas.getContext("2d");
    serializer = new XMLSerializer();
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
    console.log(svg);
    console.log(layers[current_layer]);
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
    serialized_svg = serializer.serializeToString(svg);
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
    // drawPlacedPoints()
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
    let img = new Image();
    img.onload = function () {
      context.drawImage(
        img,
        0,
        0,
        grid.x * spacing.scaled,
        grid.y * spacing.scaled,
      );
    };
    img.src = "data:image/svg+xml;base64," + btoa(serialized_svg);
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

  // TODO function drawPlacedPoints() {}

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
   * also used when buttons are clicked
   * runs other methods based on which key pressed
   * https://100r.co/site/dotgrid.html#shortcut_quick_list
   */
  function keydown(event) {
    if (event.repeat) return;

    switch (event.key) {
      case keybinds.line:
        if (preview_points.length < 2) break;
        setPath("L");
        break;
      case keybinds.arc:
        if (preview_points.length < 2) break;
        setPath("A");
        break;
      case keybinds.arc_rev:
        if (preview_points.length < 2) break;
        setPath("Arev");
        break;
      case keybinds.bezier_quad:
        if (preview_points.length < 3 || preview_points.length % 2 === 0) break;
        setPath("Q");
        break;
      case keybinds.bezier_cube:
        if (preview_points.length < 4 || (preview_points.length - 1) % 3 !== 0)
          break;
        setPath("C");
        break;
    }

    draw();
  }

  function setPath(type) {
    let new_d = `M${preview_points[0].x} ${preview_points[0].y}`;
    preview_points[0].type = "M";

    switch (type) {
      case "L":
        for (let i = 1; i < preview_points.length; i++) {
          preview_points[i].type = "L";
          new_d += `L${preview_points[i].x} ${preview_points[i].y}`;
        }
        break;
      case "A":
      case "Arev":
        const rotation = type === "A" ? 1 : 0;
        for (let i = 1; i < preview_points.length; i++) {
          preview_points[i].type = "A";
          new_d += `A${Math.abs(
            preview_points[i].x - preview_points[i - 1].x,
          )} ${Math.abs(preview_points[i].y - preview_points[i - 1].y)} 0 0 ${rotation} ${
            preview_points[i].x
          } ${preview_points[i].y}`;
        }
        break;
      case "Q":
        let control_point = true;
        for (let i = 1; i < preview_points.length; i++) {
          preview_points[i].type = "Q";
          if (control_point) {
            new_d += `Q${preview_points[i].x} ${preview_points[i].y}`;
          } else {
            new_d += ` ${preview_points[i].x} ${preview_points[i].y}`;
          }

          control_point = !control_point;
        }
        break;
      case "C":
        for (let i = 1; i < preview_points.length; i++) {
          if ((i - 1) % 3 === 0) {
            new_d += `C${preview_points[i].x} ${preview_points[i].y}`;
          } else {
            new_d += ` ${preview_points[i].x} ${preview_points[i].y}`;
          }
        }
    }

    layers[current_layer].points.push(...preview_points);
    preview_points.length = 0;
    layers[current_layer].path.setAttribute(
      "d",
      layers[current_layer].path.getAttribute("d") + new_d,
    );
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

  {@html serialized_svg}
{/if}
