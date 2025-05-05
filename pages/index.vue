<template>
  <div>
    <div id="canvas-container">
    </div>
    <button @click="createCanvas">Spawn Canvas</button>
    <button @click="toggleMode">{{ mode === "draw" ? "Switch to View Mode" : "Switch to Draw Mode" }}</button>
  </div>
</template>

<script>
export default {
  data() {
    return {
      canvases: [],
      stabalizingFactor: 0.1,
      mode: "view",
    };
  },

  methods: {
    toggleMode() {
      this.mode = this.mode === "draw" ? "view" : "draw";

      this.canvases.forEach(canvasObj => {
        if (this.mode === "view") {
          canvasObj.canvas.style.border = "2px dotted gray";
        } else {
          canvasObj.canvas.style.border = "2px solid black";
        }
      });
    },

    handleResize() {
      this.width = window.innerWidth;
      this.height = window.innerHeight;
    },

    startDrag(e, canvasIndex) {
      if (this.mode === "draw") return;
      const canvasObj = this.canvases[canvasIndex];
      canvasObj.isDragging = true;
      canvasObj.offsetX = e.clientX - canvasObj.x;
      canvasObj.offsetY = e.clientY - canvasObj.y;

      document.body.style.userSelect = "none";
    },

    stopDrag(e, canvasIndex) {
      if (this.mode === "draw") return;
      const canvasObj = this.canvases[canvasIndex];
      canvasObj.isDragging = false;
      document.body.style.userSelect = "";
    },

    dragCanvas(e, canvasIndex) {
      if (this.mode === "draw") return;
      const canvasObj = this.canvases[canvasIndex];
      if (!canvasObj.isDragging) return;

      canvasObj.x = e.clientX - canvasObj.offsetX;
      canvasObj.y = e.clientY - canvasObj.offsetY;

      canvasObj.canvas.style.left = `${canvasObj.x}px`;
      canvasObj.canvas.style.top = `${canvasObj.y}px`;
    },

    startPosition(e, canvasIndex) {
      if (this.mode === "view") return;
      this.canvases[canvasIndex].isDrawing = true;
      const canvasObj = this.canvases[canvasIndex];
      const context = canvasObj.context;

      canvasObj.prevX = null;
      canvasObj.prevY = null;

      context.beginPath();
    },

    endPosition(e, canvasIndex) {
      if (this.mode === "view") return;
      this.canvases[canvasIndex].isDrawing = false;
      const context = this.canvases[canvasIndex].context;
      context.beginPath();
    },

    draw(e, canvasIndex) {
      if (this.mode === "view") return;
      const canvasObj = this.canvases[canvasIndex];
      if (!canvasObj.isDrawing) return;

      const context = canvasObj.context;
      const canvas = canvasObj.canvas;
      const rect = canvas.getBoundingClientRect();

      const x = e.clientX - rect.left;
      const y = e.clientY - rect.top;

      if (canvasObj.prevX === null || canvasObj.prevY === null) {
        canvasObj.prevX = x;
        canvasObj.prevY = y;

        return;
      }

      const stableX = canvasObj.prevX + (x - canvasObj.prevX) * this.stabalizingFactor;
      const stableY = canvasObj.prevY + (y - canvasObj.prevY) * this.stabalizingFactor;

      context.lineWidth = 10;
      context.lineCap = "round";
      context.lineJoin = "round";

      context.moveTo(canvasObj.prevX, canvasObj.prevY);
      context.quadraticCurveTo(
        canvasObj.prevX + (stableX - canvasObj.prevX) / 2,
        canvasObj.prevY + (stableY - canvasObj.prevY) / 2,
        stableX,
        stableY
      );
      context.stroke();

      canvasObj.prevX = stableX;
      canvasObj.prevY = stableY;
    },

    createCanvas() {
      const canvasContainer = document.getElementById("canvas-container");
      const canvasIndex = this.canvases.length;

      const canvas = document.createElement("canvas");
      canvas.id = `canvas-${canvasIndex}`;
      canvas.width = 500;
      canvas.height = 500;

      const context = canvas.getContext("2d");
      canvas.style.position = "absolute";
      canvas.style.left = `${50 * canvasIndex}px`;
      canvas.style.top = `${50 * canvasIndex}px`;

      const canvasObj = {
        canvas,
        context,
        isDrawing: false,
        prevX: null,
        prevY: null,
        isDragging: false,
        x: 50 * canvasIndex,
        y: 50 * canvasIndex,
      };

      if (this.mode === "view") {
        canvas.style.border = "2px dotted gray";
      } else {
        canvas.style.border = "2px solid black";
      }

      canvasContainer.appendChild(canvas);
      this.canvases.push(canvasObj);

      canvas.addEventListener("mousedown", (e) => this.startPosition(e, canvasIndex));
      canvas.addEventListener("mouseup", (e) => this.endPosition(e, canvasIndex));
      canvas.addEventListener("mousemove", (e) => this.draw(e, canvasIndex));

      canvas.addEventListener("mousedown", (e) => this.startDrag(e, canvasIndex));
      window.addEventListener("mousemove", (e) => this.dragCanvas(e, canvasIndex));
      window.addEventListener("mouseup", (e) => this.stopDrag(e, canvasIndex));
    },
  },
};
</script>

<style scoped>
* {
  margin: 0;
  padding: 0;
  box-sizing: border-box;
}

#canvas-container {
  position: relative;
  width: 100%;
  height: 500px;
  background-color: #f0f0f0;
}

canvas {
  cursor: grab;
}
</style>