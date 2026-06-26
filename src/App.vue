<template>
  <div class="container">
    <!-- 控制後台：上傳圖片 -->
    <!-- 畫布主體 (固定 9:16 比例) -->
    <div 
      class="canvas-container" 
      :style="{ backgroundImage: `url(${bgImage})` }"
      @mousemove="handleMouseMove"
      @mouseleave="handleMouseLeave"
    >
      <!-- 【左上角】：重來箭頭按鈕 -->
      <button class="ui-btn btn-top-left" @click="resetAll" title="全部重置">
        <svg viewBox="0 0 24 24" width="18" height="18" fill="none" stroke="currentColor" stroke-width="2.5">
          <path d="M21.5 2v6h-6M21.34 15.57a10 10 0 1 1-.57-8.38l5.67-5.67"/>
        </svg>
      </button>

      <!-- 【右上角】：上傳 + 號按鈕 -->
      <label class="ui-btn btn-top-right" title="上傳背景圖片">
        <input type="file" accept="image/*" @change="handleImageUpload" />
        <svg viewBox="0 0 24 24" width="18" height="18" fill="none" stroke="currentColor" stroke-width="2.5">
          <line x1="12" y1="5" x2="12" y2="19"></line>
          <line x1="5" y1="12" x2="19" y2="12"></line>
        </svg>
      </label>

      <!-- 狀態提示 -->
      <div v-if="loading" class="status-overlay">Uploading image......</div>
      <div v-if="!bgImage" class="placeholder-text">Created by @5433_33</div>

      <!-- 線條畫布 -->
      <canvas ref="lineCanvasRef" :width="canvasWidth" :height="canvasHeight" class="line-canvas"></canvas>

      <!-- 動態濾鏡方塊 -->
      <div
        v-for="block in blocks"
        :key="block.id"
        class="filter-block"
        :class="[
          block.filterType, 
          { 'filter-active': block.isActive, 'is-dragging': draggedBlockId === block.id }
        ]"
        :style="{
          width: block.width + 'px',
          height: block.height + 'px',
          left: block.x + 'px',
          top: block.y + 'px',
          animationDuration: block.interval + 'ms'
        }"
      >
        <div class="corner tl"></div>
        <div class="corner tr"></div>
        <div class="corner bl"></div>
        <div class="corner br"></div>
        <span class="block-label">{{ block.label }}</span>
      </div>
    </div>

    <!-- SVG 濾鏡定義（用於顆粒、銳化等複雜效果） -->
     <svg style="position: absolute; width: 0; height: 0;">
      <!-- 馬賽克濾鏡 (Pixelate) -->
      <filter id="svg-mosaic" x="0%" y="0%" width="100%" height="100%" filterUnits="objectBoundingBox">
        <!-- 核心：使用 feMorphology 進行侵蝕 (erode) -->
        <!-- radius 越大，像素方塊越大 -->
        <feMorphology operator="erode" in="SourceGraphic" radius="5" />
        <feComponentTransfer>
          <feFuncA type="linear" slope="0.8"/> <!-- 微調 Alpha 通道，讓背景微微透出，增強質感 -->
        </feComponentTransfer>
        <feComposite operator="in" in2="SourceGraphic" />
      </filter>
      <filter id="svg-grain">
        <feTurbulence type="fractalNoise" baseFrequency="0.80" numOctaves="3" stitchTiles="stitch" />
        <feColorMatrix type="matrix" values="1 0 0 0 0  0 1 0 0 0  0 0 1 0 0  0 0 0 0.15 0" />
        <feComposite operator="in" in2="SourceGraphic" />
      </filter>
      <filter id="svg-sharpen">
        <feConvolveMatrix order="3" kernelMatrix="0 -1 0 -1 5 -1 0 -1 0" />
      </filter>
    </svg>
  </div>
</template>

<script setup lang="ts">
import { ref, onMounted, onUnmounted } from 'vue';

// --- 型別定義 ---
interface FilterBlock {
  id: number;
  label: string;
  filterType: 'mosaic' | 'hueRotate' | 'invert' | 'grayscale'|'invert'  ;
  width: number;
  height: number;
  x: number;
  y: number;
  vx: number; // X軸速度
  vy: number; // Y軸速度
  interval: number; // 閃爍頻率 (ms)
  isActive: boolean; // 目前是否有濾鏡
  lastToggle: number; // 上次閃爍時間
}

// 文字粒子的型別定義
interface TextParticle {
  id: number;
  text: string;
  baseX: number;     // 升起的中心軸 X 座標
  x: number;         // 實際繪製的 X 座標
  y: number;         // 實際繪製的 Y 座標
  speedY: number;    // 向上飄移的速度
  angle: number;     // 用於計算左右擺動（正弦波）的弧度
  angleSpeed: number;// 擺動頻率速度
  amplitude: number; // 左右擺動的振幅（隨高度增加而擴大）
  opacity: number;   // 透明度（到頂部淡出）
  fontSize: number;  // 字體大小
  color: string;     // 隨機濾鏡主題色
}

// --- 響應式變數 ---
const bgImage = ref<string>('');
const loading = ref<boolean>(false);
const blocks = ref<FilterBlock[]>([]);

// 畫布固定尺寸 (9:16 比例)
const canvasWidth = 450;
const canvasHeight = 800;

// 滑鼠相關狀態
const draggedBlockId = ref<number | null>(null); // 目前被滑鼠吸附的方塊 ID
const mousePos = { x: 0, y: 0 };
const isMouseInCanvas = ref<boolean>(false);

const ATTRACTION_RADIUS = 120; // 吸附半徑
const SPRING_STIFFNESS = 0.15; // 吸附平滑度

// 文字粒子變數
const particles = ref<TextParticle[]>([]);
let nextParticleId = 0;
let lastSpawnTime = 0;

// 字體顏色庫：復古繽紛感
const textColors = ['#f1c40f', '#e74c3c', '#2ecc71', '#3498db', '#9b59b6', '#ecf0f1'];

// 線條 Canvas 的 Ref
const lineCanvasRef = ref<HTMLCanvasElement | null>(null);
const MAX_CONNECT_DISTANCE = 250; // 線條連接的最大距離

// --- 全部刷新重置邏輯 ---
const resetAll = () => {
  bgImage.value = '';
  draggedBlockId.value = null;
  particles.value = []; // 重置時清空所有文字
  initBlocks();
  
  const canvas = lineCanvasRef.value;
  if (canvas) {
    const ctx = canvas.getContext('2d');
    ctx?.clearRect(0, 0, canvasWidth, canvasHeight);
  }
};

// 滑鼠事件處理函式
const handleMouseMove = (event: MouseEvent) => {
  isMouseInCanvas.value = true;
  const rect = (event.currentTarget as HTMLElement).getBoundingClientRect();
  mousePos.x = event.clientX - rect.left;
  mousePos.y = event.clientY - rect.top;
};

const handleMouseLeave = () => {
  isMouseInCanvas.value = false;
  draggedBlockId.value = null; // 滑鼠離開畫布就釋放吸附
};

// --- 圖片處理 (壓縮並裁剪為 9:16) ---
const handleImageUpload = (event: Event) => {
  const input = event.target as HTMLInputElement;
  // 💡 修正點：確保 input.files 存在且裡面真的有第一個檔案
  if (!input.files || input.files.length === 0 || !input.files[0]) return;

  loading.value = true;
  const file = input.files[0]; // 此時 TypeScript 確信它絕對是 File 型別（繼承自 Blob）
  const reader = new FileReader();

  reader.onload = (e) => {
    const img = new Image();
    img.onload = () => {
      const canvas = document.createElement('canvas');
      canvas.width = canvasWidth;
      canvas.height = canvasHeight;
      const ctx = canvas.getContext('2d');

      if (ctx) {
        const targetRatio = canvasWidth / canvasHeight;
        const imgRatio = img.width / img.height;
        let sx = 0, sy = 0, sw = img.width, sh = img.height;

        if (imgRatio > targetRatio) {
          sw = img.height * targetRatio;
          sx = (img.width - sw) / 2;
        } else {
          sh = img.width / targetRatio;
          sy = (img.height - sh) / 2;
        }

        ctx.drawImage(img, sx, sy, sw, sh, 0, 0, canvasWidth, canvasHeight);
        bgImage.value = canvas.toDataURL('image/jpeg', 0.9);
      }
      loading.value = false;
    };
    img.src = e.target?.result as string;
  };
  reader.readAsDataURL(file);
};

// --- 初始化方塊數據 ---
const initBlocks = () => {
  const types: FilterBlock['filterType'][] = ['mosaic', 'hueRotate', 'invert', 'grayscale'];
  const labels = ['Mosaic', 'HueRotate', 'Invert', 'Grayscale'];

  blocks.value = types.map((type, index) => {
    const width = Math.random() * 60 + 100;  // 100 ~ 160
    const height = Math.random() * 60 + 100; // 100 ~ 160
    return {
      id: index,
      // 💡 修正點：加上 || '' 確保萬一超出索引時，拿到的絕對是 string 而不是 undefined
      label: labels[index] || 'Filter', 
      filterType: type,
      width,
      height,
      x: Math.random() * (canvasWidth - width),
      y: Math.random() * (canvasHeight - height),
      vx: (Math.random() - 0.5) * 2, 
      vy: (Math.random() - 0.5) * 2,
      interval: Math.random() * 700 + 300, 
      isActive: true,
      lastToggle: Date.now()
    };
  });
};

// --- 物理移動與碰撞偵測迴圈 ---
let animationFrameId: number;

const updatePhysics = () => {
  const now = Date.now();

  // --- A. 滑鼠吸附偵測邏輯 ---
  if (isMouseInCanvas.value && draggedBlockId.value === null) {
    let closestId: number | null = null;
    let minDistance = ATTRACTION_RADIUS;

    blocks.value.forEach((block) => {
      const centerX = block.x + block.width / 2;
      const centerY = block.y + block.height / 2;
      const dist = Math.hypot(mousePos.x - centerX, mousePos.y - centerY);

      if (dist < minDistance) {
        minDistance = dist;
        closestId = block.id;
      }
    });

    if (closestId !== null) {
      draggedBlockId.value = closestId;
    }
  }

  // --- B. 更新各個方塊位置與速度 ---
  blocks.value.forEach((block, i) => {
    if (now - block.lastToggle > block.interval) {
      block.isActive = !block.isActive;
      block.lastToggle = now;
    }

    if (draggedBlockId.value === block.id) {
      const targetX = mousePos.x - block.width / 2;
      const targetY = mousePos.y - block.height / 2;

      block.vx = (targetX - block.x) * SPRING_STIFFNESS;
      block.vy = (targetY - block.y) * SPRING_STIFFNESS;

      block.x += block.vx;
      block.y += block.vy;

      const centerX = block.x + block.width / 2;
      const centerY = block.y + block.height / 2;
      const dist = Math.hypot(mousePos.x - centerX, mousePos.y - centerY);
      if (dist > ATTRACTION_RADIUS + 30) {
        draggedBlockId.value = null;
      }
    } else {
      block.x += block.vx;
      block.y += block.vy;

      if (Math.abs(block.vx) < 0.2 && Math.abs(block.vy) < 0.2) {
        block.vx = (Math.random() - 0.5) * 2;
        block.vy = (Math.random() - 0.5) * 2;
      }
    }

    // C. 邊界碰撞偵測
    if (block.x <= 0) { block.x = 0; block.vx = Math.abs(block.vx); }
    if (block.x + block.width >= canvasWidth) { block.x = canvasWidth - block.width; block.vx = -Math.abs(block.vx); }
    if (block.y <= 0) { block.y = 0; block.vy = Math.abs(block.vy); }
    if (block.y + block.height >= canvasHeight) { block.y = canvasHeight - block.height; block.vy = -Math.abs(block.vy); }

    // D. 方塊間相互碰撞偵測
    // D. 方塊間相互碰撞偵測
    for (let j = i + 1; j < blocks.value.length; j++) {
      const other = blocks.value[j];
      
      // 💡 修正點：如果 other 不存在（undefined），直接跳過，消除 TS18048 錯誤
      if (!other) continue;

      const overlapX = Math.min(block.x + block.width, other.x + other.width) - Math.max(block.x, other.x);
      const overlapY = Math.min(block.y + block.height, other.y + other.height) - Math.max(block.y, other.y);

      if (overlapX > 0 && overlapY > 0) {
        if (draggedBlockId.value === block.id) {
          if (overlapX < overlapY) {
            other.x += block.x < other.x ? overlapX : -overlapX;
            other.vx = -other.vx * 1.2;
          } else {
            other.y += block.y < other.y ? overlapY : -overlapY;
            other.vy = -other.vy * 1.2;
          }
          continue; 
        }
        if (draggedBlockId.value === other.id) {
          if (overlapX < overlapY) {
            block.x -= block.x < other.x ? overlapX : -overlapX;
            block.vx = -block.vx * 1.2;
          } else {
            block.y -= block.y < other.y ? overlapY : -overlapY;
            block.vy = -block.vy * 1.2;
          }
          continue;
        }

        if (overlapX < overlapY) {
          const push = overlapX / 2;
          if (block.x < other.x) { block.x -= push; other.x += push; } 
          else { block.x += push; other.x -= push; }
          const tempVx = block.vx; block.vx = other.vx; other.vx = tempVx;
        } else {
          const push = overlapY / 2;
          if (block.y < other.y) { block.y -= push; other.y += push; } 
          else { block.y += push; other.y -= push; }
          const tempVy = block.vy; block.vy = other.vy; other.vy = tempVy;
        }

        const maxSpeed = 2;
        block.vx = Math.max(-maxSpeed, Math.min(maxSpeed, block.vx));
        block.vy = Math.max(-maxSpeed, Math.min(maxSpeed, block.vy));
        other.vx = Math.max(-maxSpeed, Math.min(maxSpeed, other.vx));
        other.vy = Math.max(-maxSpeed, Math.min(maxSpeed, other.vy));
      }
    }
          

  // --- E. 畫布渲染 (文字粒子與線條共用同一個 Canvas 上下文) ---
  const canvas = lineCanvasRef.value;
  if (canvas) {
    const ctx = canvas.getContext('2d');
    if (ctx) {
      // 每一幀都要清空畫布
      ctx.clearRect(0, 0, canvasWidth, canvasHeight);

      // F-1. 動態生成新文字粒子 (僅在有背景圖時生成)
      if (now - lastSpawnTime > Math.random() * 800 + 500 && bgImage.value) {
        const filterWords = ['MOSAIC', 'HUE ROTATE', 'SHARPEN', 'GRAYSCALE', 'INVERT'];
        const randomWord = filterWords[Math.floor(Math.random() * filterWords.length)];
        
        particles.value.push({
          id: nextParticleId++,
          text: randomWord,
          baseX: canvasWidth / 2 + (Math.random() - 0.5) * 20, 
          x: canvasWidth / 2,
          y: canvasHeight + 20,                               
          speedY: Math.random() * 0.4 + 0.3,                  
          angle: Math.random() * Math.PI * 2,                 
          angleSpeed: Math.random() * 0.03 + 0.02,            
          amplitude: 10,                                      
          opacity: 1.0,
          fontSize: Math.floor(Math.random() * 4) + 8,       
          color: textColors[Math.floor(Math.random() * textColors.length)]
        });
        lastSpawnTime = now;
      }

      // F-2. 更新與繪製文字粒子
      for (let k = particles.value.length - 1; k >= 0; k--) {
        const p = particles.value[k];

        p.y -= p.speedY;        
        p.angle += p.angleSpeed; 

        const heightFactor = (canvasHeight - p.y) / canvasHeight; 
        p.amplitude = 15 + heightFactor * 110; 
        p.x = p.baseX + Math.sin(p.angle) * p.amplitude;

        // 頂部淡出邏輯
        if (p.y < 400) {
          p.opacity = p.y / 400; 
        }

        // 邊界溢出檢查與移除
        if (p.y < -20 || p.opacity <= 0) {
          particles.value.splice(k, 1);
          continue;
        }

        // 開始繪製單個文字
        ctx.save();
        ctx.globalAlpha = p.opacity;
        ctx.font = `bold ${p.fontSize}px Georgia, sans-serif`; // 💡 1. 改為襯線字體
        ctx.fillStyle = '#000000';                        // 💡 2. 固定為純黑色
        ctx.textAlign = 'center';

        // 💡 3. 移除了 strokeText 兩行代碼（去邊框）
        ctx.fillText(p.text, p.x, p.y); 
        ctx.restore();
      }

      // F-3. 繪製方塊之間的連接線條
      for (let i = 0; i < blocks.value.length; i++) {
        for (let j = i + 1; j < blocks.value.length; j++) {
          const b1 = blocks.value[i];
          const b2 = blocks.value[j];

          const c1x = b1.x + b1.width / 2;
          const c1y = b1.y + b1.height / 2;
          const c2x = b2.x + b2.width / 2;
          const c2y = b2.y + b2.height / 2;

          const dist = Math.hypot(c2x - c1x, c2y - c1y);

          if (dist < MAX_CONNECT_DISTANCE) {
            ctx.beginPath();
            ctx.moveTo(c1x, c1y);
            ctx.lineTo(c2x, c2y);

            const alpha = (1 - dist / MAX_CONNECT_DISTANCE) * 0.4; 
            ctx.strokeStyle = `rgba(0, 0, 0, ${alpha})`;
            ctx.lineWidth = 1.5;
            ctx.stroke();
          }
        }
      }
    }
  }

  animationFrameId = requestAnimationFrame(updatePhysics);
};

// --- 生命週期 ---
onMounted(() => {
  initBlocks();
  animationFrameId = requestAnimationFrame(updatePhysics);
});

onUnmounted(() => {
  cancelAnimationFrame(animationFrameId);
});
</script>

<style scoped>
/* ... Container 與 Canvas Container 保持不變 ... */

/* --- 【新增/修改】黑框白底方形按鈕樣式 --- */
.ui-btn {
  position: absolute;
  top: 15px;
  width: 36px;
  height: 36px;
  background-color: #ffffff; /* 白底 */
  border: 2px solid #000000;  /* 黑框 */
  color: #000000;             /* 黑色圖案 */
  display: flex;
  align-items: center;
  justify-content: center;
  cursor: pointer;
  z-index: 20;                /* 確保按鈕在最上層，不會被方塊遮擋 */
  padding: 0;
  box-sizing: border-box;
  transition: background-color 0.1s, transform 0.05s;
}

/* 按鈕懸停與點擊的微互動 */
.ui-btn:hover {
  background-color: #f0f0f0;
}
.ui-btn:active {
  transform: scale(0.95);
}

/* 定位：左上與右上 */
.btn-top-left {
  left: 15px;
  border-radius: 0px; /* 嚴格方形，無圓角 */
}
.btn-top-right {
  right: 15px;
  border-radius: 0px; /* 嚴格方形，無圓角 */
}
.btn-top-right input {
  display: none;      /* 隱藏原生 file input */
}

/* 狀態提示與微調 */
.status-overlay {
  position: absolute;
  top: 60px;
  left: 50%;
  transform: translateX(-50%);
  background: rgba(0,0,0,0.8);
  padding: 5px 10px;
  font-size: 12px;
  z-index: 15;
}

/* ... 原本的 line-canvas, filter-block 等樣式保持不變 ... */
/* 全域與外殼樣式 */
.container {
  display: flex;
  flex-direction: column;
  align-items: center;
  background-color: #121212;
  color: #fff;
  min-height: 100vh;
  font-family: sans-serif;
  padding: 20px;
}

.controls {
  margin-bottom: 15px;
  display: flex;
  gap: 15px;
  align-items: center;
}

.upload-btn {
  background: #3498db;
  padding: 10px 20px;
  border-radius: 5px;
  cursor: pointer;
  font-weight: bold;
  transition: background 0.2s;
}
.upload-btn:hover { background: #2980b9; }
.upload-btn input { display: none; }

/* 9:16 主畫布 */
.canvas-container {
  position: relative;
  width: 450px;
  height: 800px;
  background-color: #222;
  background-size: cover;
  background-position: center;
  border: 2px solid #333;
  border-radius: 8px;
  overflow: hidden;
  box-shadow: 0 10px 30px rgba(0,0,0,0.5);
}

.placeholder-text {
  position: absolute;
  top: 50%;
  left: 50%;
  transform: translate(-50%, -50%);
  color: #666;
}

/* 核心：濾鏡方塊與動態套用 (backdrop-filter) */
.filter-block {
  position: absolute;
  border: 1px solid rgba(255, 255, 255, 0.6);
  box-sizing: border-box;
  transition: backdrop-filter 0.1s ease; /* 讓閃爍稍微平滑一點 */
  pointer-events: none; /* 讓滑鼠不會干擾碰撞 */
}

/* 各濾鏡類型在啟用(filter-active)時的效果 */
.mosaic.filter-active     { backdrop-filter: url(#svg-mosaic); }
.grayscale.filter-active { backdrop-filter: grayscale(120%); }
.invert.filter-active    { backdrop-filter: invert(100%); }
.sharpen.filter-active   { backdrop-filter: url(#svg-sharpen); }
.hueRotate.filter-active  { backdrop-filter: hue-rotate(180deg); }

/* 方塊標籤與四角裝飾（還原 image_27d2e8.jpg 的設計感） */
.block-label {
  position: absolute;
  bottom: -22px;
  left: 0;
  background: rgba(0,0,0,0.7);
  color: #fff;
  font-size: 11px;
  padding: 2px 6px;
  white-space: nowrap;
  border-radius: 2px;
}

.corner {
  position: absolute;
  width: 6px;
  height: 6px;
  background: #fff;
  border: 1px solid #000;
}
/* 當方塊被滑鼠吸附時的視覺樣式 */
.filter-block.is-dragging {
  border: 1.5px solid #000000; /* 變成藍色邊框 */
  box-shadow: 0 0 15px rgba(52, 152, 219, 0.4);
  z-index: 10; /* 確保被吸附的方塊在最上層 */
}
.filter-block.is-dragging .corner {
  background: #3498db;
}

/* 【新增】線條畫布樣式 */
.line-canvas {
  position: absolute;
  top: 0;
  left: 0;
  width: 100%;
  height: 100%;
  pointer-events: none; /* 確保滑鼠事件可以穿透它，不影響方塊的滑鼠吸附 */
  z-index: 1;           /* 讓它在背景圖之上，但在 filter-block (預設 z-index 或 10) 之下 */
}

/* 確保方塊的 z-index 比線條高 */
.filter-block {
  position: absolute;
  border: 1px solid rgba(0, 0, 0, 0.1);
  box-sizing: border-box;
  pointer-events: none;
  z-index: 2; /* 【新增】確保濾鏡方塊壓在線條上面，這樣線條也會被方塊的濾鏡影響！ */
}
.tl { top: -3px; left: -3px; }
.tr { top: -3px; right: -3px; }
.bl { bottom: -3px; left: -3px; }
.br { bottom: -3px; right: -3px; }
</style>