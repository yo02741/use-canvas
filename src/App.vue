<template>
  <div id="app">
    <div ref="paths" class="paths">
      idx: {{ idx }} / {{ describe }}
      <ul>
        <li
          v-for="(item, index) in reversePaths"
          :key="index"
          class="step-item"
        >
          <span>{{ index === idx ? "原圖" : `第 ${idx - index} 步` }}</span>
          <img
            :src="item"
            alt=""
          >
        </li>
      </ul>
    </div>
    <div class="canvas-container">
      <canvas ref="canvas" width="450" height="468"></canvas>
      <div v-if="isBrushMode" class="brush-group">
        <span>
          畫筆顏色： <input type="color" v-model="brushSetting.strokeStyle">
        </span>
        <span>
          畫筆粗細： <input type="number" v-model="brushSetting.lineWidth">
        </span>
      </div>
      <div v-if="isEraserMode" class="eraser-group">
        <span>
          橡皮擦粗細： <input type="number" v-model="eraserSetting.lineWidth">
        </span>
      </div>
      <div class="button-group">
        <button @click="updateMode('brush')" :class="{ 'is-active': isBrushMode }">畫筆</button>
        <button @click="updateMode('eraser')" :class="{ 'is-active': isEraserMode }">橡皮擦</button>
        
        <button @click="onUndo" :disabled="!idx">上一步</button>
        <button @click="onClear" :disabled="!idx">清除畫布</button>
        
        <button @click="onSaveDrawing">儲存繪畫路徑</button>
        <button @click="onSaveCombine">儲存組合圖片</button>
      </div>
    </div>
  </div>
</template>

<script>
export default {
  data() {
    return {
      isDrawing: false,
      
      mode: "brush", // brush, eraser
      idx: 0,
      
      canvas: null,
      ctx: null,

      drawCanvas: null,
      drawCtx: null,
      previewCanvas: null,
      previewCtx: null,

      brushSetting: {
        strokeStyle: "#ff0000",
        lineWidth: 10,
      },
      eraserSetting: {
        lineWidth: 25,
      },
      paths: [],
    };
  },
  computed: {
    describe() {
      return !this.idx ? "還沒開始畫圖" : `目前已畫了 ${this.idx} 步`
    },
    reversePaths() {
      return [...this.paths].reverse();
    },
    isBrushMode() {
      return this.mode === "brush";
    },
    isEraserMode() {
      return this.mode === "eraser";
    },
  },
  watch: {
    brushSetting: {
      handler() {
        this.updateDrawingSettings();
      },
      deep: true,
    },
  },
  mounted() {
    this.initCanvas();
    this.updateDrawingSettings();
    this.setupAllEventListener();
  },
  beforeDestroy() {
    this.removeAllEventListener();
  },
  methods: {
    // 初始化畫布
    initCanvas() {
      // 獲取canvas元素
      this.canvas = this.$refs.canvas;
      this.ctx = this.canvas.getContext("2d");
      this.paths.push(this.canvas.toDataURL());

      this.drawCanvas = document.createElement("canvas");
      this.previewCanvas = document.createElement("canvas");
      this.drawCtx = this.drawCanvas.getContext("2d");
      this.previewCtx = this.previewCanvas.getContext("2d");
      this.drawCanvas.width = this.previewCanvas.width = this.canvas.width;
      this.drawCanvas.height = this.previewCanvas.height = this.canvas.height;
    },

    // 更新畫筆設定
    updateDrawingSettings() {
      // 設置繪圖樣式
      this.drawCtx.strokeStyle = this.brushSetting.strokeStyle;
      this.drawCtx.lineWidth = this.brushSetting.lineWidth;
      this.drawCtx.lineCap = "round";
    },
    
    updateMode(payload) {
      this.mode = payload;
    },
    
    // 更新是否正在畫
    updateIsDrawing(payload) {
      this.isDrawing = payload;
    },
    
    // 初始化監聽器
    setupAllEventListener() {
      this.canvas.addEventListener("mousedown", (event) => {
        this.updateIsDrawing(true);

        // 設置起始點
        const x = event.pageX - this.canvas.offsetLeft;
        const y = event.pageY - this.canvas.offsetTop;
        this.drawCtx.beginPath();
        this.drawCtx.moveTo(x, y);       
      })
      
      this.canvas.addEventListener("mousemove", (event) => {
        const x = event.pageX - this.canvas.offsetLeft;
        const y = event.pageY - this.canvas.offsetTop;

        if (!this.isDrawing) return;

        switch (this.mode) {
          case "brush":
            this.drawLine(x, y);
            break;
          case "eraser":
            this.clearLine(x, y);
            break;
        }
      })
      
      this.canvas.addEventListener("mouseup", (event) => {
        this.idx += 1;
        this.paths.push(this.canvas.toDataURL());
        this.updateIsDrawing(false);
      })
    },
    
    // 移除監聽器
    removeAllEventListener() {
      // 
    },

    // 繪製線條
    drawLine(x, y) {
      this.drawCtx.lineTo(x, y);
      this.drawCtx.stroke();
      this.ctx.drawImage(this.drawCanvas, 0, 0);
    },

    // 清除線條
    clearLine(x, y) {
      // 原本想這樣實現，但不知道啥無法清除
      // this.drawCtx.clearRect((x - this.eraserSetting.lineWidth / 2), (y - this.eraserSetting.lineWidth / 2), this.eraserSetting.lineWidth, this.eraserSetting.lineWidth)
      // this.ctx.drawImage(this.drawCanvas, 0, 0);

      this.ctx.clearRect((x - this.eraserSetting.lineWidth / 2), (y - this.eraserSetting.lineWidth / 2), this.eraserSetting.lineWidth, this.eraserSetting.lineWidth)
      this.drawCtx.clearRect(0, 0, this.canvas.width, this.canvas.height);
      this.drawCtx.drawImage(this.canvas, 0, 0);
    },

    // 將繪製的部份作為png文件下載
    onSaveDrawing() {
      const link = document.createElement("a");
      link.download = "drawing.png";
      link.href = this.canvas.toDataURL("image/png");
      link.click();
    },
    
    // 將繪製的部份與原圖結合，最後將結果作為png文件下載
    onSaveCombine() {
      // 創建一個新的canvas元素，並將其尺寸設置為與畫布相同
      const combinedCanvas = document.createElement("canvas");
      combinedCanvas.width = this.canvas.width;
      combinedCanvas.height = this.canvas.height;

      // 將背景圖像繪製到新canvas上
      const bgImage = new Image();
      bgImage.crossOrigin = "Anonymous"
      bgImage.src = "https://i.imgur.com/XDNo48G.png";
      bgImage.onload = () => {
        const ctx = combinedCanvas.getContext("2d");
        ctx.drawImage(bgImage, 0, 0, combinedCanvas.width, combinedCanvas.height);

        // 將繪製的路徑繪製到新canvas上
        ctx.drawImage(this.canvas, 0, 0);

        // 下載新canvas作為png文件
        const link = document.createElement("a");
        link.download = "combined.png";
        link.href = combinedCanvas.toDataURL("image/png");
        link.click();
      };
    },
    
    // 點擊 上一步 按鈕
    onUndo() {
      if (this.idx < 1) return;
      
      this.idx -= 1;

      const lastDraw = new Image();
      lastDraw.src = this.paths[this.idx];
      lastDraw.onload = () => {
        this.clearAllCanvas();
        this.ctx.drawImage(lastDraw, 0, 0);
        this.drawCtx.drawImage(lastDraw, 0, 0);
        this.paths.pop();
      }
    },
    
    // 清除畫布
    clearAllCanvas() {
      this.ctx.clearRect(0, 0, this.canvas.width, this.canvas.height);
      this.drawCtx.clearRect(0, 0, this.canvas.width, this.canvas.height);
    },
    
    // 點擊 清除畫布 按鈕
    onClear() {
      this.clearAllCanvas();
      this.idx = 0;
      this.paths.length = 0;
      this.paths.push(this.canvas.toDataURL());
    },
  }
};
</script>

<style>
* {
  box-sizing: border-box;
}
  
html,
body {
  margin: 0;
  padding: 0;
  width: 100%;
  height: 100%;
}

#app {
  width: 100%;
  height: 100%;
  padding: 50px 20px;
  display: flex;
  font-family: Avenir, Helvetica, Arial, sans-serif;
  text-align: center;
  color: #2c3e50;
  overflow: hidden
}
  
.paths {
  flex: 3;
  display: flex;
  flex-direction: column;
  align-items: center;
  padding: 0 20px;
  gap: 10px;
  overflow-y: scroll;
}
  
ul {
  list-style: none;
  display: flex;
  flex-direction: column;
  gap: 10px;
  
}
  
.step-item {
  display: flex;
  justify-content: space-between;
  align-items: center;
  gap: 10px;
}
  
.step-item img {
  width: 50%;
  border: 2px solid #000;
}
  
.canvas-container {
  flex: 7;
  display: flex;
  flex-direction: column;
  justify-content: center;
  align-items: center;
  gap: 15px;
}

a,
button {
  color: #4fc08d;
}

button {
  background: none;
  border: solid 1px;
  border-radius: 2em;
  font: inherit;
  padding: 5px 10px;
  cursor: pointer;
  transition: all .1s;
}
  
button:hover,
button.is-active {
  color: #ffffff;
  background: #4fc08d;
}
  
button:disabled {
  color: #ffffff;
  background-color: #cccccc;
  cursor: not-allowed;
}
  
.button-group {
  display: grid;
  grid-template: repeat(2, 1fr) / repeat(2, 1fr);
  gap: 10px;
}
  
canvas {
  border: 1px solid black;
  background-image: url("https://i.imgur.com/XDNo48G.png");
  background-size: contain;
  background-repeat: no-repeat;
}
</style>