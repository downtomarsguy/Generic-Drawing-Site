<template>
    <div class="">
        <canvas id="canvas-container">

        </canvas>
    </div>
</template>

<script>
    
    export default {
        data() {
            return {
                width: window.innerWidth,
                height: window.innerHeight,
                painting: false,
                context: null,
            }
        },

        mounted() {
            const canvas = document.getElementById("canvas-container");
            this.context = canvas.getContext("2d");
            canvas.height = this.height;
            canvas.width = this.width;

            window.addEventListener("resize", this.handleResize);
            canvas.addEventListener("mousedown", this.startPosition);
            canvas.addEventListener("mouseup", this.endPosition);
            canvas.addEventListener("mousemove", this.draw);
        },

        methods: {
            handleResize() {
                this.width = window.innerWidth;
                this.height = window.innerHeight;
            },
            
            startPosition() {
                this.painting = true;
            },

            endPosition() {
                this.painting = false;
                this.context.beginPath();
            },

            draw(e) {
                if(!this.painting) return;
                
                this.context.lineWidth = 10;
                this.context.lineCap = "round";

                this.context.lineTo(e.clientX, e.clientY);
                this.context.stroke();
            }
        }
    }
    
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
</style>