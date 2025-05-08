<template>
    <div>
      <div id="canvas-container" ref="canvasContainer"></div>
  
      <div class="controls">
        <div class="tool-panel">
          <button @click="toggleDrawingMode" :class="{ active: isDrawingMode }">
            {{ isDrawingMode ? "Switch to Select Mode" : "Switch to Draw Mode" }}
          </button>
        </div>
  
        <div class="color-panel">
          <label for="brush-color">Color:</label>
          <input type="color" id="brush-color" v-model="brushColor" @change="updateBrushSettings">
  
          <label for="brush-width">Width:</label>
          <input type="range" id="brush-width" min="1" max="50" v-model.number="brushWidth" @input="updateBrushSettings">
          <span>{{ brushWidth }}px</span>
        </div>
      </div>
    </div>
  </template>
  
  <script>
    import * as fabric from "fabric";
  
    export default {
      data() {
        return {
          canvas: null,
          isDrawingMode: true,
          brushWidth: 5,
          brushColor: "#000000",
          canvasWidth: 800,
          canvasHeight: 500
        };
      },
  
      mounted() {
        this.initCanvas();
      },
  
      beforeDestroy() {
        if (this.canvas) {
          this.canvas.dispose();
        }
      },
  
      methods: {
        initCanvas() {
          const canvasEl = document.createElement("canvas");
          canvasEl.id = "main-canvas";
          canvasEl.width = this.canvasWidth;
          canvasEl.height = this.canvasHeight;
          this.$refs.canvasContainer.appendChild(canvasEl);
  
          this.canvas = new fabric.Canvas("main-canvas", {
            width: this.canvasWidth,
            height: this.canvasHeight,
            preserveObjectStacking: true,
            backgroundColor: "#f0f0f0"
          });
  
          this.canvas.freeDrawingBrush = new fabric.PencilBrush(this.canvas);
          this.updateBrushSettings();
  
          this.canvas.isDrawingMode = this.isDrawingMode;
        },
  
        toggleDrawingMode() {
          this.isDrawingMode = !this.isDrawingMode;
          this.canvas.isDrawingMode = this.isDrawingMode;
  
          if (!this.isDrawingMode) {
            this.canvas.selection = true;
            this.canvas.forEachObject(obj => {
              obj.selectable = true;
              obj.hasControls = true;
              obj.hasBorders = true;
            });
          } else {
            this.canvas.discardActiveObject();
            this.canvas.renderAll();
          }
        },
  
        updateBrushSettings() {
          if (this.canvas && this.canvas.freeDrawingBrush) {
            this.canvas.freeDrawingBrush.width = this.brushWidth;
            this.canvas.freeDrawingBrush.color = this.brushColor;
            this.canvas.freeDrawingBrush.decimate = 8;
          }
        }
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
      margin-bottom: 20px;
      overflow: hidden;
      border: 1px solid #ddd;
    }
  </style>
  