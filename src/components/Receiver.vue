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
const canvasRef = ref<HTMLCanvasElement | null>(null); // 用于扫描的canvas
const displayCanvasRef = ref<HTMLCanvasElement | null>(null); // 用于显示的canvas
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
    console.log("Scaning now...")
    scanQRCode();
  }, 50);
};

// 扫描二维码
const scanQRCode = () => {
  if (!videoRef.value || !canvasRef.value || !displayCanvasRef.value) return;
  
  const scanCanvas = canvasRef.value;
  const displayCanvas = displayCanvasRef.value;
  const scanCtx = scanCanvas.getContext('2d', {willReadFrequently: true});
  const displayCtx = displayCanvas.getContext('2d');
  
  if (!scanCtx || !displayCtx) return;
  
  // 设置canvas尺寸
  const videoWidth = videoRef.value.videoWidth || 640;
  const videoHeight = videoRef.value.videoHeight || 480;
  
  // 设置扫描canvas尺寸
  scanCanvas.width = videoWidth;
  scanCanvas.height = videoHeight;
  
  // 设置显示canvas尺寸
  displayCanvas.width = videoWidth;
  displayCanvas.height = videoHeight;
  
  // 绘制视频帧到两个canvas
  scanCtx.drawImage(videoRef.value, 0, 0, scanCanvas.width, scanCanvas.height);
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
  
  // 获取图像数据
  const imageData = scanCtx.getImageData(0, 0, scanCanvas.width, scanCanvas.height);
  
  // 使用jsQR库扫描二维码，尝试多种反转模式以提高识别率
  const code = jsQR(imageData.data, scanCanvas.width, scanCanvas.height, {
    inversionAttempts: 'both', // 尝试正常和反转模式
  });
  
  // 如果扫描到二维码
  if (code && code.data) {
    console.log('扫描到二维码:', code.data);
    lastScanResult.value = code.data;
    
    // 在显示canvas上标记二维码位置
    if (code.location) {
      displayCtx.strokeStyle = 'rgba(255, 255, 0, 0.8)';
      displayCtx.lineWidth = 3;
      displayCtx.beginPath();
      
      // 绘制二维码的四个角点连线
      displayCtx.moveTo(code.location.topLeftCorner.x, code.location.topLeftCorner.y);
      displayCtx.lineTo(code.location.topRightCorner.x, code.location.topRightCorner.y);
      displayCtx.lineTo(code.location.bottomRightCorner.x, code.location.bottomRightCorner.y);
      displayCtx.lineTo(code.location.bottomLeftCorner.x, code.location.bottomLeftCorner.y);
      displayCtx.closePath();
      displayCtx.stroke();
      
      // 显示扫描成功提示
      scanSuccess.value = true;
      setTimeout(() => {
        scanSuccess.value = false;
      }, 2000); // 2秒后隐藏提示
    }
  }
};

// 测试用变量
const scanSuccess = ref(false);
const testQRCodeContent = ref('Hello World!');
const generatedQRCode = ref('');
const directScanResult = ref('');
const directScanStatus = ref('');
const testCanvasRef = ref<HTMLCanvasElement | null>(null);

// 生成测试用二维码
const generateTestQRCode = () => {
  try {
    generatedQRCode.value = renderSVG(testQRCodeContent.value, {
      pixelSize: 8,
      whiteColor: '#555',
      blackColor: '#1D1E1F',
    });
    console.log('测试二维码已生成:', testQRCodeContent.value);
  } catch (error) {
    console.error('生成二维码失败:', error);
  }
};

// 直接从生成的二维码中识别内容
const directScanQRCode = () => {
  if (!testCanvasRef.value) {
    directScanStatus.value = '❌ Canvas元素不存在';
    return;
  }
  
  const canvas = testCanvasRef.value;
  const ctx = canvas.getContext('2d');
  
  if (!ctx) {
    directScanStatus.value = '❌ 无法获取Canvas上下文';
    return;
  }
  
  try {
    directScanStatus.value = '🔄 正在识别二维码...';
    
    // 创建一个临时的DOM元素来解析SVG
    const tempDiv = document.createElement('div');
    tempDiv.innerHTML = generatedQRCode.value;
    const svgElement = tempDiv.querySelector('svg');
    
    if (!svgElement) {
      directScanStatus.value = '❌ 无法获取SVG元素';
      return;
    }
    
    // 设置canvas尺寸与SVG一致
    const svgWidth = parseInt(svgElement.getAttribute('width') || '200');
    const svgHeight = parseInt(svgElement.getAttribute('height') || '200');
    canvas.width = svgWidth;
    canvas.height = svgHeight;
    
    // 清空canvas
    ctx.fillStyle = 'white';
    ctx.fillRect(0, 0, canvas.width, canvas.height);
    
    // 将SVG内容绘制到canvas上
    const image = new Image();
    const svgData = new XMLSerializer().serializeToString(svgElement);
    const svgBlob = new Blob([svgData], {type: 'image/svg+xml;charset=utf-8'});
    const url = URL.createObjectURL(svgBlob);
    
    image.onload = () => {
      try {
        // 绘制SVG到canvas
        ctx.drawImage(image, 0, 0);
        
        // 获取图像数据
        const imageData = ctx.getImageData(0, 0, canvas.width, canvas.height);
        
        // 使用jsQR识别二维码
        const code = jsQR(imageData.data, canvas.width, canvas.height, {
          inversionAttempts: 'both'
        });
        
        if (code && code.data) {
          directScanResult.value = code.data;
          directScanStatus.value = '✅ 识别成功！';
          scanSuccess.value = true;
          setTimeout(() => {
            scanSuccess.value = false;
          }, 2000);
        } else {
          directScanResult.value = '';
          directScanStatus.value = '❌ 未能识别出二维码';
        }
        
        // 清理
        URL.revokeObjectURL(url);
      } catch (error) {
        console.error('绘制或识别过程中出错:', error);
        directScanStatus.value = `❌ 识别出错: ${error.message}`;
        URL.revokeObjectURL(url);
      }
    };
    
    image.onerror = (err) => {
      console.error('图像加载失败:', err);
      directScanStatus.value = '❌ 图像加载失败';
      URL.revokeObjectURL(url);
    };
    
    image.src = url;
  } catch (error) {
    console.error('直接识别二维码失败:', error);
    directScanStatus.value = `❌ 识别失败: ${error.message}`;
  }
};

// 组件挂载后生成测试二维码并自动识别
nextTick(() => {
  generateTestQRCode();
  // 延迟一下让SVG完全生成
  setTimeout(() => {
    directScanQRCode();
  }, 100);
});

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
                <!-- 隐藏的video元素 -->
                <video
                  ref="videoRef"
                  class="hidden"
                  autoplay
                  muted
                  playsinline
                ></video>
                
                <!-- 隐藏的canvas用于二维码扫描 -->
                <canvas ref="canvasRef" class="hidden"></canvas>
                
                <!-- 可见的canvas用于显示摄像头画面 -->
                <canvas 
                  ref="displayCanvasRef" 
                  class="w-full h-full object-cover"
                  v-show="isCameraActive"
                ></canvas>
                
                <!-- 摄像头未激活提示 -->
                <!-- <div class="absolute inset-0 flex flex-col items-center justify-center text-gray-400">
                  <div class="text-4xl mb-2">📷</div>
                  <p class="text-sm">点击下方按钮启动摄像头</p>
                </div> -->
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
                
                <!-- 测试二维码区域 -->
                  <div class="w-full mt-4 p-2 bg-gray-800 rounded-lg">
                    <h4 class="text-xs text-gray-400 mb-2">测试二维码 (直接识别测试)</h4>
                    <div class="flex justify-center">
                      <div v-html="generatedQRCode" class="border-2 border-white p-4 bg-black aspect-square w-full"></div>
                      <!-- 隐藏的测试canvas -->
                      <canvas ref="testCanvasRef" class="hidden"></canvas>
                    </div>
                    <p class="text-center text-xs text-green mt-2">{{ testQRCodeContent }}</p>
                    
                    <!-- 直接识别按钮 -->
                    <div class="mt-2 flex justify-center">
                      <button 
                        @click="directScanQRCode()"
                        class="bg-blue-600 text-white text-xs px-3 py-1 rounded hover:bg-blue-700"
                      >
                        🔍 直接识别二维码
                      </button>
                    </div>
                    
                    <!-- 直接识别状态 -->
                    <div class="mt-2 text-xs p-2 rounded" :class="{
                      'bg-yellow-900': directScanStatus.includes('正在'),
                      'bg-green-700': directScanStatus.includes('成功'),
                      'bg-red-700': directScanStatus.includes('失败') || directScanStatus.includes('出错')
                    }">
                      {{ directScanStatus }}
                    </div>
                    
                    <!-- 直接识别结果 -->
                    <div v-if="directScanResult" class="mt-2 text-xs bg-gray-700 p-2 rounded">
                      <span class="text-yellow-400">识别结果:</span> {{ directScanResult }}
                    </div>
                    
                    <!-- 扫描成功提示 -->
                    <div v-if="scanSuccess" class="mt-2 text-xs bg-green-700 p-2 rounded text-white">
                      ✅ 二维码扫描成功！
                    </div>
                    
                    <!-- 对比结果 -->
                    <div v-if="directScanResult" class="mt-2 text-xs p-2 rounded" :class="{
                      'bg-green-800': directScanResult === testQRCodeContent,
                      'bg-red-800': directScanResult !== testQRCodeContent
                    }">
                      <span class="font-bold">验证:</span> {{ directScanResult === testQRCodeContent ? '内容匹配 ✓' : '内容不匹配 ✗' }}
                    </div>
                  </div>
                
                <!-- 按钮控制区域 -->
                <div class="w-full rounded-b-xl py-4 flex items-center justify-center space-x-5">
                  <!-- 左侧：发送端切换按钮 -->
                  <el-button 
                    @click="handleSwitchMode" 
                    type="primary"

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