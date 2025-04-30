<template>
    <div class="">
        <div id="canvas-container">
        </div>
        <button @click="createCanvas">Spawn Canvas</button>
    </div>
</template>

<script>
    export default {
        data() {
            return {
                width: window.innerWidth - 500,
                height: window.innerHeight,
                activeDraw: false,
                canvases: [],
                stabalizingFactor: 0.1,
            };
        },

        methods: {
            handleResize() {
                this.width = window.innerWidth;
                this.height = window.innerHeight;
            },

            startPosition(e, canvasIndex) {
                this.canvases[canvasIndex].isDrawing = true;
                this.canvases[canvasIndex].isDrawing = true;
                const canvasObj = this.canvases[canvasIndex];
                const context = canvasObj.context;

                canvasObj.prevX = null;
                canvasObj.prevY = null;

                context.beginPath();
            },

            endPosition(e, canvasIndex) {
                this.canvases[canvasIndex].isDrawing = false;
                const context = this.canvases[canvasIndex].context;
                context.beginPath();
            },

            draw(e, canvasIndex) {
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
                canvasContainer.appendChild(canvas);

                this.canvases.push({
                    canvas,
                    context,
                    isDrawing: false,
                    prevX: null,
                    prevY: null,
                });

                canvas.addEventListener("mousedown", (e) => this.startPosition(e, canvasIndex));
                canvas.addEventListener("mouseup", (e) => this.endPosition(e, canvasIndex));
                canvas.addEventListener("mousemove", (e) => this.draw(e, canvasIndex));
                window.addEventListener("resize", this.handleResize);
            },
        },
    };
</script>

<style>
    * {
        margin: 0;
        padding: 0;
        box-sizing: border-box;
    }

    canvas {
        border: 2px solid black;
    }

    div {
        display: flex;
        align-items: flex-start;
    }
</style>