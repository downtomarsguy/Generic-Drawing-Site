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
            <div class="image-panel">
                <label for="image-upload">Upload Image:</label>
                <input type="file" id="image-upload" @change="uploadImage" accept="image/*">
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
                canvasHeight: 500,
                selectedObject: null,
                clipboardObject: null
            };
        },
    
        mounted() {
            this.initCanvas();
            window.addEventListener("resize", this.updateCanvasSize);
            window.addEventListener("keydown", this.handleKeyDown);
        },
    
        beforeDestroy() {
            if (this.canvas) {
                this.canvas.dispose();
            }
            window.removeEventListener("resize", this.updateCanvasSize);
            window.removeEventListener("keydown", this.handleKeyDown);
        },
    
        methods: {
            initCanvas() {
                const canvasElem = document.createElement("canvas");
                canvasElem.id = "main-canvas";
                canvasElem.height = this.canvasHeight;
                this.$refs.canvasContainer.appendChild(canvasElem);
                
                this.canvas = new fabric.Canvas("main-canvas", {
                    height: this.canvasHeight,
                    preserveObjectStacking: true,
                    backgroundColor: "#f0f0f0"
                });
                
                this.updateCanvasSize();
                
                this.canvas.freeDrawingBrush = new fabric.PencilBrush(this.canvas);
                this.updateBrushSettings();
                this.canvas.isDrawingMode = this.isDrawingMode;
                
                this.canvas.on("selection:created", (e) => {
                    this.selectedObject = e.selected[0];
                });
                
                this.canvas.on("selection:updated", (e) => {
                    this.selectedObject = e.selected[0];
                });
                
                this.canvas.on("selection:cleared", () => {
                    this.selectedObject = null;
                });
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
            },
            
            updateCanvasSize() {
                const containerWidth = this.$refs.canvasContainer.offsetWidth;
                this.canvas.setWidth(containerWidth);
                this.canvas.renderAll();
            },
            
            deleteSelectedObject() {
                if (this.selectedObject) {
                    this.canvas.remove(this.selectedObject);
                    this.selectedObject = null;
                    this.canvas.renderAll();
                }
            },
            
            copySelectedObject() {
                if (!this.selectedObject) return;
                
                this.selectedObject.clone().then((cloned) => {
                    this.clipboardObject = cloned;
                });
            },
            
            async pasteObject() {
                if (!this.clipboardObject) return;
                
                try {
                    const clonedObj = await this.clipboardObject.clone();
                    
                    this.canvas.discardActiveObject();
                    
                    clonedObj.set({
                        left: clonedObj.left + 10,
                        top: clonedObj.top + 10,
                        evented: true,
                    });
                    
                    if (clonedObj instanceof fabric.ActiveSelection) {
                        clonedObj.canvas = this.canvas;
                        clonedObj.forEachObject((obj) => {
                            this.canvas.add(obj);
                        });
                        clonedObj.setCoords();
                    } else {
                        this.canvas.add(clonedObj);
                    }
                    
                    this.clipboardObject.top += 10;
                    this.clipboardObject.left += 10;
                    
                    this.canvas.setActiveObject(clonedObj);
                    this.canvas.requestRenderAll();
                } catch (error) {
                    console.error("Error pasting object:", error);
                }
            },

            uploadImage(event) {
                const file = event.target.files[0];
                if (file) {
                    const reader = new FileReader();
                    reader.onload = (e) => {
                        const imgElement = new Image();
                        imgElement.src = e.target.result;
                        
                        imgElement.onload = () => {
                            const fabricImage = new fabric.Image(imgElement);
                            this.canvas.add(fabricImage);
                            this.canvas.setActiveObject(fabricImage);
                            this.canvas.requestRenderAll();
                        };
                    };
                    reader.readAsDataURL(file);
                }
            },

            handleKeyDown(e) {
                if (this.isDrawingMode) return;
                
                if ((e.ctrlKey || e.metaKey) && e.key === "c") {
                    if (this.selectedObject) {
                        this.copySelectedObject();
                        e.preventDefault();
                    }
                }
                
                if ((e.ctrlKey || e.metaKey) && e.key === "v") {
                    if (this.clipboardObject) {
                        this.pasteObject();
                        e.preventDefault();
                    }
                }

                if ((e.ctrlKey || e.metaKey) && e.key === "p") {
                    e.preventDefault();
                    this.pasteImageFromClipboard();
                }
                
                if (e.key === "Delete") {
                    if (this.selectedObject) {
                        this.deleteSelectedObject();
                        e.preventDefault();
                    }
                }
            },

            pasteImageFromClipboard() {
                navigator.clipboard.read().then(async (data) => {
                    for (const item of data) {
                        if (item.types.includes("image/png")) {
                            const blob = await item.getType("image/png");
                            const url = URL.createObjectURL(blob);
    
                            const imgElement = new Image();
                            imgElement.src = url;
    
                            imgElement.onload = () => {
                                const fabricImage = new fabric.Image(imgElement);
                                fabricImage.set({
                                    left: this.canvas.width / 2,
                                    top: this.canvas.height / 2,
                                });
                                this.canvas.add(fabricImage);
                                this.canvas.setActiveObject(fabricImage);
                                this.canvas.requestRenderAll();
                            };
                        }
                    }
                }).catch((err) => {
                    console.error("Failed to read clipboard content: ", err);
                });
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

    .image-panel {
        margin-top: 20px;
    }
</style>
