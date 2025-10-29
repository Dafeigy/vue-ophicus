<script setup lang="ts">
import QrScanner from 'qr-scanner'; 
import Camera from './Camera.vue'
import { encode, decode } from 'js-base64';
import {
  renderSVG,
} from 'uqr'
import { inject, ref, onUnmounted, nextTick, Ref } from 'vue';

// 从父组件App.vue注入切换模式函数
const handleSwitchMode = inject('handleSwitchMode') as (event: MouseEvent) => void;
const isStartStreaming = false;
const isMobile = ref(window.innerWidth < 768); // 768px以下视为移动设备
const file = ref(true);
const bitRATE =ref(0.0)
const tranFPS = ref(20);
const totalBlocks = ref(0)
const transBlockIndices = ref([]);

// 摄像头相关变量
const isCameraActive = ref(false);
const videoRef = ref<HTMLVideoElement | null>(null);
const canvasRef = ref<HTMLCanvasElement | null>(null); // 用于扫描的canvas
const displayCanvasRef = ref<HTMLCanvasElement | null>(null); // 用于显示的canvas
const lastScanResult = ref<string | null>(null);
// 存储二维码扫描结果
const scanResult = ref('')

// 扫描二维码
const scanQRCode = async () => {
  if (!videoRef.value || !canvasRef.value || !displayCanvasRef.value) return;
  
  const scanCanvas = canvasRef.value;
  const displayCanvas = displayCanvasRef.value;
  const ctx = scanCanvas.getContext('2d', {willReadFrequently: true});
  
  if (!ctx) return;
  
  // 设置canvas尺寸
  const videoWidth = videoRef.value.videoWidth || 640;
  const videoHeight = videoRef.value.videoHeight || 480;
  
  scanCanvas.width = videoWidth;
  scanCanvas.height = videoHeight;
  
  // 绘制当前视频帧到canvas
  ctx.drawImage(videoRef.value, 0, 0, scanCanvas.width, scanCanvas.height);
  
  // 计算扫描框位置和大小（与updateDisplayCanvas中的计算保持一致）
  const scanBoxSize = Math.min(scanCanvas.width, scanCanvas.height) * 0.7;
  const scanBoxX = (scanCanvas.width - scanBoxSize) / 2;
  const scanBoxY = (scanCanvas.height - scanBoxSize) / 2;
  
  // 创建一个临时canvas用于裁剪扫描框内的图像
  const tempCanvas = document.createElement('canvas');
  tempCanvas.width = scanBoxSize;
  tempCanvas.height = scanBoxSize;
  const tempCtx = tempCanvas.getContext('2d');
  
  if (!tempCtx) return;
  
  try {
    // 从原canvas裁剪出扫描框内的图像
    tempCtx.drawImage(
      scanCanvas, 
      scanBoxX, scanBoxY, scanBoxSize, scanBoxSize, // 源区域
      0, 0, scanBoxSize, scanBoxSize // 目标区域
    );
    
    // 使用qr-scanner识别裁剪后的图像中的二维码
    const result = await QrScanner.scanImage(tempCanvas, {
      returnDetailedScanResult: true
    });
    
    if (result && result.data) {
      console.log('扫描到二维码:', result.data);
      lastScanResult.value = result.data;
      
      // 在显示canvas上标记二维码位置（需要将裁剪后的坐标转换回原始坐标）
      const displayCtx = displayCanvas.getContext('2d');
      if (displayCtx && (result as any).location) {
        displayCtx.strokeStyle = 'rgba(255, 255, 0, 0.8)';
        displayCtx.lineWidth = 3;
        displayCtx.beginPath();
        
        // 转换坐标：裁剪区域内的坐标 + 裁剪区域的偏移量
        displayCtx.moveTo(
          (result as any).location.topLeftCorner.x + scanBoxX,
          (result as any).location.topLeftCorner.y + scanBoxY
        );
        displayCtx.lineTo(
          (result as any).location.topRightCorner.x + scanBoxX,
          (result as any).location.topRightCorner.y + scanBoxY
        );
        displayCtx.lineTo(
          (result as any).location.bottomRightCorner.x + scanBoxX,
          (result as any).location.bottomRightCorner.y + scanBoxY
        );
        displayCtx.lineTo(
          (result as any).location.bottomLeftCorner.x + scanBoxX,
          (result as any).location.bottomLeftCorner.y + scanBoxY
        );
        displayCtx.closePath();
        displayCtx.stroke();
        
        // 显示扫描成功提示
        scanSuccess.value = true;
        setTimeout(() => {
          scanSuccess.value = false;
        }, 2000); // 2秒后隐藏提示
      }
    }
  } catch (error) {
    // 解码错误，继续下一次扫描
    // console.error('二维码扫描错误:', error);
  } finally {
    // 清理临时canvas（虽然GC会处理，但显式清理是好习惯）
    tempCanvas.remove();
  }
};

// 更新显示canvas，显示摄像头画面
const updateDisplayCanvas = () => {
  if (!videoRef.value || !displayCanvasRef.value) return;
  
  const displayCanvas = displayCanvasRef.value;
  const displayCtx = displayCanvas.getContext('2d');
  
  if (!displayCtx) return;
  
  // 设置canvas尺寸
  const videoWidth = videoRef.value.videoWidth || 640;
  const videoHeight = videoRef.value.videoHeight || 480;
  
  // 设置显示canvas尺寸
  displayCanvas.width = videoWidth;
  displayCanvas.height = videoHeight;
  
  // 绘制视频帧到canvas
  const drawVideo = () => {
    if (!isCameraActive.value || !videoRef.value) return;
    
    displayCtx.drawImage(videoRef.value, 0, 0, displayCanvas.width, displayCanvas.height);
    
    // 绘制扫描框到显示canvas
    displayCtx.strokeStyle = 'rgba(0, 255, 0, 0.8)';
    displayCtx.lineWidth = 2;
    
    // 计算扫描框位置和大小
    const scanBoxSize = Math.min(displayCanvas.width, displayCanvas.height) * 0.7;
    const scanBoxX = (displayCanvas.width - scanBoxSize) / 2;
    const scanBoxY = (displayCanvas.height - scanBoxSize) / 2;
    
    // 绘制扫描框
    displayCtx.strokeRect(scanBoxX, scanBoxY, scanBoxSize, scanBoxSize);
    
    // 绘制四个角
    const cornerSize = 20;
    displayCtx.lineWidth = 4;
    
    // 左上角
    displayCtx.beginPath();
    displayCtx.moveTo(scanBoxX, scanBoxY + cornerSize);
    displayCtx.lineTo(scanBoxX, scanBoxY);
    displayCtx.lineTo(scanBoxX + cornerSize, scanBoxY);
    displayCtx.stroke();
    
    // 右上角
    displayCtx.beginPath();
    displayCtx.moveTo(scanBoxX + scanBoxSize - cornerSize, scanBoxY);
    displayCtx.lineTo(scanBoxX + scanBoxSize, scanBoxY);
    displayCtx.lineTo(scanBoxX + scanBoxSize, scanBoxY + cornerSize);
    displayCtx.stroke();
    
    // 左下角
    displayCtx.beginPath();
    displayCtx.moveTo(scanBoxX, scanBoxY + scanBoxSize - cornerSize);
    displayCtx.lineTo(scanBoxX, scanBoxY + scanBoxSize);
    displayCtx.lineTo(scanBoxX + cornerSize, scanBoxY + scanBoxSize);
    displayCtx.stroke();
    
    // 右下角
    displayCtx.beginPath();
    displayCtx.moveTo(scanBoxX + scanBoxSize - cornerSize, scanBoxY + scanBoxSize);
    displayCtx.lineTo(scanBoxX + scanBoxSize, scanBoxY + scanBoxSize);
    displayCtx.lineTo(scanBoxX + scanBoxSize, scanBoxY + scanBoxSize - cornerSize);
    displayCtx.stroke();
    
    if (isCameraActive.value) {
      requestAnimationFrame(drawVideo);
    }
  };
  
  requestAnimationFrame(drawVideo);
};

// 测试用变量
const scanSuccess = ref(false);
const testQRCodeContent = ref('Hello World!');
const generatedQRCode = ref('');
const directScanResult = ref('');
const directScanStatus = ref('');
const testCanvasRef = ref<HTMLCanvasElement | null>(null);

const handleQRScanned = (content) => {
  console.log('接收到扫描结果:', content)
  scanResult.value = content
}

</script>

<template>
    <div id="con" class="text-theme xl:aspect-video h-full w-full max-w-[2160px] mx-auto flex flex-col md:flex-row items-center bg-[#202020] p-2 sm:border-0 md:border-0 justify-start">
        <div id="left" class="w-full h-2/5 xl:w-[50%] md:h-[80%] lg:h-[80%] flex flex-col xl:px-8 mb-2 lg:mb-0 overflow-hidden justify-center md:justify-start xl:justify-center">
            <div id="TODO" class="w-full py-2 xl:py-4 px-4 items-center xl:text-2xl text-xl flex text-green font-display font-bold bg-theme lg:text-3xl">
                PROJECT OPHICULUS [R]
            </div>
            <div id="status" class=" w-full flex justify-center font-display flex-col" >
                <div class="card-header font-display lg:text-2xl bg-orange px-4 mt-[2%]">
                    ▧ DECODE STATUS▸
                </div>
                <div id="details" class="grid grid-cols-5 mt-[1%] px-4 gap-1 sm:gap-2 text-sm sm:text-base">
                    <p class="bg-green text-theme px-1 col-span-2 ">▣ FILENAME:</p> 
                        <p class="col-span-2 sm:col-span-2 truncate overflow-hidden whitespace-nowrap" v-if="isCameraActive"> 占位 </p>
                        <p class="col-span-3 " v-else> ..?Camera</p>
                    <p class="bg-green text-theme px-1 col-span-2 ">▣ BYTES:</p>
                        <p class="col-span-3 " v-if="file" >..? BYTES</p>
                        <p class="col-span-3 " v-else>0 Bytes</p>
                    <p class="bg-green text-theme px-1 col-span-2 ">▣ TOTAL:</p>
                        <p class="col-span-3 " v-if="file">..? Length</p>
                        <p class="col-span-3 " v-else>0 </p>
                    <p class="bg-theme text-green px-1 col-span-2 select-none xl:flex ">▣ INDICES</p>
                        <p class="col-span-3 xl:flex" v-if="file && transBlockIndices.length > 0">{{ transBlockIndices[transBlockIndices.length - 1] }}</p>
                        <p class="col-span-3 xl:flex" v-else>[ ]</p>
                    <p class="bg-theme text-green px-1 col-span-2 select-none">▣ BITRATE</p >
                        <p class="col-span-3 " v-if="file">{{ bitRATE }} bit/s</p>
                        <p class="col-span-3 " v-else>0.0 bits/s</p>
                    <p class="bg-theme text-green px-1 col-span-2 select-none">▣ FPS</p>
                        <p class="col-span-3 " v-if="file">{{ tranFPS}} </p>
                        <p class="col-span-3 " v-else>0</p>
                </div>
                <div class="card-header font-display lg:text-2xl bg-orange px-4 mt-[2%] hidden xl:flex">
                    ▧ BLOCKS STATUS▸
                </div>
                <div id="notrans" class="xl:grid xl:grid-cols-30 mt-[2%] px-2 border rounded-md text-center min-h-[calc(50% - 10px)] sm:max-h-[100px] md:min-h-[150px] flex items-center justify-center">
                  <div class="col-span-30 flex items-center justify-center text-green text-xl animate-blink select-none">WAITING FOR FILE BLOCKS ... ...</div>
                </div>
                <!-- <div id="transblocks" v-show="!isMobile" class="hidden xl:grid xl:grid-cols-30 mt-[2%] px-2 border rounded-2xl overflow-y-auto" style="max-height: 150px; scrollbar-color: transparent transparent; overflow-x: hidden;">
                  <div v-for="_ in chunks.length" :key="_" 
                      class="bg-[#343536] text-theme text-[1vmin] m-1 flex aspect-square rounded justify-center items-center transition-all duration-80 ease-in-out"
                      :class="{
                        transactive: currentTransmittingIndices.includes(_ - 1),
                        'transmitted': transmittedIndices.includes(_ - 1)
                      }">
                    {{ _ }}
                  </div>
                </div> -->
              </div>
        </div>
        <div id="right" class="w-full h-2/5 xl:w-[50%] md:h-[80%] lg:h-[80%] flex flex-col lg:mx-0 items-center px-2 md:justify-center">
            <div id="camera" class="w-full max-w-[340px] items-center flex justify-center">
              <div id="scan-container" class="w-full aspect-square flex justify-center">
                <Camera width="100%" @qr-scanned="handleQRScanned" @camera-error="error => scanResult = `错误: ${error}`"/>
              </div>
              
            </div>
            <div id="debug" class="flex justify-center mt-2 w-3/5">
              <div id="debug-info" class="text-xs truncate text-[#eeeddd] bg-[#333] px-3 py-1.5 rounded-md min-w-[200px] w-full lg:w-1/2 text-center">
                {{ scanResult || 'Somthing just like this with a long string but not long enougth' }}
              </div>
            </div>
            <!-- 按钮控制区域 -->
                <div class="w-full rounded-b-xl py-4 flex items-center justify-center space-x-5">
                  <!-- 左侧：接收端按钮 -->
                  <button
                    @click="handleSwitchMode" 
                    class="flex items-center justify-center gap-2 px-4 py-2 border border-theme rounded-full hover:bg-[#343536] transition-all cursor-pointer"
                  >
                    <span>⇄</span>
                    <span class="text-xs xl:flex hidden">RECEIVER</span>
                </button>
                  <!-- 右侧：选择文件按钮 -->
                  <button 
                    @click="" 
                    class="flex items-center justify-center gap-2 px-4 py-2 border border-theme rounded-full hover:bg-[#343536] transition-all cursor-pointer"
                  >
                    <span>📁</span>
                    <span class="text-xs xl:flex hidden">SELECT</span>
                  </button>
                </div>
        </div>
    </div>

</template>