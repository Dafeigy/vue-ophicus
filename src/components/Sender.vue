<script setup lang="ts">
// encode("123")
// ES6 import
import jsQR from "jsqr";
import {
  renderSVG,
} from 'uqr'


import { ref, onMounted, onUnmounted } from 'vue';
const file = ref(null);
const chunks = ref([]);
const transBlockIndices = ref([]);
const tranFPS = ref(0);
const bitRATE = ref(0);
// Use string type for seed to avoid BigInt serialization issues
  const seed = ref('42');
const isEncoding = ref(false);
const encodingInterval = ref(null);
const currentEncodedBlock = ref(null);

// 将svgg改为响应式变量，初始显示欢迎信息
const svgg = ref(renderSVG("PROJECT OPHICULUS ", {
    pixelSize : 12,
    whiteColor : '#1D1E1F',
    blackColor :'#f5eddc',
    }
));

// 导入Luby Transform相关模块
import init, { LubyTransformEncoder, EncodedBlock } from '../../wasm/luby_transform.js';
let encoder = null;
let isWasmInitialized = false;

// 初始化WASM模块
onMounted(async () => {
  try {
    await init();
    isWasmInitialized = true;
    console.log('Luby Transform WASM initialized successfully');
  } catch (error) {
    console.error('Failed to initialize WASM module:', error);
  }
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
  
  // 每隔30ms生成一个编码块并更新二维码
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
      const dataToEncode = encodedBlock.data.slice(0, 200); // 限制数据长度以确保能在二维码中显示
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
  svgg.value = renderSVG("PROJECT OPHICULUS ", {
    pixelSize: 12,
    whiteColor: '#1D1E1F',
    blackColor: '#f5eddc',
  });
  
  console.log('Encoding stopped and resources freed');
}

// 组件卸载时确保清理定时器
onUnmounted(() => {
  stopEncoding();
});


const handleFileClick = async() => {
  const input = document.createElement('input');
  input.type = 'file';
  input.onchange = async (event) => {
    file.value = (event.target as HTMLInputElement).files[0];
    const chunkSize = 1024 * 2; // 2KB   
    chunks.value = [];
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
    console.log('文件切片:', chunks.value); 
    
    // 初始化Luby Transform编码器
    initializeLubyTransformEncoder();
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
    
    // 生成一些编码块进行测试
    generateEncodedBlocks(5); // 生成5个编码块作为示例
  } catch (error) {
    console.error('Failed to initialize Luby Transform encoder:', error);
  }
}

// 生成编码块
function generateEncodedBlocks(count) {
  if (!encoder) {
    console.warn('Encoder not initialized');
    return;
  }
  
  const encodedBlocks = [];
  for (let i = 0; i < count; i++) {
    try {
      // 生成一个编码块
      const encodedBlock = encoder.generate_block();
      encodedBlocks.push({
        seed: encodedBlock.seed,
        degree: encodedBlock.degree,
        data: encodedBlock.data
      });
      
      console.log(`Generated encoded block ${i}:`, {
        seed: encodedBlock.seed.toString(),
        degree: encodedBlock.degree,
        dataLength: encodedBlock.data.length
      });
      
      // 注意：实际使用时应该在适当的时候调用encodedBlock.free()来释放内存
      // 这里为了简化示例，暂不处理
    } catch (error) {
      console.error(`Failed to generate encoded block ${i}:`, error);
    }
  }
  
  return encodedBlocks;
}


const handleFileSlice = () => {
  if (file.value) {
    const chunkSize = 1024 * 1024; // 1MB
    chunks.value = [];
    let start = 0;

    while (start < file.value.size) {
      const end = Math.min(start + chunkSize, file.value.size);
      chunks.value.push(file.value.slice(start, end));
      start = end;
    }

    console.log('文件切片:', chunks.value);
  }
};

</script>

<template>
    <div id="con" class="aspect-video flex sm:h-[96vh] lg:flex-row flex-col items-center bg-[#202020] p-2 sm:border-0  md:border-0 justify-center">
        <div id="left" class="lg:w-[40%]">
            <div id="TODO" class="w-full h-[20%] items-center sm:text-4xl text-3xl flex text-green text-left font-display px-2.5 font-bold bg-theme">
                PROJECT OPHICULUS
            </div>
        <div id="status" class=" w-full flex justify-center font-display flex-col " >
            <div class="card-header font-display sm:text-3xl bg-orange px-4 mt-[2%]">
                ▧ ENCODE STATUS▸
            </div>
            <div id="details" class="grid grid-cols-5 mt-[2%] px-4 gap-2 ">
                <p class="bg-green text-theme px-1 col-span-2 ">▣ FILENAME:</p> <p class="col-span-3 " v-if="file">{{ file.name }}</p><p class="col-span-3 " v-else> No File Selected yet...</p>
                <p class="bg-green text-theme px-1 col-span-2 ">▣ BYTES:</p><p class="col-span-3 " v-if="file" >{{ file.size }} BYTES</p><p class="col-span-3 " v-else>0 Bytes</p>
                <p class="bg-green text-theme px-1 col-span-2 ">▣ TOTAL:</p><p class="col-span-3 " v-if="file">{{ chunks.length }}</p><p class="col-span-3 " v-else>0 </p>
                <p class="bg-theme text-green px-1 col-span-2 select-none">▣ INDICES</p><p class="col-span-3 " v-if="file && transBlockIndices.length > 0">{{ transBlockIndices[transBlockIndices.length - 1] }}</p><p class="col-span-3 " v-else>[ ]</p>
                <p class="bg-theme text-green px-1 col-span-2 select-none">▣ BITRATE</p ><p class="col-span-3 " v-if="file">{{ bitRATE }} bit/s</p><p class="col-span-3 " v-else>0.0 bits/s</p>
                <p class="bg-theme text-green px-1 col-span-2 select-none">▣ FPS</p><p class="col-span-3 " v-if="file">{{ tranFPS}} </p><p class="col-span-3 " v-else>0</p>
            </div>
            
        </div>
        </div>
        <div id="right" class="lg:w-[50%] flex flex-col justify-center lg:mx-0 mt-5 items-center">
            <div id="img" class="w-[90%] lg:w-[50%] items-center flex mt-2.5 justify-center">
                <el-card style="width: 95%; height: 95%;">
                    <div v-html="svgg" class="qrcode"></div>
                </el-card>
            </div>
            <div id="control" class="w-[90%] lg:w-[50%] h-[5%] grid-cols-5 grid px-4 text-theme mt-2">
                <el-button type="success" color="#5c7f71" class="col-span-2" @click="handleFileSlice">RECEIVER</el-button>
                <template v-if="isEncoding">
                  <el-button type="danger" color="#f55e5e" class="col-span-1" @click="stopEncoding">Stop</el-button>
                </template>
                <template v-else>
                  <el-button type="success" color="#f5eddc" class="col-span-1" @click="startEncoding" :disabled="!encoder">Start</el-button>
                </template>
                <el-button type="warning" color="#ba8530" class="col-span-2"  @click="handleFileClick">SELECT FILE </el-button>
            </div>
        </div>
        
    </div>

</template>

<style>

</style>