<template>
    <div>
        <div id="canvas-container"></div>
        <button @click="createCanvas">Spawn Canvas</button>
        
        <button @click="toggleMode">{{ mode === "draw" ? "Switch to View Mode" : "Switch to Draw Mode" }}</button>

        <div v-if="canvases.length > 0">
            <h3>Layer Controls</h3>
            <div v-for="(canvasObj, index) in canvases" :key="canvasObj.canvas.id" class="layer-control">
                <div>
                    <label>Layer {{ index + 1 }}</label>
                    <input type="checkbox" v-model="canvasObj.visible" @change="toggleLayerVisibility(index)" /> Visible
                    <button @click="moveLayerUp(index)">↑</button>
                    <button @click="moveLayerDown(index)">↓</button>
                    <input type="range" min="0" max="1" step="0.1" v-model="canvasObj.opacity" @input="adjustOpacity(index)" />
                    <span>{{ (canvasObj.opacity * 100).toFixed(0) }}%</span>
                    
                    <button @click="toggleLayerMode(index)">
                        {{ canvasObj.mode === "draw" ? "Switch to View Mode" : "Switch to Draw Mode" }}
                    </button>
                </div>
            </div>
        </div>
    </div>
</template>

<script>
import { fabric } from "fabric";

export default {
    data() {
        return {
            canvases: [],
            mode: "draw",
        };
    },

    methods: {
        toggleMode() {
            this.mode = this.mode === "draw" ? "view" : "draw";
            this.canvases.forEach(canvasObj => {
                this.toggleCanvasMode(canvasObj);
            });
        },

        toggleLayerMode(index) {
            const canvasObj = this.canvases[index];
            this.toggleCanvasMode(canvasObj);
        },

        toggleCanvasMode(canvasObj) {
            if (canvasObj.mode === "draw") {
                canvasObj.mode = "view";
                canvasObj.canvas.selection = false;
            } else {
                canvasObj.mode = "draw";
                canvasObj.canvas.selection = true;
            }
        },

        handleResize() {
            this.width = window.innerWidth;
            this.height = window.innerHeight;
        },

        createCanvas() {
            const canvasContainer = document.getElementById("canvas-container");
            const canvasIndex = this.canvases.length;

            const canvas = new fabric.Canvas(`canvas-${canvasIndex}`, {
                width: 500,
                height: 500,
                backgroundColor: "#f0f0f0",
            });

            canvas.set({ left: 50 * canvasIndex, top: 50 * canvasIndex });
            canvas.selection = false;

            const canvasObj = {
                canvas,
                mode: "view",
                opacity: 1,
                visible: true,
            };

            canvasContainer.appendChild(canvas.lowerCanvasEl);

            this.canvases.push(canvasObj);

            canvas.on("mouse:down", (e) => this.startPosition(e, canvasIndex));
            canvas.on("mouse:up", (e) => this.endPosition(e, canvasIndex));
            canvas.on("mouse:move", (e) => this.draw(e, canvasIndex));
        },

        toggleLayerVisibility(index) {
            const canvasObj = this.canvases[index];
            canvasObj.canvas.setVisible(canvasObj.visible);
            canvasObj.canvas.renderAll();
        },

        adjustOpacity(index) {
            const canvasObj = this.canvases[index];
            canvasObj.canvas.setOpacity(canvasObj.opacity);
            canvasObj.canvas.renderAll();
        },

        moveLayerUp(index) {
            if (index > 0) {
                const temp = this.canvases[index];
                this.canvases.splice(index, 1);
                this.canvases.splice(index - 1, 0, temp);
                this.reorderLayers();
            }
        },

        moveLayerDown(index) {
            if (index < this.canvases.length - 1) {
                const temp = this.canvases[index];
                this.canvases.splice(index, 1);
                this.canvases.splice(index + 1, 0, temp);
                this.reorderLayers();
            }
        },

        reorderLayers() {
            const canvasContainer = document.getElementById("canvas-container");
            this.canvases.forEach((canvasObj, index) => {
                canvasObj.canvas.set({ left: 50 * index, top: 50 * index });
                canvasContainer.appendChild(canvasObj.canvas.lowerCanvasEl);
            });
        },

        startPosition(e, canvasIndex) {
            if (this.canvases[canvasIndex].mode === "view") return;

            const canvasObj = this.canvases[canvasIndex];
            canvasObj.isDrawing = true;
            canvasObj.prevX = e.e.offsetX;
            canvasObj.prevY = e.e.offsetY;
        },

        endPosition(e, canvasIndex) {
            if (this.canvases[canvasIndex].mode === "view") return;

            const canvasObj = this.canvases[canvasIndex];
            canvasObj.isDrawing = false;
        },

        draw(e, canvasIndex) {
            if (this.canvases[canvasIndex].mode === "view") return;

            const canvasObj = this.canvases[canvasIndex];
            if (!canvasObj.isDrawing) return;

            const canvas = canvasObj.canvas;
            const x = e.e.offsetX;
            const y = e.e.offsetY;

            if (canvasObj.prevX === null || canvasObj.prevY === null) {
                canvasObj.prevX = x;
                canvasObj.prevY = y;
                return;
            }

            const stableX = canvasObj.prevX + (x - canvasObj.prevX) * 0.1;
            const stableY = canvasObj.prevY + (y - canvasObj.prevY) * 0.1;

            const line = new fabric.Line([canvasObj.prevX, canvasObj.prevY, stableX, stableY], {
                stroke: "black",
                strokeWidth: 5,
                selectable: false
            });

            canvas.add(line);
            canvasObj.prevX = stableX;
            canvasObj.prevY = stableY;
        },
    }
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

    .layer-control {
        margin-bottom: 10px;
    }

    .layer-control input[type="range"] {
        width: 100px;
    }
</style>
