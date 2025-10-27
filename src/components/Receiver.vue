<script setup lang="ts">
import jsQR from "jsqr";
import { encode, decode } from 'js-base64';
import {
  renderSVG,
} from 'uqr'
import { inject, ref, onUnmounted, nextTick, Ref } from 'vue';
import { ElButton } from 'element-plus';

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
const canvasRef = ref<HTMLCanvasElement | null>(null);
const stream: Ref<MediaStream | null> = ref(null);
const scanInterval: Ref<number | null> = ref(null);
const lastScanResult = ref<string | null>(null);

// 初始化摄像头
const initCamera = async () => {
  try {
    // 检查浏览器兼容性
    const navigator = window.navigator;
    const getUserMedia = navigator.mediaDevices?.getUserMedia ||
                         (navigator as any).webkitGetUserMedia ||
                         (navigator as any).mozGetUserMedia ||
                         (navigator as any).msGetUserMedia;
    
    if (!getUserMedia) {
      console.error('您的浏览器不支持摄像头访问，请使用更新版本的浏览器');
      return;
    }

    // 检查安全上下文
    if (window.location.protocol !== 'https:' && window.location.hostname !== 'localhost' && window.location.hostname !== '127.0.0.1') {
      console.warn('警告: 摄像头访问在非HTTPS环境下可能受限，请考虑使用HTTPS');
    }

    // 使用Promise封装旧版API
    const getUserMediaPromise = (constraints: MediaStreamConstraints): Promise<MediaStream> => {
      if (navigator.mediaDevices && navigator.mediaDevices.getUserMedia) {
        return navigator.mediaDevices.getUserMedia(constraints);
      }
      
      return new Promise((resolve, reject) => {
        getUserMedia.call(navigator, constraints, resolve, reject);
      });
    };

    // 获取视频流，不限制摄像头方向
    stream.value = await getUserMediaPromise({
      video: {
        facingMode: 'environment',
        width: { ideal: 1280 },
        height: { ideal: 720 }
      }
    });

    // 显示视频流
    await nextTick();
    if (videoRef.value) {
      videoRef.value.srcObject = stream.value;
      videoRef.value.play();
    }
    
    isCameraActive.value = true;
    console.log('摄像头已成功启动');
    
    // 开始扫描二维码
    startQRCodeScanning();
  } catch (error) {
    console.error('启动摄像头失败:', error);
    if (error instanceof DOMException) {
      if (error.name === 'NotAllowedError') {
        console.error('用户拒绝了摄像头访问权限');
      } else if (error.name === 'NotFoundError') {
        console.error('未找到摄像头设备');
      } else if (error.name === 'NotReadableError') {
        console.error('摄像头已被其他应用占用');
      } else {
        console.error('摄像头错误:', error.message);
      }
    }
    isCameraActive.value = false;
  }
};

// 停止摄像头
const stopCamera = () => {
  if (scanInterval.value) {
    clearInterval(scanInterval.value);
    scanInterval.value = null;
  }
  
  if (stream.value) {
    stream.value.getTracks().forEach(track => track.stop());
    stream.value = null;
  }
  
  if (videoRef.value) {
    videoRef.value.srcObject = null;
  }
  
  isCameraActive.value = false;
  console.log('摄像头已关闭');
};

// 开始扫描二维码
const startQRCodeScanning = () => {
  if (scanInterval.value) {
    clearInterval(scanInterval.value);
  }
  
  // 每500毫秒扫描一次
  scanInterval.value = window.setInterval(() => {
    scanQRCode();
  }, 500);
};

// 扫描二维码
const scanQRCode = () => {
  if (!videoRef.value || !canvasRef.value) return;
  
  const canvas = canvasRef.value;
  const ctx = canvas.getContext('2d');
  
  if (!ctx) return;
  
  // 设置canvas尺寸
  canvas.width = videoRef.value.videoWidth;
  canvas.height = videoRef.value.videoHeight;
  
  // 绘制视频帧到canvas
  ctx.drawImage(videoRef.value, 0, 0, canvas.width, canvas.height);
  
  // 获取图像数据
  const imageData = ctx.getImageData(0, 0, canvas.width, canvas.height);
  
  // 使用jsQR库扫描二维码
  const code = jsQR(imageData.data, canvas.width, canvas.height, {
    inversionAttempts: 'dontInvert',
  });
  
  // 如果扫描到二维码且结果与上次不同
  if (code && code.data && code.data !== lastScanResult.value) {
    console.log('扫描到二维码:', code.data);
    lastScanResult.value = code.data;
  }
};

// 组件销毁时停止摄像头
onUnmounted(() => {
  stopCamera();
});
const svgg = renderSVG("PROJECT OPHICULUS PROJECT OPHICULUSPROJECT OPHICULUSPROJECT OPHICULUSPROJECT OPHICULUSPROJECT OPHICULUSPROJECT OPHICULUSPROJECT OPHICULUSPROJECT OPHICULUSPROJECT OPHICULUSPROJECT OPHICULUSPROJECT OPHICULUSPROJECT OPHICULUSPROJECT OPHICULUSPROJECT OPHICULUSPROJECT OPHICULUSPROJECT OPHICULUSPROJECT OPHICULUSPROJECT OPHICULUSPROJECT OPHICULUSPROJECT OPHICULUSPROJECT OPHICULUSPROJECT OPHICULUSPROJECT OPHICULUSPROJECT OPHICULUSPROJECT OPHICULUSPROJECT OPHICULUSPROJECT OPHICULUSPROJECT OPHICULUSPROJECT OPHICULUSPROJECT OPHICULUSPROJECT OPHICULUSPROJECT OPHICULUSPROJECT OPHICULUSPROJECT OPHICULUSPROJECT OPHICULUSPROJECT OPHICULUSPROJECT OPHICULUSPROJECT OPHICULUSPROJECT OPHICULUSPROJECT OPHICULUSPROJECT OPHICULUSPROJECT OPHICULUSPROJECT OPHICULUSPROJECT OPHICULUSPROJECT OPHICULUSPROJECT OPHICULUSPROJECT OPHICULUSPROJECT OPHICULUSPROJECT OPHICULUSPROJECT OPHICULUS", {
    pixelSize : 12,
    whiteColor : '#1D1E1F',
    blackColor :'#f5eddc',
    }
)

</script>

<template>
    <div id="con" class="xl:aspect-video h-full w-full max-w-[2160px] mx-auto flex flex-col md:flex-row items-center bg-[#202020] p-2 sm:border-0 md:border-0 justify-start">
        <div id="left" class="w-full h-2/5 xl:w-[50%] md:h-[80%] lg:h-[80%] flex flex-col xl:px-8 mb-2 lg:mb-0 overflow-hidden justify-center md:justify-start xl:justify-center">
            <div id="TODO" class="w-full py-2 xl:py-4 px-4 items-center xl:text-2xl text-xl flex text-green font-display font-bold bg-theme lg:text-3xl">
                PROJECT OPHICULUS [R]
            </div>
            <div id="status" class=" w-full flex justify-center font-display flex-col" >
                <div class="card-header font-display lg:text-2xl bg-orange px-4 mt-[2%]">
                    ▧ DECODE STATUS▸
                </div>
                <div id="details" class="grid grid-cols-5 mt-[1%] px-4 gap-1 sm:gap-2 text-sm sm:text-base">
                    <p class="bg-theme text-green px-1 col-span-2 select-none xl:flex hidden">▣ INDICES</p>
                        <p class="col-span-3 xl:flex hidden" v-if="file && transBlockIndices.length > 0">{{ transBlockIndices[transBlockIndices.length - 1] }}</p>
                        <p class="col-span-3 xl:flex hidden" v-else>[ ]</p>
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
                <div id="notrans" v-show="!isStartStreaming && !isMobile" class="hidden xl:grid xl:grid-cols-30 mt-[2%] px-2 border rounded-2xl text-center min-h-[150px] ">
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
            <!-- 相机录频区域 -->
            <div id="camera" class="w-full  md:max-w-[calc(min(75vmin,240px))] lg:max-w-[300px] xl:max-w-[340px] items-center flex justify-center">
              <!-- 视频元素 -->
              <div class="relative w-full aspect-square bg-black rounded-lg overflow-hidden">
                <video
                  ref="videoRef"
                  class="w-full h-full object-cover"
                  autoplay
                  muted
                  playsinline
                ></video>
                <!-- 隐藏的canvas用于二维码扫描 -->
                <canvas ref="canvasRef" class="hidden"></canvas>
                
                <!-- 扫描框UI -->
                <div v-if="isCameraActive" class="absolute inset-0 flex items-center justify-center pointer-events-none">
                  <div class="w-3/4 h-3/4 border-2 border-green-500 rounded-md flex items-center justify-center">
                    <div class="w-full h-full relative">
                      <!-- 扫描角标 -->
                      <div class="absolute top-0 left-0 w-4 h-4 border-t-2 border-l-2 border-green-500"></div>
                      <div class="absolute top-0 right-0 w-4 h-4 border-t-2 border-r-2 border-green-500"></div>
                      <div class="absolute bottom-0 left-0 w-4 h-4 border-b-2 border-l-2 border-green-500"></div>
                      <div class="absolute bottom-0 right-0 w-4 h-4 border-b-2 border-r-2 border-green-500"></div>
                    </div>
                  </div>
                </div>
                
                <!-- 摄像头未激活提示 -->
                <div v-else class="absolute inset-0 flex flex-col items-center justify-center text-gray-400">
                  <div class="text-4xl mb-2">📷</div>
                  <p class="text-sm">点击下方按钮启动摄像头</p>
                </div>
              </div>
            </div>
            <!-- 控制区域 - 音乐播放器风格 -->
            <div id="control" class="w-full max-w-[300px] ">
                <!-- 波形图 -->
                <div class="hidden xl:flex w-full h-24 rounded-t-xl  items-center px-4">
                  <div class="w-full flex items-center justify-between">
                    
                  </div>
                  <!-- 播放指示器 -->
                  <!-- <div class="w-1 h-8 bg-orange rounded-full -ml-1.5 shadow-lg" v-show="isEncoding"></div> -->
                </div>
                
                <!-- 按钮控制区域 -->
                <div class="w-full rounded-b-xl py-4 flex items-center justify-center space-x-5">
                  <!-- 左侧：发送端按钮 -->
                  <el-button 
                    @click="handleSwitchMode" 
                    type="primary"
                    size="medium"
                    :rounded="'rounded-full'"
                    class="flex items-center justify-center gap-2"
                  >
                    <span>⇄</span>
                    <span class="text-xs xl:flex hidden">SENDER</span>
                  </el-button>
                  
                  <!-- 右侧：摄像头控制按钮 -->
                  <el-button 
                    @click="isCameraActive ? stopCamera() : initCamera()" 
                    :type="isCameraActive ? 'danger' : 'success'"
                    size="medium"
                    :rounded="'rounded-full'"
                    class="flex items-center justify-center gap-2"
                  >
                    <span>{{ isCameraActive ? '🔴' : '🟢' }}</span>
                    <span class="text-xs xl:flex hidden">{{ isCameraActive ? '关闭摄像头' : '启动摄像头' }}</span>
                  </el-button>
                </div>
              </div>
            </div>
    </div>

</template>