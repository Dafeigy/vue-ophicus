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
const totalBlocks = ref(100)
const transBlockIndices = ref([]);
const currentReceiveBlocks = ref(64);
const isDecoding = ref(false);

// 摄像头相关变量
const isCameraActive = ref(false);
// 存储二维码扫描结果
const scanResult = ref('')


const test_toggle_decoding = ()=>{
  isDecoding.value = !isDecoding.value;
}
// 测试用变量
const scanSuccess = ref(false);


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
                <div class="font-display lg:text-2xl bg-orange px-4 mt-[2%]">
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
                <div class="font-display lg:text-2xl bg-orange px-4 mt-[2%] hidden xl:flex">
                    ▧ TRANS STATUS▸
                </div>
                <div id="notrans" class="flex mt-[2%] px-2 text-center min-h-[calc(50% - 10px)] sm:max-h-[100px] md:min-h-[150px] flex flex-col justify-start p-3">
                  <!-- 进度条项目 -->
                  <div class="mb-4">
                    <div class="flex justify-between items-center mb-1">
                      <span class="text-white font-bold">Trans progress</span>
                      <span class="text-white px-2 py-0.5 bg-[#333] rounded text-sm">
                        {{ Math.round((currentReceiveBlocks / totalBlocks) * 100) }}%
                      </span>
                    </div>
                    <div class="w-full h-4 bg-[#333] overflow-hidden">
                      <div 
                        class="h-full bg-green transition-all duration-300 ease-out"
                        :style="{ width: (currentReceiveBlocks / totalBlocks) * 100 + '%' }"
                      ></div>
                    </div>
                  </div>
                  <!-- 隐藏原有底部空白区域 -->
                  <div class="w-full text-sm text-gray-300 mt-2">
                     
                  </div>
                </div>
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

                  <button 
                      v-if="isDecoding"
                      @click="test_toggle_decoding" 
                      class="w-10 aspect-square rounded-full border-2 border-theme flex items-center justify-center hover:bg-[#343536] transition-all cursor-pointer"
                    >
                      ▧
                  </button>
                  <button 
                    v-else
                    @click="test_toggle_decoding" 
                    class="w-10 aspect-square rounded-full border-2 border-theme flex items-center justify-center hover:bg-[#343536] transition-all text-2xl cursor-pointer"
                  >
                   ▸ 
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