<script setup lang="ts">
import QrScanner from 'qr-scanner';
import { ref, onUnmounted, nextTick, watch, onMounted } from 'vue';

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
  qrCodeDetected: [result: string]; // 用于调试的二维码识别事件
}>();

// 响应式变量
const isCameraActive = ref(false);
const videoRef = ref<HTMLVideoElement | null>(null);
const canvasRef = ref<HTMLCanvasElement | null>(null);
const displayCanvasRef = ref<HTMLCanvasElement | null>(null);
const stream = ref<MediaStream | null>(null);
const scanInterval = ref<number | null>(null);
const lastScanResult = ref<string | null>(null);

// 组件样式
const containerStyle = ref({
  width: props.width
});

// 监听宽度变化
watch(() => props.width, (newWidth) => {
  containerStyle.value.width = newWidth;
});

// 组件挂载时的清理
onUnmounted(() => {
  stopCamera();
});

// 停止摄像头
const stopCamera = () => {
  // 清除扫描间隔
  if (scanInterval.value) {
    clearInterval(scanInterval.value);
    scanInterval.value = null;
  }
  
  // 停止视频流
  if (stream.value) {
    stream.value.getTracks().forEach(track => track.stop());
    stream.value = null;
  }
  
  // 清除视频源
  if (videoRef.value) {
    videoRef.value.srcObject = null;
  }
  
  isCameraActive.value = false;
  console.log('摄像头已关闭');
};

// 更新显示canvas，显示摄像头画面
  const updateDisplayCanvas = () => {
    // 使用querySelector作为备选方案
    let videoElement = videoRef.value || document.getElementById('cameraVideo') as HTMLVideoElement | null;
    let displayCanvasElement = displayCanvasRef.value || document.getElementById('displayCanvas') as HTMLCanvasElement | null;
    
    console.log('updateDisplayCanvas调用，检查元素:', {
      videoElementExists: !!videoElement,
      displayCanvasElementExists: !!displayCanvasElement
    });
    
    if (!videoElement || !displayCanvasElement) {
      console.error('updateDisplayCanvas: 缺少必要的DOM元素');
      return;
    }
    
    const displayCanvas = displayCanvasElement;
    const displayCtx = displayCanvas.getContext('2d');
    
    if (!displayCtx) return;
    
    // 确保canvas尺寸与视频元素匹配
    const updateCanvasSize = () => {
      if (!videoElement) return;
      
      const videoWidth = videoElement.videoWidth || 640;
      const videoHeight = videoElement.videoHeight || 480;
      
      // 只有当尺寸变化时才更新canvas尺寸
      if (displayCanvas.width !== videoWidth || displayCanvas.height !== videoHeight) {
        displayCanvas.width = videoWidth;
        displayCanvas.height = videoHeight;
        console.log(`Canvas尺寸更新为: ${videoWidth}x${videoHeight}`);
      }
    };
    
    // 初始更新尺寸
    updateCanvasSize();
    
    // 绘制视频帧到canvas
    const drawVideo = () => {
      if (!isCameraActive.value || !videoElement) return;
      
      try {
        // 更新尺寸以适应视频尺寸变化
        updateCanvasSize();
        
        // 绘制视频帧
        displayCtx.drawImage(videoElement, 0, 0, displayCanvas.width, displayCanvas.height);
        
        // 绘制扫描框（中间80%的区域）
        displayCtx.strokeStyle = 'rgba(0, 255, 0, 0.8)';
        displayCtx.lineWidth = 2;
        
        // 计算扫描框位置和大小（中间80%的区域）
        const scanBoxSize = Math.min(displayCanvas.width, displayCanvas.height) * 0.8;
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
      } catch (error) {
        console.error('绘制视频帧失败:', error);
      }
      
      // 继续下一帧
      if (isCameraActive.value) {
        requestAnimationFrame(drawVideo);
      }
    };
    
    // 启动动画循环
    console.log('开始渲染视频画面');
    requestAnimationFrame(drawVideo);
  };

// 扫描二维码
    const scanQRCode = async () => {
      // 使用querySelector作为备选方案
      let videoElement = videoRef.value || document.getElementById('cameraVideo') as HTMLVideoElement | null;
      let canvasElement = canvasRef.value || document.getElementById('scanCanvas') as HTMLCanvasElement | null;
      
      if (!videoElement || !canvasElement) return;
      
      const scanCanvas = canvasElement;
      const ctx = scanCanvas.getContext('2d', { willReadFrequently: true });
      
      if (!ctx) return;
      
      try {
        // 设置canvas尺寸
        const videoWidth = videoElement.videoWidth || 640;
        const videoHeight = videoElement.videoHeight || 480;
        
        scanCanvas.width = videoWidth;
        scanCanvas.height = videoHeight;
        
        // 绘制当前视频帧到canvas
        ctx.drawImage(videoElement, 0, 0, scanCanvas.width, scanCanvas.height);
        
        // 计算扫描框位置和大小（中间80%的区域）
        const scanBoxSize = Math.min(videoWidth, videoHeight) * 0.8;
        const scanBoxX = (videoWidth - scanBoxSize) / 2;
        const scanBoxY = (videoHeight - scanBoxSize) / 2;
        
        // 创建临时canvas用于裁剪中间80%的区域
        const tempCanvas = document.createElement('canvas');
        const tempCtx = tempCanvas.getContext('2d');
        
        if (tempCtx) {
          tempCanvas.width = scanBoxSize;
          tempCanvas.height = scanBoxSize;
          
          // 从原图裁剪中间80%的区域到临时canvas
          tempCtx.drawImage(
            scanCanvas,
            scanBoxX, scanBoxY, scanBoxSize, scanBoxSize, // 源区域
            0, 0, scanBoxSize, scanBoxSize // 目标区域
          );
          
          // 识别二维码
          const result = await QrScanner.scanImage(tempCanvas, {
            returnDetailedScanResult: true
          });
          
          if (result) {
            console.log('扫描到二维码:', result);
            
            // 避免重复触发相同的结果
            if (lastScanResult.value !== result.data) {
              lastScanResult.value = result.data;
              emit('qrScanned', result.data);
              emit('qrCodeDetected', result.data);
              
              // 直接更新debug-info div（作为备用方案）
              setTimeout(() => {
                const debugInfoDiv = document.getElementById('debug-info');
                if (debugInfoDiv) {
                  debugInfoDiv.textContent = `识别结果: ${result.data   }`;
                  console.log('已更新debug-info div:', result);
                }
              }, 0);
            }
          }
        }
      } catch (error) {
        // 静默失败，继续下一次扫描
        console.debug('扫描失败，继续下一次:', error);
      } finally {
        // 清理事件监听器
        if (videoElement) {
          videoElement.onerror = null;
        }
      }
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

// 初始化摄像头
const initCamera = async () => {
  try {
    console.log("初始化摄像头...");
    
    // 确保摄像头界面已渲染
    isCameraActive.value = true; // 先激活状态，确保DOM元素已渲染
    
    // 等待DOM完全挂载
    await nextTick();
    console.log('nextTick完成，检查ref绑定:', {
      videoRefExists: !!videoRef.value,
      canvasRefExists: !!canvasRef.value,
      displayCanvasRefExists: !!displayCanvasRef.value
    });
    
    // 使用querySelector作为备选方案获取video元素
    let videoElement = videoRef.value || document.getElementById('cameraVideo') as HTMLVideoElement | null;
    
    if (!videoElement) {
      console.error('错误: 无法获取video元素，ref绑定失败');
      isCameraActive.value = false; // 恢复状态
      emit('cameraError', '摄像头组件初始化失败');
      return;
    }
    
    // 临时保存有效的video元素引用供其他函数使用
    if (!videoRef.value && videoElement) {
      // 使用反射设置值，这是Vue 3中ref的内部机制
      (videoRef as any)._rawValue = videoElement;
      (videoRef as any).value = videoElement;
      console.log('已通过querySelector修复videoRef绑定');
    }
    
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

    // 显示视频流 - 使用我们已经验证的videoElement
    if (videoElement) {
      videoElement.srcObject = stream.value;
      
      // 监听视频加载完成事件
      videoElement.onloadeddata = () => {
        console.log('视频流加载完成，开始播放');
        videoElement?.play().catch(e => console.error('播放视频失败:', e));
        
        console.log('摄像头已成功启动');
        
        // 开始扫描二维码
        startQRCodeScanning();
      };
      
      // 监听视频播放事件
      videoElement.onplay = () => {
        console.log('视频开始播放');
      };
      
      // 监听错误事件
      videoElement.onerror = (e) => {
        console.error('视频播放错误:', e);
        emit('cameraError', '视频播放失败');
      };
    }
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

// 点击组件时切换摄像头状态
const toggleCamera = async () => {
  if (isCameraActive.value) {
    stopCamera();
  } else {
    // 确保DOM已完全挂载
    await nextTick();
    console.log('点击切换摄像头状态，检查ref:', {
      videoRefExists: !!videoRef.value
    });
    initCamera();
  }
};

// 暴露方法给父组件
defineExpose({
  toggleCamera,
  isCameraActive
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
      v-show="!isCameraActive" 
      class="absolute inset-0 flex flex-col items-center justify-center bg-theme text-white z-10"
    >
      <svg viewBox="0 0 256 256" xmlns="http://www.w3.org/2000/svg"><rect fill="none" height="256" width="256"/><rect fill="none" height="64" rx="8" stroke="#000" stroke-linecap="round" stroke-linejoin="round" stroke-width="16" width="64" x="48" y="48"/><rect fill="none" height="64" rx="8" stroke="#000" stroke-linecap="round" stroke-linejoin="round" stroke-width="16" width="64" x="48" y="144"/><rect fill="none" height="64" rx="8" stroke="#000" stroke-linecap="round" stroke-linejoin="round" stroke-width="16" width="64" x="144" y="48"/><line fill="none" stroke="#000" stroke-linecap="round" stroke-linejoin="round" stroke-width="16" x1="144" x2="144" y1="144" y2="176"/><polyline fill="none" points="144 208 176 208 176 144" stroke="#000" stroke-linecap="round" stroke-linejoin="round" stroke-width="16"/><line fill="none" stroke="#000" stroke-linecap="round" stroke-linejoin="round" stroke-width="16" x1="176" x2="208" y1="160" y2="160"/><line fill="none" stroke="#000" stroke-linecap="round" stroke-linejoin="round" stroke-width="16" x1="208" x2="208" y1="192" y2="208"/></svg>
    </div>

    <!-- 摄像头相关元素 - 始终渲染但条件显示 -->
    <div class="relative w-full h-full">
      <!-- 隐藏的video元素 -->
      <video
        id="cameraVideo"
        ref="videoRef"
        class="hidden absolute top-0 left-0 w-full h-full"
        autoplay
        muted
        playsinline
        preload="auto"
        tabindex="-1"
      ></video>
      
      <!-- 隐藏的canvas用于二维码扫描 -->
      <canvas 
        id="scanCanvas"
        ref="canvasRef" 
        class="hidden absolute top-0 left-0 w-full h-full"
      ></canvas>
      
      <!-- 可见的canvas用于显示摄像头画面 -->
      <canvas 
        id="displayCanvas"
        ref="displayCanvasRef" 
        class="absolute top-0 left-0 z-0" 
        :class="{ 'hidden': !isCameraActive, 'w-full h-full object-cover': isCameraActive }"
      ></canvas>
      
      <!-- 摄像头状态指示器 -->
       <!-- 计划做帧数显示 -->
      <!-- <div class="absolute top-2 right-2 flex items-center space-x-1 z-20">
        <div class="w-2 h-2 rounded-full bg-red-500 animate-pulse"></div>
        <span class="text-xs text-white font-medium">识别中</span>
      </div> -->
      
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
  object-fit: cover;
  image-rendering: -webkit-optimize-contrast;
  image-rendering: crisp-edges;
}

/* 容器样式优化 */
.relative {
  position: relative;
}

.aspect-square {
  aspect-ratio: 1 / 1;
}

/* 视频元素隐藏但保持功能 */
.hidden {
  opacity: 0;
  position: absolute;
  top: 0;
  left: 0;
  width: 100%;
  height: 100%;
  pointer-events: none;
}

/* 确保圆角正确应用 */
.overflow-hidden {
  overflow: hidden;
}
</style>