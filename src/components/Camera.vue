<script setup lang="ts">
import QrScanner from 'qr-scanner';
import { ref, onUnmounted, nextTick, watch } from 'vue';

// 组件属性定义
interface Props {
  width?: string; // 组件宽度，默认为父容器宽度的80%
}

const props = withDefaults(defineProps<Props>(), {
  width: '80%'
});

// 事件定义
const emit = defineEmits<{
  qrScanned: [content: string]; // 二维码识别成功事件
  cameraError: [error: string]; // 摄像头错误事件
}>();

// 响应式变量
const isCameraActive = ref(false);
const videoRef = ref<HTMLVideoElement | null>(null);
const canvasRef = ref<HTMLCanvasElement | null>(null);
const displayCanvasRef = ref<HTMLCanvasElement | null>(null);
const stream = ref<MediaStream | null>(null);
const scanInterval = ref<number | null>(null);
const lastScanResult = ref<string | null>(null);
const scanSuccess = ref(false);

// 组件样式
const containerStyle = ref({
  width: props.width
});

// 监听宽度变化
watch(() => props.width, (newWidth) => {
  containerStyle.value.width = newWidth;
});

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
      emit('cameraError', '您的浏览器不支持摄像头访问，请使用更新版本的浏览器');
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

    // 获取视频流，优先使用后置摄像头
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
    let errorMessage = '启动摄像头失败';
    
    if (error instanceof DOMException) {
      if (error.name === 'NotAllowedError') {
        errorMessage = '用户拒绝了摄像头访问权限';
      } else if (error.name === 'NotFoundError') {
        errorMessage = '未找到摄像头设备';
      } else if (error.name === 'NotReadableError') {
        errorMessage = '摄像头已被其他应用占用';
      } else {
        errorMessage = `摄像头错误: ${error.message}`;
      }
    }
    
    emit('cameraError', errorMessage);
    isCameraActive.value = false;
  }
};

// 停止摄像头
const stopCamera = () => {
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
  // 清除之前的扫描间隔
  if (scanInterval.value) {
    clearInterval(scanInterval.value);
    scanInterval.value = null;
  }
  
  // 启动显示摄像头画面的更新
  updateDisplayCanvas();
  
  // 每100毫秒扫描一次
  scanInterval.value = window.setInterval(() => {
    scanQRCode();
  }, 100);
};

// 扫描二维码
const scanQRCode = async () => {
  if (!videoRef.value || !canvasRef.value || !displayCanvasRef.value) return;
  
  const scanCanvas = canvasRef.value;
  const displayCanvas = displayCanvasRef.value;
  const ctx = scanCanvas.getContext('2d', { willReadFrequently: true });
  
  if (!ctx) return;
  
  // 设置canvas尺寸
  const videoWidth = videoRef.value.videoWidth || 640;
  const videoHeight = videoRef.value.videoHeight || 480;
  
  scanCanvas.width = videoWidth;
  scanCanvas.height = videoHeight;
  
  // 绘制当前视频帧到canvas
  ctx.drawImage(videoRef.value, 0, 0, scanCanvas.width, scanCanvas.height);
  
  // 计算扫描框位置和大小（中间70%的区域）
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
      
      // 避免重复触发相同的结果
      if (lastScanResult.value !== result.data) {
        lastScanResult.value = result.data;
        emit('qrScanned', result.data);
        
        // 显示扫描成功提示
        scanSuccess.value = true;
        setTimeout(() => {
          scanSuccess.value = false;
        }, 2000); // 2秒后隐藏提示
      }
      
      // 在显示canvas上标记二维码位置
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
      }
    }
  } catch (error) {
    // 解码错误，继续下一次扫描
  } finally {
    // 清理临时canvas
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
    
    // 计算扫描框位置和大小（中间70%的区域）
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

// 点击组件时切换摄像头状态
const toggleCamera = () => {
  if (isCameraActive.value) {
    stopCamera();
  } else {
    initCamera();
  }
};

// 暴露方法给父组件
const start = () => {
  if (!isCameraActive.value) {
    initCamera();
  }
};

const stop = () => {
  if (isCameraActive.value) {
    stopCamera();
  }
};

// 暴露方法给父组件
defineExpose({
  start,
  stop,
  isCameraActive
});

// 组件销毁时停止摄像头
onUnmounted(() => {
  stopCamera();
});
</script>

<template>
  <div 
    class="relative aspect-square bg-black rounded-2xl overflow-hidden cursor-pointer transition-all duration-300 hover:shadow-lg hover:shadow-green-500/20"
    :style="containerStyle"
    @click="toggleCamera"
  >
    <!-- 摄像头未激活时的提示 -->
    <div 
      v-if="!isCameraActive" 
      class="absolute inset-0 flex flex-col items-center justify-center bg-[#1e1d1f] text-white z-10"
    >
      <div class="text-4xl mb-2">📷</div>
      <div class="text-sm font-medium">点击开启摄像头</div>
      <div class="text-xs text-gray-300 mt-1">用于二维码识别</div>
    </div>

    <!-- 摄像头激活时的内容 -->
    <div v-else class="relative w-full h-full">
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
      ></canvas>
      
      <!-- 扫描成功提示 -->
      <div 
        v-if="scanSuccess"
        class="absolute top-2 left-1/2 transform -translate-x-1/2 bg-green-500 text-white px-3 py-1 rounded-full text-sm font-medium shadow-lg z-20 animate-pulse"
      >
        ✅ 识别成功
      </div>
      
      <!-- 摄像头状态指示器 -->
      <div class="absolute top-2 right-2 flex items-center space-x-1 z-20">
        <div class="w-2 h-2 rounded-full bg-red-500 animate-pulse"></div>
        <span class="text-xs text-white font-medium">录制中</span>
      </div>
      
      <!-- 操作提示 -->
      <div class="absolute bottom-2 left-1/2 transform -translate-x-1/2 text-xs text-white/70 bg-black/50 px-2 py-1 rounded">
        点击关闭摄像头
      </div>
    </div>
    
    <!-- 组件边框 -->
    <div class="absolute inset-0 border-2 border-gray-600/30 rounded-2xl pointer-events-none"></div>
  </div>
</template>

<style scoped>
/* 自定义动画 */
@keyframes pulse {
  0%, 100% { opacity: 1; }
  50% { opacity: 0.5; }
}

.animate-pulse {
  animation: pulse 2s cubic-bezier(0.4, 0, 0.6, 1) infinite;
}

/* 确保canvas元素正确显示 */
canvas {
  display: block;
  max-width: 100%;
  max-height: 100%;
}
</style>