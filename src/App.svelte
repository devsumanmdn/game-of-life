<script lang="ts">
  import { onMount, tick } from "svelte";

  let canvas: HTMLCanvasElement;

  let drawing: boolean = false;
  let mouseDown: boolean = false;

  let boxes: Array<Array<0 | 1>> = [];

  let gameFPS: number = 10;
  let scale: number = 10;
  const boxBorderWidth = 1;

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
      context.fillRect(x, y, scale, scale);
    }

    context.strokeRect(x, y, scale, scale);
  };

  let lastTotalPossibleBoxes: null | { h: number; v: number } = null;
  const drawBoxes = async (
    scale: number,
    passedBoxes?: Array<Array<0 | 1>>
  ) => {
    if (!canvas) {
      await tick();
    }

    const context = canvas.getContext("2d");

    context.clearRect(0, 0, canvas.width, canvas.height);

    context.fillStyle = "#000000";
    context.fillRect(0, 0, canvas.width, canvas.height);

    const totalPossibleBoxes = {
      h: Math.floor(canvas.width / scale),
      v: Math.floor((canvas.height - 60) / scale),
    };

    let updated = false;

    const newBoxes = Array(totalPossibleBoxes.h)
      .fill([])
      .map((_, idx1) => {
        return Array(totalPossibleBoxes.v)
          .fill(0)
          .map((alive, idx2) => {
            drawBox(idx1, idx2, passedBoxes?.[idx1]?.[idx2] || 0, scale);
            return passedBoxes?.[idx1]?.[idx2] || 0;
          });
      });

    boxes = newBoxes;
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

    if (lastMouseX !== x || lastMouseY !== y) {
      boxes = boxes.map((hBoxes, idx1) =>
        hBoxes.map((box, idx2) => (idx1 === x && idx2 === y ? 1 : box))
      );

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
      window.removeEventListener("mousemove", onMouseMove);
    }
  };

  $: handleDrawChange(drawing);
  $: handleMouseDownChange(mouseDown);

  $: drawBoxes(scale, boxes);

  const startGame = () => {
    const getAliveNeighbours = (idx1, idx2) => {
      let result = 0;
      [-1, 0, +1].forEach((direction) => {
        result += boxes?.[idx1 + direction]?.[idx2 - 1];
        if (direction !== 0) {
          result += boxes?.[idx1 + direction]?.[idx2 + 0] || 0;
        }
        result += boxes?.[idx1 + direction]?.[idx2 + 1];
      });

      return result;
    };

    const paintNext = () => {
      const newBoxes = boxes.map((hboxes, idx1) =>
        hboxes.map((box, idx2): 0 | 1 => {
          const aliveNeighbours = getAliveNeighbours(idx1, idx2);

          if (box === 1 && aliveNeighbours > 1 && aliveNeighbours < 4) {
            return 1;
          }

          if (box === 0 && aliveNeighbours === 3) {
            return 1;
          }

          return 0;
        })
      );

      boxes = newBoxes;

      gameTimeoutId = setTimeout(paintNext, 1000 / gameFPS);
    };

    paintNext();
  };

  const handlePause = () => {
    window.clearTimeout(gameTimeoutId);
    gameTimeoutId = null;
  };
</script>

<main>
  <canvas bind:this={canvas} />
  <div class="actions">
    <button on:click={() => drawBoxes(scale)}> Clear </button>
    {#if gameTimeoutId}
      <button on:click={handlePause}> Pause Game </button>
    {:else}
      <button on:click={startGame}> Start Game </button>
    {/if}

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

      &:hover,
      &:focus {
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
