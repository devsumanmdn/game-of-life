<script lang="ts">
  import { onMount, tick } from "svelte";

  let canvas: HTMLCanvasElement;

  let drawing: boolean = false;
  let mouseDown: boolean = false;

  let boxes: Array<string> = [];

  let gameFPS: number = 10;
  let scale: number = 10;
  const boxBorderWidth = 1;

  let movedPixels: { x: number; y: number } = { x: 0, y: 0 };
  let moveMode: boolean = false;

  let gameTimeoutId: NodeJS.Timeout | null = null;

  const drawCanvasBorder = (): void => {
    const context = canvas.getContext("2d");

    context.fillStyle = "#000000";
    context.fillRect(0, 0, canvas.width, canvas.height);

    drawBoxes(scale, boxes);
  };

  onMount(() => {
    const handleResize = () => {
      canvas.height = window.innerHeight;
      canvas.width = window.innerWidth;
      drawCanvasBorder();

      drawing = true;
    };

    window.addEventListener("resize", handleResize);

    handleResize();

    return () => window.removeEventListener("resize", handleResize);
  });

  const getKey = (x: number, y: number) => x + "/" + y;

  const drawBox = (
    idx1: number,
    idx2: number,
    alive: number,
    scale: number
  ) => {
    const context = canvas.getContext("2d");

    context.lineWidth = boxBorderWidth;
    context.strokeStyle = "#666666";

    const x = idx1 * scale + boxBorderWidth / 2;
    const y = idx2 * scale + boxBorderWidth / 2;

    if (alive) {
      context.fillStyle = "#FFFFFFAA";
    } else {
      context.fillStyle = "#000000";
    }

    context.fillRect(x, y, scale, scale);
    context.strokeRect(x, y, scale, scale);
  };

  let lastPaintedBoxes: Array<string> = [];
  let lastScale: number;
  // let lastMovedPixels: { x: number; y: number };
  // let paintedEmpty: boolean = false;

  const drawBoxes = async (
    scale: number,
    passedBoxes?: Array<string>,
    movedPixels?: null | { x: number; y: number },
    optimisePainting: boolean = false
  ) => {
    if (!canvas) {
      await tick();
    }

    const context = canvas.getContext("2d");

    const scaleUpdated = scale !== lastScale;
    if (!optimisePainting) {
      await tick();
      context.clearRect(0, 0, canvas.width, canvas.height);
      context.fillStyle = "#000000";
      context.fillRect(0, 0, canvas.width, canvas.height);
    }


    const totalPossibleBoxes = {
      h: Math.floor(canvas.width / scale),
      v: Math.floor((canvas.height - 60) / scale),
    };

    const paintedBoxes = [];

    for (let idx1 = 0; idx1 < totalPossibleBoxes.h; idx1++) {
      for (let idx2 = 0; idx2 < totalPossibleBoxes.v; idx2++) {
        const updatedValue = passedBoxes?.includes(
          getKey(idx1 - (movedPixels?.x || 0), idx2 - (movedPixels?.y || 0))
        )
          ? 1
          : 0;

        if (updatedValue) {
          paintedBoxes.push(getKey(idx1, idx2));
        }

        if (
          !optimisePainting ||
          !!updatedValue !== lastPaintedBoxes.includes(getKey(idx1, idx2))
        ) {
          drawBox(idx1, idx2, updatedValue, scale);
        }
      }
    }

    lastPaintedBoxes = paintedBoxes;
    // lastMovedPixels = movedPixels;
    lastScale = scale;
  };

  const onMouseDown = () => {
    mouseDown = true;
  };

  const onMouseUp = () => {
    mouseDown = false;
  };

  const handleDrawChange = (drawing) => {
    if (!canvas) {
      return;
    }

    if (drawing) {
      canvas.addEventListener("mousedown", onMouseDown);
      canvas.addEventListener("mouseup", onMouseUp);
    } else {
      canvas.removeEventListener("mousedown", onMouseDown);
      canvas.removeEventListener("mouseup", onMouseUp);
    }
  };

  let lastMouseX: number | null = null;
  let lastMouseY: number | null = null;

  const onMouseMove = (e: MouseEvent) => {
    const x = Math.floor(e.offsetX / scale);
    const y = Math.floor(e.offsetY / scale);

    if (moveMode) {
      if ((lastMouseX !== x || lastMouseY !== y) && lastMouseX && lastMouseY) {
        movedPixels = {
          x: movedPixels.x - (lastMouseX - x),
          y: movedPixels.y - (lastMouseY - y),
        };
      }

      lastMouseX = x;
      lastMouseY = y;
    } else if (lastMouseX !== x || lastMouseY !== y) {
      const key = getKey(x - movedPixels.x, y - movedPixels.y);

      boxes = [
        ...new Set(
          boxes.includes(key) ? boxes.filter((k) => k !== key) : [...boxes, key]
        ),
      ];

      lastMouseX = x;
      lastMouseY = y;
    }
  };

  const handleMouseDownChange = (mouseDown) => {
    if (!canvas) {
      return;
    }

    if (mouseDown) {
      window.addEventListener("mousemove", onMouseMove);
    } else {
      lastMouseX = null;
      lastMouseY = null;
      window.removeEventListener("mousemove", onMouseMove);
    }
  };

  const callDrayBoxes = (s, b, m, o) =>
    drawBoxes(s || scale, b || boxes, m || movedPixels, o);

  $: handleDrawChange(drawing || moveMode);
  $: handleMouseDownChange(mouseDown);

  $: callDrayBoxes(scale, undefined, undefined, false);

  $: callDrayBoxes(undefined, boxes, movedPixels, true);

  let boxesToCalculate: Array<string> = [];

  const startGame = () => {
    const getAliveNeighbours = (idx1, idx2) => {
      const neighborsKeys = [];
      [-1, 0, +1].forEach((direction) => {
        neighborsKeys.push(getKey(idx1 + direction, idx2 - 1));

        if (direction !== 0) {
          neighborsKeys.push(getKey(idx1 + direction, idx2 + 0));
        }
        neighborsKeys.push(getKey(idx1 + direction, idx2 + 1));
      });

      const aliveNeighbours = neighborsKeys.reduce(
        (acc, neighbor) => acc + (boxes?.includes(neighbor) ? 1 : 0),
        0
      );
      return { aliveNeighbours, neighborsKeys };
    };

    let initialBoxesToCalculate = [];
    boxes.forEach((box) => {
      const [idx1, idx2] = box.split("/").map((num) => parseInt(num));

      const { neighborsKeys } = getAliveNeighbours(idx1, idx2);

      initialBoxesToCalculate.push(...neighborsKeys);
    });

    initialBoxesToCalculate = [...new Set([...initialBoxesToCalculate])];

    const paintNext = () => {
      let newBoxes = [];

      const nextBoxesToCalculate = [];
      (boxesToCalculate.length
        ? boxesToCalculate
        : initialBoxesToCalculate
      ).map((box) => {
        const [idx1, idx2] = box.split("/").map((num) => parseInt(num));
        const { aliveNeighbours, neighborsKeys } = getAliveNeighbours(
          idx1,
          idx2
        );

        if (
          (!boxes.includes(getKey(idx1, idx2)) && aliveNeighbours === 3) ||
          (boxes.includes(getKey(idx1, idx2)) &&
            aliveNeighbours > 1 &&
            aliveNeighbours < 4)
        ) {
          newBoxes = [...newBoxes, getKey(idx1, idx2)];
          nextBoxesToCalculate.push(...neighborsKeys);
        }
      });

      boxesToCalculate = [...new Set([...nextBoxesToCalculate])];

      boxes = newBoxes;

      gameTimeoutId = setTimeout(paintNext, 1000 / gameFPS);
    };

    paintNext();
  };

  const handlePause = () => {
    window.clearTimeout(gameTimeoutId);
    boxesToCalculate = [];
    gameTimeoutId = null;
  };
</script>

<main>
  <canvas bind:this={canvas} />
  <div class="actions">
    <button
      on:click={() => {
        boxes = [];
        drawBoxes(scale);
      }}
    >
      Clear
    </button>
    {#if gameTimeoutId}
      <button on:click={handlePause}> Pause Game </button>
    {:else}
      <button on:click={startGame}> Start Game </button>
    {/if}
    <button class:selected={moveMode} on:click={() => (moveMode = !moveMode)}
      >Move</button
    >

    <label>
      <span>
        Game FPS({gameFPS}):
      </span>
      <input
        type="range"
        min={1}
        max={60}
        bind:value={gameFPS}
        class="rangeSlider"
      />
    </label>
    <label>
      <span>
        Scale ({scale}):
      </span>
      <input
        type="range"
        min={5}
        max={200}
        bind:value={scale}
        class="rangeSlider"
      />
    </label>
  </div>
</main>

<style lang="scss">
  :global(*) {
    box-sizing: border-box;
  }

  :global(body) {
    margin: 0;
    padding: 0;
    overflow: hidden;
  }

  .actions {
    position: absolute;
    bottom: 0;
    display: flex;
    height: 40px;
    align-items: center;
    margin: 10px;

    & > *:not(:last-child) {
      margin-right: 20px;
    }

    button {
      margin-bottom: 0;
      border-radius: 4px;
      border: none;
      background-color: #fff3;
      color: #fff;
      padding: 0 20px;
      line-height: 40px;
      transition-duration: 0.3s;
      width: 160px;

      &.selected {
        border: #aaaaaa 2px solid;
        color: #000000;
        background-color: #ffffff;
      }

      &:hover,
      &:focus:not(.selected) {
        background-color: #fff5;

        &:focus {
          outline: 1px solid #fffa;
        }
      }
    }

    label {
      display: inline-flex;
      flex-direction: column;
      color: #ffffffbb;
      span {
        line-height: 20px;
      }

      .rangeSlider {
        width: 200px;
        margin-bottom: 0;
      }
    }
  }
</style>
