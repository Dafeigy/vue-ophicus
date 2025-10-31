<script setup lang="ts">
// ES6 import
import jsQR from "jsqr";
import {
  renderSVG,
} from 'uqr'


import { ref, onMounted, onUnmounted, computed, inject } from 'vue';

// 从父组件App.vue注入切换模式函数
const handleSwitchMode = inject('handleSwitchMode') as (event: MouseEvent) => void;
const file = ref(null);
const chunks = ref([]);
const transBlockIndices = ref([]);
const tranFPS = ref(0);
const bitRATE = ref(0);
// Use string type for seed to avoid BigInt serialization issues
const seed = ref('42');
const isEncoding = ref(false);
const isFileChunked = ref(false);
const encodingInterval = ref(null);
const currentEncodedBlock = ref(null);
const blocks_counts = ref(0);
// Track current transmitting block indices
const currentTransmittingIndices = ref([]);
// Track all transmitted block indices (to maintain #b2653b background)
const transmittedIndices = ref([]);
// 响应式检测是否为移动设备
const isMobile = ref(window.innerWidth < 768); // 768px以下视为移动设备

const welcome = ref("Hello World!")

// 将svgg改为响应式变量，初始显示欢迎信息
const svgg = ref(renderSVG(welcome.value, {
    pixelSize : 12,
    whiteColor : '#1D1E1F',
    blackColor :'#f5eddc',
    }
));

// 导入Luby Transform相关模块
import init, { LubyTransformEncoder, LubyTransformDecoder } from '../../wasm/luby_transform.js';
import { encode } from "js-base64";
let encoder = null;

let isWasmInitialized = false;

// 窗口大小变化处理函数
const handleResize = () => {
  isMobile.value = window.innerWidth < 768;
};

// 初始化WASM模块和响应式布局
onMounted(async () => {
  try {
    await init();
    isWasmInitialized = true;
    console.log('Luby Transform WASM initialized successfully');
  } catch (error) {
    console.error('Failed to initialize WASM module:', error);
  }
  
  // 添加窗口大小变化监听
  window.addEventListener('resize', handleResize);
});

function blobToBase64(blob) {
    return new Promise((resolve, reject) => {
      const reader = new FileReader();
      reader.onload = () => {
        // reader.result 的格式是 "data:application/octet-stream;base64,AAAA..."
        // 我们通常只需要逗号后面的部分
        const dataUrl = reader.result;
        // 如果编码器需要纯Base64字符串（不包含Data URL前缀），则进行拆分
        const base64:string = (dataUrl as string).split(',')[1];
        resolve(base64);
      };
      reader.onerror = reject;
      reader.readAsDataURL(blob); // 这个方法会生成Data URL
    })
  }

// 开始编码并显示二维码
function startEncoding() {
  if (!encoder || isEncoding.value) {
    console.warn('Cannot start encoding: encoder not initialized or already encoding');
    return;
  }
  
  isEncoding.value = true;
  let frameCount = 0;
  const startTime = Date.now();
  
  // 每隔50ms生成一个编码块并更新二维码
  encodingInterval.value = setInterval(() => {
    try {
      // 生成编码块
      const encodedBlock = encoder.generate_block();
      currentEncodedBlock.value = encodedBlock;
      
      // 更新状态信息
      frameCount++;
      const elapsedTime = (Date.now() - startTime) / 1000; // 转换为秒
      tranFPS.value = Math.round(frameCount / elapsedTime); // 计算FPS
      
      // 将编码块的data用于生成二维码
      // 注意：由于二维码有数据量限制，可能需要对data进行适当处理
      const dataToEncode = encodedBlock.data // NOTE: 在这里确认要传输的数据
      svgg.value = renderSVG(dataToEncode, {
        pixelSize: 12,
        whiteColor: '#1D1E1F',
        blackColor: '#f5eddc',
      });
      
      // 计算比特率（假设每个字符是8比特）
      const bitsTransferred = frameCount * dataToEncode.length * 8;
      bitRATE.value = Math.round(bitsTransferred / elapsedTime);
      
      // 记录当前编码块的索引信息，将BigInt转换为字符串
      transBlockIndices.value.push({
        seed: encodedBlock.seed.toString(),
        degree: encodedBlock.degree
      });
      
      // Update current transmitting indices using the indices provided by the WASM module
      try {
        if (encodedBlock.indices && typeof encodedBlock.indices === 'object') {
          // Convert array-like object to array
          const indicesArray = [];
          
          // Extract values from the array-like object
          for (const key in encodedBlock.indices) {
            if (Object.prototype.hasOwnProperty.call(encodedBlock.indices, key) && !isNaN(parseInt(key))) {
              const idx = encodedBlock.indices[key];
              // Handle different possible index types
              const numericIdx = typeof idx === 'bigint' ? Number(idx) : Number.parseInt(idx.toString(), 10);
              indicesArray.push(numericIdx);
            }
          }
          
          currentTransmittingIndices.value = indicesArray;
          // Add all current indices to transmitted list
          indicesArray.forEach(index => {
            if (!transmittedIndices.value.includes(index)) {
              transmittedIndices.value.push(index);
            }
          });
          // console.log('Current transmitting block indices:', currentTransmittingIndices.value);
        } else {
          currentTransmittingIndices.value = [];
          console.log('No indices available for this encoded block');
        }
      } catch (e) {
        console.error('Error processing indices:', e);
        currentTransmittingIndices.value = [];
      }
      
    } catch (error) {
      console.error('Error generating encoded block:', error);
      stopEncoding();
    }
  },50); // 30ms间隔
}

// 停止编码
function stopEncoding() {
  if (encodingInterval.value) {
    clearInterval(encodingInterval.value);
    encodingInterval.value = null;
  }
  isEncoding.value = false;
  
  // Clear current transmitting indices but keep transmitted indices
  currentTransmittingIndices.value = [];
  
  // 释放当前编码块资源
  if (currentEncodedBlock.value) {
    try {
      currentEncodedBlock.value.free();
      currentEncodedBlock.value = null;
    } catch (error) {
      console.error('Error freeing encoded block:', error);
    }
  }
  
  // 重置FPS和比特率指标
  tranFPS.value = 0;
  bitRATE.value = 0;
  
  // 恢复显示初始欢迎信息二维码
  // svgg.value = renderSVG(welcome.value, {
  //   pixelSize: 12,
  //   whiteColor: '#1D1E1F',
  //   blackColor: '#f5eddc',
  // });
  
  console.log('Encoding stopped and resources freed');
}

// 组件卸载时确保清理所有资源
onUnmounted(() => {
  stopEncoding();
  // 移除窗口大小变化监听
  window.removeEventListener('resize', handleResize);
});


const handleFileClick = async() => {
  const input = document.createElement('input');
  input.type = 'file';
  input.onchange = async (event) => {
    file.value = (event.target as HTMLInputElement).files[0];
    const chunkSize = 1024 * 0.5; // 2KB   
    chunks.value = [];
    // Clear transmitting indices when new file is selected
    transmittedIndices.value = [];
    currentTransmittingIndices.value = [];
    transBlockIndices.value = [];
    console.log(file.value)
    
    // 1. 先将整个文件转换为Base64字符串
    const fullBase64 = await blobToBase64(file.value);
    
    // 2. 将Base64字符串转换为二进制数据
    const binaryString = atob(fullBase64 as string);
    const totalBytes = binaryString.length;
    
    // 3. 按块大小切分
    for (let i = 0; i < totalBytes; i += chunkSize) {
      // 获取当前块的二进制字符串
      let chunkBinary = binaryString.substring(i, Math.min(i + chunkSize, totalBytes));
      
      // 如果不足块大小，用空字符填充（对应0）
      if (chunkBinary.length < chunkSize) {
        chunkBinary = chunkBinary.padEnd(chunkSize, '\0');
      }
      
      // 将二进制字符串转换回Base64
      const base64String = btoa(chunkBinary);
      chunks.value.push(base64String);
    }

    console.log('文件切片数量:', chunks.value.length);
    console.log('每个切片长度:', chunks.value[0]?.length); // 所有切片长度相等
    console.log('首个文件切片:', chunks.value); 
    
    // 初始化Luby Transform编码器
    initializeLubyTransformEncoder();
    // 标记文件已切块
    isFileChunked.value = true;
  };
  input.click();
};

// 初始化Luby Transform编码器
function initializeLubyTransformEncoder() {
  if (!isWasmInitialized || chunks.value.length === 0) {
    console.warn('Cannot initialize encoder: WASM not initialized or no chunks available');
    return;
  }
  
  try {
    // 使用chunks作为源数据块初始化编码器，将字符串seed转换为BigInt
    encoder = new LubyTransformEncoder(chunks.value, BigInt(parseInt(seed.value)));
    console.log('Luby Transform encoder initialized with', encoder.source_block_count(), 'source blocks');
  } catch (error) {
    console.error('Failed to initialize Luby Transform encoder:', error);
  }
}


// test Area
// to fix
const debug_load = ref(false);
const dataToEncode  = ref("");
const test_generate_blocks = () => {
  chunks.value = [
    "6L+Z5piv5LiA5Liq5rWL6K+V5paH5qGj77yM5aSn5qaC5Y+q5pyJM0tCDQrov5nmmK/kuIDkuKrmtYvor5XmlofmoaPvvIzlpKfmpoLlj6rmnIkzS0INCui/meaYr+S4gOS4qua1i+ivleaWh+aho++8jOWkp+amguWPquaciTNLQg0K6L+Z5piv5LiA5Liq5rWL6K+V5paH5qGj77yM5aSn5qaC5Y+q5pyJM0tCDQrov5nmmK/kuIDkuKrmtYvor5XmlofmoaPvvIzlpKfmpoLlj6rmnIkzS0INCui/meaYr+S4gOS4qua1i+ivleaWh+aho++8jOWkp+amguWPquaciTNLQg0K6L+Z5piv5LiA5Liq5rWL6K+V5paH5qGj77yM5aSn5qaC5Y+q5pyJM0tCDQrov5nmmK/kuIDkuKrmtYvor5XmlofmoaPvvIzlpKfmpoLlj6rmnIkzS0INCui/meaYr+S4gOS4qua1i+ivleaWh+aho++8jOWkp+amguWPquaciTNLQg0K6L+Z5piv5LiA5Liq5rWL6K+V5paH5qGj77yM5aSn5qaC5Y+q5pyJM0tCDQrov5nmmK/kuIDkuKrmtYvor5XmlofmoaPvvIzlpKfmpoLlj6rmnIkzS0INCui/meaYr+S4gOS4qua1i+ivleaWh+aho++8jOU=",
    "pKfmpoLlj6rmnIkzS0INCui/meaYr+S4gOS4qua1i+ivleaWh+aho++8jOWkp+amguWPquaciTNLQg0K6L+Z5piv5LiA5Liq5rWL6K+V5paH5qGj77yM5aSn5qaC5Y+q5pyJM0tCDQrov5nmmK/kuIDkuKrmtYvor5XmlofmoaPvvIzlpKfmpoLlj6rmnIkzS0INCui/meaYr+S4gOS4qua1i+ivleaWh+aho++8jOWkp+amguWPquaciTNLQg0K6L+Z5piv5LiA5Liq5rWL6K+V5paH5qGj77yM5aSn5qaC5Y+q5pyJM0tCDQrov5nmmK/kuIDkuKrmtYvor5XmlofmoaPvvIzlpKfmpoLlj6rmnIkzS0INCui/meaYr+S4gOS4qua1i+ivleaWh+aho++8jOWkp+amguWPquaciTNLQg0K6L+Z5piv5LiA5Liq5rWL6K+V5paH5qGj77yM5aSn5qaC5Y+q5pyJM0tCDQrov5nmmK/kuIDkuKrmtYvor5XmlofmoaPvvIzlpKfmpoLlj6rmnIkzS0INCui/meaYr+S4gOS4qua1i+ivleaWh+aho++8jOWkp+amguWPquaciTNLQg0K6L+Z5piv5LiA5Liq5rWL6K+V5paH5qGj77yM5aSn5qaC5Y+q5pyJM0tCDQrov5nmmK/kuIDkuKo=",
    "5rWL6K+V5paH5qGj77yM5aSn5qaC5Y+q5pyJM0tCDQrov5nmmK/kuIDkuKrmtYvor5XmlofmoaPvvIzlpKfmpoLlj6rmnIkzS0INCui/meaYr+S4gOS4qua1i+ivleaWh+aho++8jOWkp+amguWPquaciTNLQg0K6L+Z5piv5LiA5Liq5rWL6K+V5paH5qGj77yM5aSn5qaC5Y+q5pyJM0tCDQrov5nmmK/kuIDkuKrmtYvor5XmlofmoaPvvIzlpKfmpoLlj6rmnIkzS0INCui/meaYr+S4gOS4qua1i+ivleaWh+aho++8jOWkp+amguWPquaciTNLQg0K6L+Z5piv5LiA5Liq5rWL6K+V5paH5qGj77yM5aSn5qaC5Y+q5pyJM0tCDQrov5nmmK/kuIDkuKrmtYvor5XmlofmoaPvvIzlpKfmpoLlj6rmnIkzS0INCui/meaYr+S4gOS4qua1i+ivleaWh+aho++8jOWkp+amguWPquaciTNLQg0K6L+Z5piv5LiA5Liq5rWL6K+V5paH5qGj77yM5aSn5qaC5Y+q5pyJM0tCDQrov5nmmK/kuIDkuKrmtYvor5XmlofmoaPvvIzlpKfmpoLlj6rmnIkzS0INCui/meaYr+S4gOS4qua1i+ivleaWh+aho++8jOWkp+amguWPquaciTM=",
    "S0INCui/meaYr+S4gOS4qua1i+ivleaWh+aho++8jOWkp+amguWPquaciTNLQg0K6L+Z5piv5LiA5Liq5rWL6K+V5paH5qGj77yM5aSn5qaC5Y+q5pyJM0tCDQrov5nmmK/kuIDkuKrmtYvor5XmlofmoaPvvIzlpKfmpoLlj6rmnIkzS0INCui/meaYr+S4gOS4qua1i+ivleaWh+aho++8jOWkp+amguWPquaciTNLQg0K6L+Z5piv5LiA5Liq5rWL6K+V5paH5qGj77yM5aSn5qaC5Y+q5pyJM0tCDQrov5nmmK/kuIDkuKrmtYvor5XmlofmoaPvvIzlpKfmpoLlj6rmnIkzS0INCui/meaYr+S4gOS4qua1i+ivleaWh+aho++8jOWkp+amguWPquaciTNLQg0K6L+Z5piv5LiA5Liq5rWL6K+V5paH5qGj77yM5aSn5qaC5Y+q5pyJM0tCDQrov5nmmK/kuIDkuKrmtYvor5XmlofmoaPvvIzlpKfmpoLlj6rmnIkzS0INCui/meaYr+S4gOS4qua1i+ivleaWh+aho++8jOWkp+amguWPquaciTNLQg0K6L+Z5piv5LiA5Liq5rWL6K+V5paH5qGj77yM5aSn5qaC5Y+q5pyJM0tCDQrov5nmmK/kuIDkuKrmtYvor5XmlofmoaM=",
    "77yM5aSn5qaC5Y+q5pyJM0tCDQrov5nmmK/kuIDkuKrmtYvor5XmlofmoaPvvIzlpKfmpoLlj6rmnIkzS0INCui/meaYr+S4gOS4qua1i+ivleaWh+aho++8jOWkp+amguWPquaciTNLQg0K6L+Z5piv5LiA5Liq5rWL6K+V5paH5qGj77yM5aSn5qaC5Y+q5pyJM0tCDQrov5nmmK/kuIDkuKrmtYvor5XmlofmoaPvvIzlpKfmpoLlj6rmnIkzS0INCui/meaYr+S4gOS4qua1i+ivleaWh+aho++8jOWkp+amguWPquaciTNLQg0K6L+Z5piv5LiA5Liq5rWL6K+V5paH5qGj77yM5aSn5qaC5Y+q5pyJM0tCDQrov5nmmK/kuIDkuKrmtYvor5XmlofmoaPvvIzlpKfmpoLlj6rmnIkzS0INCui/meaYr+S4gOS4qua1i+ivleaWh+aho++8jOWkp+amguWPquaciTNLQg0K6L+Z5piv5LiA5Liq5rWL6K+V5paH5qGj77yM5aSn5qaC5Y+q5pyJM0tCDQrov5nmmK/kuIDkuKrmtYvor5XmlofmoaPvvIzlpKfmpoLlj6rmnIkzS0INCgAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA="
];
  initializeLubyTransformEncoder();
  debug_load.value = true;
}

const test_render_next_frame = () => {
    if (!isWasmInitialized || chunks.value.length === 0) {
      console.warn('Cannot initialize encoder: WASM not initialized or no chunks available');
      return;
    }
    const encodedBlock = encoder.generate_block();
    currentEncodedBlock.value = encodedBlock;
    // const dataToEncode = encodedBlock.data // 限制数据长度以确保能在二维码中显示
    dataToEncode.value = JSON.stringify({
      filename: "text.txt",
      degree: encodedBlock.degree,
      indices:encodedBlock.indices,
      data:encodedBlock.data,
      seed: parseInt(encodedBlock.seed)
    })
    svgg.value = renderSVG(dataToEncode.value, {
      pixelSize: 12,
      whiteColor: '#1D1E1F',
      blackColor: '#f5eddc',
    });
    transBlockIndices.value.push({
        // seed: encodedBlock.seed.toString(),
        // degree: encodedBlock.degree,
        indices: encodedBlock.indices.map(index => index.toString())
    });
    // console.log(transBlockIndices.value)
}
const decode_counts = ref(0);
// 创建响应式解码器实例，使其在组件生命周期内保持状态
const decoderInstance = ref(null);
// 解码状态响应式变量
const decodeStatus = ref('');
const decodedBlocks = ref([]);

const test_decode_process = () => {
  if(chunks.value.length === 0 || !isWasmInitialized ){
    console.warn('Cannot initialize decoder: WASM not initialized or no data available.');
    decodeStatus.value = '错误：WASM未初始化或无数据';
    return;
  }
  
  // 如果解码器实例不存在，则创建新实例
  if (!decoderInstance.value) {
    decoderInstance.value = new LubyTransformDecoder(chunks.value.length, chunks.value[0].length);
    console.log(`解码器初始化：总块数=${chunks.value.length}，块大小=${chunks.value[0].length}`);
    decodeStatus.value = '解码器已初始化';
  }
  
  // 检查是否有可用的编码块数据
  if (!dataToEncode.value) {
    console.warn('No encoded block data available. Please click "Next" first.');
    decodeStatus.value = '错误：无编码块数据，请先点击Next';
    return;
  }
  
  try {
    // 解析编码块数据
    let received_data_json = JSON.parse(dataToEncode.value);
    console.log(`添加编码块 #${decode_counts.value + 1}`, {
      seed: received_data_json.seed,
      degree: received_data_json.degree,
      dataLength: received_data_json.data.length
    });
    
    // 将编码块添加到解码器
    decoderInstance.value.add_encoded_block(
      BigInt(received_data_json.seed), 
      received_data_json.degree, 
      received_data_json.data
    );
    
    // 增加解码计数
    decode_counts.value++;
    
    // 检查解码是否完成
    if (!decoderInstance.value.is_complete()) {
      const decodedCount = decoderInstance.value.decoded_count();
      const progress = Math.round((decodedCount / chunks.value.length) * 100);
      decodeStatus.value = `解码中：已解码 ${decodedCount}/${chunks.value.length} 块 (${progress}%)`;
      console.log(`解码进度：${decodedCount}/${chunks.value.length} 块 (${progress}%)`);
    } else {
      // 解码完成，获取并验证结果
      let results = decoderInstance.value.get_all_decoded_blocks();
      decodedBlocks.value = results;
      
      // 验证解码结果
      let isAllCorrect = true;
      for (let i = 0; i < results.length && i < chunks.value.length; i++) {
        if (results[i] !== chunks.value[i]) {
          isAllCorrect = false;
          console.error(`解码块 ${i} 验证失败`);
          break;
        }
      }
      
      decodeStatus.value = isAllCorrect 
        ? `解码成功！共使用 ${decode_counts.value} 个编码块`
        : `解码完成，但部分数据验证失败`;
      
      console.log("解码完成！");
      console.log(`总使用 ${decode_counts.value} 个编码块`);
      console.log(`解码结果验证：${isAllCorrect ? '全部正确' : '存在错误'}`);
      
      // 如果需要，可以在这里重置解码器和计数
      // resetDecoder();
    }
  } catch (error) {
    console.error('解码过程中发生错误:', error);
    decodeStatus.value = `解码错误: ${error.message}`;
  }
};

// 重置解码器函数
function resetDecoder() {
  if (decoderInstance.value) {
    try {
      // 释放解码器资源（如果WASM提供了相应方法）
      if (decoderInstance.value.free) {
        decoderInstance.value.free();
      }
    } catch (error) {
      console.error('释放解码器资源时出错:', error);
    }
    decoderInstance.value = null;
  }
  decode_counts.value = 0;
  decodedBlocks.value = [];
  decodeStatus.value = '';
  console.log('解码器已重置');
}




</script>

<template>
    <div id="con" class="text-theme xl:aspect-video h-full w-full max-w-[2160px] mx-auto flex flex-col md:flex-row items-center bg-[#202020] p-2 sm:border-0 md:border-0 justify-start">
        <div id="left" class="w-full h-2/5 xl:w-[50%] md:h-[80%] lg:h-[80%] flex flex-col xl:px-8 mb-2 lg:mb-0 overflow-hidden justify-center md:justify-start xl:justify-center">
            <div id="TODO" class="w-full py-2 xl:py-4 px-4 items-center xl:text-2xl text-xl flex text-green font-display font-bold bg-theme lg:text-3xl">
                PROJECT OPHICULUS [T]
            </div>
            <div id="status" class=" w-full flex justify-center font-display flex-col" >
                <div class="card-header font-display lg:text-2xl bg-orange px-4 mt-[2%] ">
                    ▧ ENCODE STATUS▸
                </div>
                <div id="details" class="grid grid-cols-5 mt-[1%] px-4 gap-1 sm:gap-2 text-sm sm:text-base">
                    <p class="bg-green text-theme px-1 col-span-2 ">▣ FILENAME:</p> <p class="col-span-2 sm:col-span-2 truncate overflow-hidden whitespace-nowrap" v-if="file">{{ file.name }}</p><p class="col-span-3 " v-else> No File Selected yet...</p>
                    <p class="bg-green text-theme px-1 col-span-2 ">▣ BYTES:</p><p class="col-span-3 " v-if="file" >{{ file.size }} BYTES</p><p class="col-span-3 " v-else>0 Bytes</p>
                    <p class="bg-green text-theme px-1 col-span-2 ">▣ TOTAL:</p><p class="col-span-3 " v-if="file">{{ chunks.length }}</p><p class="col-span-3 " v-else>0 </p>
                    <p class="bg-theme text-green px-1 col-span-2 select-none xl:flex hidden">▣ INDICES</p><p class="col-span-3 xl:flex hidden" v-if="(file && transBlockIndices.length > 0) || (debug_load && transBlockIndices.length > 0)">{{ transBlockIndices[transBlockIndices.length - 1] }}</p><p class="col-span-3 xl:flex hidden" v-else>[ ]</p>
                    <p class="bg-theme text-green px-1 col-span-2 select-none">▣ BITRATE</p ><p class="col-span-3 " v-if="file">{{ bitRATE }} bit/s</p><p class="col-span-3 " v-else>0.0 bits/s</p>
                    <p class="bg-theme text-green px-1 col-span-2 select-none">▣ FPS</p><p class="col-span-3 " v-if="file">{{ tranFPS}} </p><p class="col-span-3 " v-else>0</p>
                </div>
                <div class="card-header font-display lg:text-2xl bg-orange pl-4 mt-[2%] hidden xl:flex w-full">
                    <p class="w-1/5">▧ BLOCKS STATUS▸</p>
                    <div id="test_module" class="w-4/5  flex justify-between items-center h-full">
                      <div class="w-full flex justify-end items-center h-full">
                        <div id="test-file-load" @click="test_generate_blocks" class="select-none cursor-pointer h-full text-sm w-1/8 text-theme justify-center items-center flex hover:bg-theme transition-all duration-250 hover:text-orange">Load</div>
                        <div id="test-file-encode" @click="test_render_next_frame" class="select-none cursor-pointer h-full text-sm w-1/8 text-theme justify-center items-center flex hover:bg-theme transition-all duration-250 hover:text-orange">Next</div>
                        <div id="test-file-encode" @click="test_decode_process" class="select-none cursor-pointer h-full text-sm w-1/8 text-theme justify-center items-center flex hover:bg-theme transition-all duration-250 hover:text-orange">Decode</div>
                        <div @click="resetDecoder" class="select-none cursor-pointer h-full text-sm w-1/8 text-theme justify-center items-center flex hover:bg-theme transition-all duration-250 hover:text-orange">Reset</div>
                      </div>
                    </div>
                    
                </div>
                <div id="notrans" v-show="!isFileChunked && !isMobile" class="hidden xl:grid xl:grid-cols-30 mt-[2%] px-2 text-center min-h-[150px]">
                  <div v-if="debug_load" class="col-span-30 flex items-center justify-center text-orange text-xl animate-blink select-none">Loaded.</div>
                  <div v-else class="col-span-30 flex items-center justify-center text-green text-xl animate-blink select-none">WAITING FOR FILE BLOCKS ... ...</div>
                  <div v-if="decodeStatus" class="mb-1 col-span-30" :class="{
                          'text-theme': decodeStatus.includes('成功') || decodeStatus.includes('解码中'),
                          'text-orange': decodeStatus.includes('错误') || decodeStatus.includes('失败')
                        }">
                          {{ decodeStatus }}
                  </div>

                </div>
                <div id="transblocks" v-show="isFileChunked && !isMobile" class="hidden xl:grid xl:grid-cols-30 mt-[2%] px-2 overflow-y-auto" style="max-height: 150px; scrollbar-color: transparent transparent; overflow-x: hidden;">
                  <div v-for="_ in chunks.length" :key="_" 
                      class="bg-[#343536] text-theme text-[1vmin] font-sheikah m-1 flex aspect-square rounded justify-center items-center transition-all duration-80 ease-in-out"
                      :class="{
                        transactive: currentTransmittingIndices.includes(_ - 1),
                        'transmitted': transmittedIndices.includes(_ - 1)
                      }">
                    {{ _ }}
                  </div>
                </div>
              </div>
        </div>
        <div id="right" class="w-4/5 h-2/5 xl:w-[50%] md:h-[80%] lg:h-[80%] flex flex-col lg:mx-0 items-center px-2 md:justify-center">
            <div id="img" class="w-full md:max-w-[calc(min(75vmin,240px))] lg:max-w-[300px] xl:max-w-[340px] items-center flex justify-center">
                  <div id="qrcontainer" class="w-full aspect-square flex justify-center">
                    <div v-html="svgg" class="qrcode w-full"></div>
                  </div>
            </div>
            <!-- 控制区域 - 音乐播放器风格 -->
            <div id="control" class="w-full max-w-[300px] ">
                <!-- 波形图 -->
                <div class="hidden xl:flex w-full h-24 rounded-t-xl  items-center px-4">
                  <div class="w-full flex items-center justify-between">
                    <!-- 波形条 -->
                    <div v-for="i in 60" :key="i" 
                         class="h-6 w-0.5 bg-theme rounded-full"
                         :style="{ 
                           height: isEncoding ? `${Math.random() * 20 + 8}px` : '2px',
                           opacity: isEncoding ? 1 : 0.7
                         }">
                    </div>
                  </div>
                  <!-- 播放指示器 -->
                  <div class="w-1 h-8 bg-orange rounded-full -ml-1.5 shadow-lg" v-show="isEncoding"></div>
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
                  
                  <!-- 中间：开始/暂停按钮 -->
                  <button 
                    v-if="isEncoding"
                    @click="stopEncoding" 
                    class="w-10 aspect-square rounded-full border-2 border-theme flex items-center justify-center hover:bg-[#343536] transition-all cursor-pointer"
                  >
                    ▧
                  </button>
                  <button 
                    v-else
                    @click="startEncoding" 
                    class="w-10 aspect-square rounded-full border-2 border-theme flex items-center justify-center hover:bg-[#343536] transition-all text-2xl cursor-pointer"
                    :disabled="!encoder"
                    :class="{ 'opacity-50 cursor-not-allowed': !encoder }"
                  >
                  ▸
                    <!-- <div class="w-0 h-0 border-t-5 border-t-transparent border-b-5 border-b-transparent border-r-8 border-r-theme ml-1"></div> -->
                  </button>
                  
                  <!-- 右侧：选择文件按钮 -->
                  <button 
                    @click="handleFileClick" 
                    class="flex items-center justify-center gap-2 px-4 py-2 border border-theme rounded-full hover:bg-[#343536] transition-all cursor-pointer"
                  >
                    <span>📁</span>
                    <span class="text-xs xl:flex hidden">SELECT</span>
                  </button>
                </div>
              </div>
            </div>
    </div>
</template>

<style>
/* 闪烁动画效果 */
@keyframes blink {
  0%, 50%, 100% {
    opacity: 1;
  }
  25%, 75% {
    opacity: 0.3;
  }
}

.animate-blink {
  animation: blink 2s infinite;
}


.transmitted{
  background-color: #f5eddc;
  color: #333333;
}

.transactive{
  background-color: #5c7f71 !important;
  color: #f5eddc;
  scale: 1.2;
  /* 动画时间已在基础样式中设置 */
}
</style>