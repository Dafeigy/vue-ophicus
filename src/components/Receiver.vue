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
const stream: Ref<MediaStream | null> = ref(null);
const scanInterval: Ref<number | null> = ref(null);
const lastScanResult = ref<string | null>(null);
const qrScanner: Ref<QrScanner | null> = ref(null);
// 存储二维码扫描结果
const scanResult = ref('')
// 处理二维码扫描事件
const handleQRScanned = (content) => {
  console.log('接收到扫描结果:', content)
  scanResult.value = content
}
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
  // 停止QR Scanner
  if (qrScanner.value) {
    qrScanner.value.stop();
    qrScanner.value.destroy();
    qrScanner.value = null;
  }
  
  // 清除扫描间隔
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
  // 停止之前的扫描器
  if (qrScanner.value) {
    qrScanner.value.stop();
    qrScanner.value.destroy();
    qrScanner.value = null;
  }
  
  // 清除之前的扫描间隔
  if (scanInterval.value) {
    clearInterval(scanInterval.value);
    scanInterval.value = null;
  }
  
  // 启动显示摄像头画面的更新
  updateDisplayCanvas();
  
  // 每50毫秒扫描一次
  scanInterval.value = window.setInterval(() => {
    console.log("Scaning now...");
    scanQRCode();
  }, 50);
};

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
    
    image.onload = async () => {
      try {
        // 绘制SVG到canvas
        ctx.drawImage(image, 0, 0);
        
        // 使用qr-scanner识别canvas中的二维码
        const result = await QrScanner.scanImage(canvas, {
          returnDetailedScanResult: true });
        
        if (result) {
          directScanResult.value = result.data;
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
        directScanStatus.value = `❌ 识别出错: ${error instanceof Error ? error.message : String(error)}`;
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
    directScanStatus.value = `❌ 识别失败: ${error instanceof Error ? error.message : String(error)}`;
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
                <div id="notrans" class="xl:grid xl:grid-cols-30 mt-[2%] px-2 border rounded-2xl text-center min-h-[calc(50% - 10px)] sm:max-h-[100px] md:min-h-[150px] flex items-center justify-center">
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