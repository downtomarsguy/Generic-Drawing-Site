<template>
    <div>
        <div id="canvas-container" ref="canvasContainer"></div>
        <div class="controls">
            <div class="tool-panel">
                <button @click="toggleDrawingMode" :class="{ active: isDrawingMode }">
                    {{ isDrawingMode ? "Switch to Select Mode" : "Switch to Draw Mode" }}
                </button>
                <button @click="undo" :disabled="!canUndo">Undo</button>
                <button @click="redo" :disabled="!canRedo">Redo</button>
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
                clipboardObject: null,
                historyStack: [],
                redoStack: [],
                canUndo: false,
                canRedo: false,
                isPerformingUndoRedo: false
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

                this.canvas.on("path:created", (e) => {
                    if (!this.isPerformingUndoRedo) {
                        this.addToHistory({
                            type: "path:created",
                            object: e.path
                        });
                    }
                });

                this.canvas.on("object:modified", (e) => {
                    if (!this.isPerformingUndoRedo) {
                        this.addToHistory({
                            type: "object:modified",
                            object: e.target,
                            oldState: this._getObjectState(e.target, true)
                        });
                    }
                });

                this.canvas.on("object:moving", (e) => {
                    if (!this.isPerformingUndoRedo && !e.target.lastMovingState) {
                        e.target.lastMovingState = this._getObjectState(e.target);
                    }
                });

                this.canvas.on("object:scaling", (e) => {
                    if (!this.isPerformingUndoRedo && !e.target.lastScalingState) {
                        e.target.lastScalingState = this._getObjectState(e.target);
                    }
                });

                this.canvas.on("object:rotating", (e) => {
                    if (!this.isPerformingUndoRedo && !e.target.lastRotatingState) {
                        e.target.lastRotatingState = this._getObjectState(e.target);
                    }
                });

                this.canvas.on("mouse:up", (e) => {
                    if (this.isPerformingUndoRedo) return;
                    
                    if (this.canvas._activeObject) {
                        const obj = this.canvas._activeObject;
                        
                        if (obj.lastMovingState) {
                            this.addToHistory({
                                type: "object:moved",
                                object: obj,
                                oldState: obj.lastMovingState
                            });
                            delete obj.lastMovingState;
                        }
                        
                        if (obj.lastScalingState) {
                            this.addToHistory({
                                type: "object:scaled",
                                object: obj,
                                oldState: obj.lastScalingState
                            });
                            delete obj.lastScalingState;
                        }
                        
                        if (obj.lastRotatingState) {
                            this.addToHistory({
                                type: "object:rotated",
                                object: obj,
                                oldState: obj.lastRotatingState
                            });
                            delete obj.lastRotatingState;
                        }
                    }
                });
            },
            
            _getObjectState(obj, fullState = false) {
                const state = {
                    left: obj.left,
                    top: obj.top,
                    scaleX: obj.scaleX,
                    scaleY: obj.scaleY,
                    angle: obj.angle
                };
                
                if (fullState) {
                    Object.assign(state, {
                        width: obj.width,
                        height: obj.height,
                        fill: obj.fill,
                        stroke: obj.stroke,
                        strokeWidth: obj.strokeWidth,
                        opacity: obj.opacity,
                    });
                }
                
                return state;
            },
            
            addToHistory(action) {
                if (this.redoStack.length > 0) {
                    this.redoStack = [];
                }
                
                this.historyStack.push(action);
                this.canUndo = true;
                this.canRedo = false;
                
                console.log("History:", this.historyStack);
            },
            
            undo() {
                if (!this.canUndo || this.historyStack.length === 0) return;
                
                this.isPerformingUndoRedo = true;
                
                const action = this.historyStack.pop();
                this.redoStack.push(action);
                
                this.canRedo = true;
                this.canUndo = this.historyStack.length > 0;
                
                this._processUndoAction(action);
                
                this.isPerformingUndoRedo = false;
            },
            
            redo() {
                if (!this.canRedo || this.redoStack.length === 0) return;
                
                this.isPerformingUndoRedo = true;
                
                const action = this.redoStack.pop();
                this.historyStack.push(action);
                
                this.canUndo = true;
                this.canRedo = this.redoStack.length > 0;
                
                this._processRedoAction(action);
                
                this.isPerformingUndoRedo = false;
            },
            
            _processUndoAction(action) {
                switch (action.type) {
                    case "path:created":
                        this.canvas.remove(action.object);
                        break;
                        
                    case "object:added":
                        this.canvas.remove(action.object);
                        break;
                        
                    case "object:removed":
                        this.canvas.add(action.object);
                        break;
                        
                    case "object:modified":
                    case "object:moved":
                    case "object:scaled":
                    case "object:rotated":
                        action.object.set(action.oldState);
                        action.object.setCoords();
                        break;
                }
                
                this.canvas.renderAll();
            },
            
            _processRedoAction(action) {
                switch (action.type) {
                    case "path:created":
                    case "object:added":
                        this.canvas.add(action.object);
                        break;
                        
                    case "object:removed":
                        this.canvas.remove(action.object);
                        break;
                        
                    case "object:modified":
                    case "object:moved":
                    case "object:scaled":
                    case "object:rotated":
                        const currentState = this._getObjectState(action.object, true);
                        action.object.set(action.newState || currentState);
                        action.object.setCoords();
                        break;
                }
                
                this.canvas.renderAll();
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
                    this.addToHistory({
                        type: "object:removed",
                        object: this.selectedObject
                    });
                    
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
                            this.addToHistory({
                                type: "object:added",
                                object: obj
                            });
                        });
                        clonedObj.setCoords();
                    } else {
                        this.canvas.add(clonedObj);
                        this.addToHistory({
                            type: "object:added",
                            object: clonedObj
                        });
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
                            
                            this.addToHistory({
                                type: "object:added",
                                object: fabricImage
                            });
                            
                            this.canvas.setActiveObject(fabricImage);
                            this.canvas.requestRenderAll();
                        };
                    };
                    reader.readAsDataURL(file);
                }
            },

            handleKeyDown(e) {
                if ((e.ctrlKey || e.metaKey) && e.key === "z") {
                    e.preventDefault();
                    this.undo();
                    return;
                }
                
                if ((e.ctrlKey || e.metaKey) && e.key === "y") {
                    e.preventDefault();
                    this.redo();
                    return;
                }
                
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
                                
                                this.addToHistory({
                                    type: "object:added",
                                    object: fabricImage
                                });
                                
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
</style>