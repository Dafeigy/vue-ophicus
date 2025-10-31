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
const CHUNK_SIZE = 1024 * 2;
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
      
      // 按照要求组织数据格式：seed(4字节) + degree(4字节) + filename(32字节) + encodedBlock.data(2048字节)
      const bufferSize = 4 + 4 + 32 + 2048; // 总大小为2088字节
      const buffer = new ArrayBuffer(bufferSize);
      const byteView = new Uint8Array(buffer);
      
      // 1. 写入seed（4字节，小端序）
      const seedValue = parseInt(encodedBlock.seed);
      byteView[0] = seedValue & 0xFF;
      byteView[1] = (seedValue >> 8) & 0xFF;
      byteView[2] = (seedValue >> 16) & 0xFF;
      byteView[3] = (seedValue >> 24) & 0xFF;
      
      // 2. 写入degree（4字节，小端序）
      byteView[4] = encodedBlock.degree & 0xFF;
      byteView[5] = (encodedBlock.degree >> 8) & 0xFF;
      byteView[6] = (encodedBlock.degree >> 16) & 0xFF;
      byteView[7] = (encodedBlock.degree >> 24) & 0xFF;
      
      // 3. 写入文件名（从第8字节开始，固定32字节）
      const filename = "text.txt";
      for (let i = 0; i < 32; i++) {
        if (i < filename.length) {
          byteView[i + 8] = filename.charCodeAt(i);
        } else {
          byteView[i + 8] = 0; // 剩余空间用0填充
        }
      }
      
      // 4. 写入encodedBlock.data（从第40字节开始，共2048字节）
      const dataArray = stringToReadOnlyByteArray(encodedBlock.data);
      for (let i = 0; i < 2048; i++) {
        if (i < dataArray.length) {
          byteView[i + 40] = dataArray[i];
        } else {
          byteView[i + 40] = 0; // 不足部分用0填充
        }
      }
      
      // 将 Uint8Array 转换为只读的 number[] 数组
      const readonlyByteArray = Object.freeze(Array.from(byteView));
      
      // 使用组织好的数据生成二维码
      svgg.value = renderSVG(readonlyByteArray, {
        pixelSize: 12,
        whiteColor: '#1D1E1F',
        blackColor: '#f5eddc',
      });
      
      // 计算比特率（假设每个字符是8比特）
      const bitsTransferred = frameCount * dataToEncode.value.length * 8;
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
    chunks.value = [];
    // Clear transmitting indices when new file is selected
    transmittedIndices.value = [];
    currentTransmittingIndices.value = [];
    transBlockIndices.value = [];
    console.log(file.value)
    
    // 直接以字节形式读取文件
    const fileData = await readFileAsByteArray(file.value);
    const totalBytes = fileData.length;
    
    
    // 按块大小切分，使用CHUNK_SIZE常量确保每块大小相等
    for (let i = 0; i < totalBytes; i += CHUNK_SIZE) {
      // 创建一个新的数组用于存储当前块的字节
      const chunk = new Array(CHUNK_SIZE).fill(0);
      
      // 复制实际文件数据到块中
      const chunkLength = Math.min(CHUNK_SIZE, totalBytes - i);
      for (let j = 0; j < chunkLength; j++) {
        chunk[j] = fileData[i + j];
      }
      
      // 对于不足CHUNK_SIZE的部分，已用0填充
      chunks.value.push(chunk);
    }

    console.log('文件切片数量:', chunks.value.length);
    console.log('每个切片长度:', CHUNK_SIZE); // 所有切片长度为CHUNK_SIZE
    console.log('首个文件切片字节数组前10个字节:', chunks.value[0]?.slice(0, 10)); 
    
    // 初始化Luby Transform编码器
    initializeLubyTransformEncoder();
    // 标记文件已切块
    isFileChunked.value = true;
  };
  input.click();
};

// 将文件读取为字节数组的辅助函数
const readFileAsByteArray = (file: File): Promise<Uint8Array> => {
  return new Promise((resolve, reject) => {
    const reader = new FileReader();
    reader.onload = () => {
      if (reader.result instanceof ArrayBuffer) {
        resolve(new Uint8Array(reader.result));
      } else {
        reject(new Error('Failed to read file as ArrayBuffer'));
      }
    };
    reader.onerror = reject;
    reader.readAsArrayBuffer(file);
  });
};

// 初始化Luby Transform编码器
function initializeLubyTransformEncoder() {
  if (!isWasmInitialized || chunks.value.length === 0) {
    console.warn('Cannot initialize encoder: WASM not initialized or no chunks available');
    return;
  }
  
  try {
    // 将字节数组转换为Luby Transform编码器可接受的格式
    // 这里假设LubyTransformEncoder可以接受字节数组形式的块数据
    const sourceBlocks = chunks.value.map(chunk => {
      // 如果需要字符串形式，可以将字节数组转换为字符串
      // 否则直接传递字节数组
      
      return String.fromCharCode(...chunk);
    });
    console.log("Before Mapping: ", chunks.value)
    console.log("After mapping:", sourceBlocks)
    // 使用转换后的源数据块初始化编码器，将字符串seed转换为BigInt
    encoder = new LubyTransformEncoder(sourceBlocks, BigInt(parseInt(seed.value)));
    console.log(chunks.value)
    console.log('Luby Transform encoder initialized with', encoder.source_block_count(), 'source blocks');
    console.log('每个源块大小:', CHUNK_SIZE, '字节');
  } catch (error) {
    console.error('Failed to initialize Luby Transform encoder:', error);
  }
}


// test Area

const debug_load = ref(false);
const dataToEncode  = ref("");
const test_generate_blocks = () => {
  chunks.value = [
    "## Rachelå¤å»è®°å½\r\n\r\n![Rachel](https://pic1.imgdb.cn/item/68d3a332c5157e1a882cad57.png)\r\n\r\nGithub é¾æ¥ï¼https://github.com/Forairaaaaa/Rachel\r\n\r\nåçç¡¬ä»¶é¾æ¥ï¼https://oshwhub.com/eedadada/mason\r\n\r\næ¹çç¡¬ä»¶é¾æ¥ï¼https://oshwhub.com/vaan/rachel-kai-yuan-mi-ni-you-xi-ji\r\n\r\nBilibiliæ¼ç¤ºè§é¢ï¼https://www.bilibili.com/video/BV1Ga4y1f7d3\r\n\r\n## åæåå¤\r\n\r\nåæåå¤éè¦åå¤å¥½å·¥å·ãPCBçµè·¯æ¿ãçµå­åå¨ä»¶ã\r\n\r\n### å·¥æ¬²åå¶äºå¿åå©å¶å¨\r\n\r\nå¨ä¸åå¼å§åï¼ä¸å®è¦å¤å¥½ææçå·¥å·ï¼åæ¬ï¼\r\n\r\n- è¶æçéæé©¬è¹å¤´çµçéï¼ç¨æ¥åç»­å¤çåä»¶çæ¥ã\r\n- å ç­å°ï¼ç¨æ¥éæ¿ç§å¤§é¢ç§¯çè´´çåä»¶ã\r\n- ç­é£æªï¼å¦æä½ ä½¿ç¨çæ¹çç¡¬ä»¶ï¼é£ä¹é£é¢ESP32-1uçæ¨¡åä½ åªè½éè¿ç­é£æªå»çæ¥ã\r\n- çé¡ä¸ï¼ææ¨èä½¿ç¨ä»·æ ¼ç¨å¾®é«ç¹çï¼å°½éä¸è¦ç¨ä¹°çééçï¼éçä¸è¬é½å¾é¾çå¥½çï¼å¯¹çæ¥è´¨éæ²¡æè¦æ±çå¯ä»¥å¿½ç¥ï¼\r\n- é¡èï¼æä½¿ç¨çæ¯ä¸­ä½æ¸©çé¡èï¼ç¨äºéæ¿ç§ã\r\n- å©çåï¼ç¥å¨ï¼ç¨æ¥å¤çè´´çåä»¶è¿é¡çæåµç¹å«æç¨ï¼æ¯å¦è¯´Typecå£ççæ¥ã\r\n- ä¸ç¨è¡¨ï¼ç¨æ¥debugï¼æ£æ¥çµè·¯çæ¥ã\r\n\r\n### ä¸å ä¹±ä¸å«ç³çåä»¶\r\n\r\nè´­ä¹°å¾çæ¥çåä»¶æ¯ä¸ä¸ªéè¦æè¡¡æ¶é´åè´¢åçtradeoffè¿ç¨ãä½ å¯ä»¥éæ©å¨åç«åä¸é®ä¸åï¼ä¹å¯ä»¥éæ©èªå·±ä»æ·å®éé¢å»ä¸ä¸ªä¸ªæ¾ãåèæ¹ä¾¿æ èï¼ä½ä»·æ ¼å¯è½ä¼ç¥å¾®ä¸åä¸ç¹ï¼åèåæ¯éè¦ä½ è´¹ç¥å°å¤æç½ãæ¯å¯¹å¨ä»¶åå·ãæçåæ³æ¯ï¼è¯çãå¤§ä»¶ä¸ç¹çåå¨ä»¶å°±èªå·±ä¹°ï¼æ¯å¦è¯´ä¸»æ§è¯çãå±å¹ç­ï¼ãè¿éææ¨èå»ä¼ä¿¡çµå­å»ä¹°å±å¹ï¼1.3å¯¸TFT 240x240ï¼ï¼BMI270çéèºä»ªè¯çä¹å¯ä»¥å¨ä»è¿ä¹°ï¼ä½ä¼ç¨å¾®è´µä¸ç¹ï¼æä¹°çæ¶åæ¯2.7ä¸é¢ï¼ä½æçå°æå¶ä»åº1.xå°±æï¼ã\r\n\r\n### çµè·¯æ¿ï¼çµè·¯æ¿ï¼çµè·¯æ¿ï¼\r\n\r\nPCBå¶é ååéè¦ä¸ä¸ªGerber.zipçå¶æ¿æä»¶æ¥å¶ä½PCBæ¿ãä½ å¯ä»¥éæ©ä½ åæ¬¢çPCBå¶é åï¼æç°å¨ç¨è¿ä¸¤å®¶ï¼\r\n\r\n- åç«åï¼",
    "ä»·æ ¼ç¥å¾®è´µç¹ï¼ä½å¶æ¿è´¨éè¿å¯ä»¥ãéè¦æ³¨æçæ¯å¦æä½ å¯¹PCBçå¤è§æ¡ä»¶æè¦æ±ï¼ä¸å¸ææ·»å çäº§ç¼å·ãæèäº§åä¿¡æ¯çäºç»´ç æ¶\r\n- æ·éï¼åè´¹ææ ·å¾é¦ï¼ä½è¿ä¸ªé¡¹ç®ç¨çåå±æ¿è¦ä¼ä¸è®¤è¯ç¨æ·æå¯ä»¥åè´¹ææ ·ãä½æ¯å³ä¾¿å¦æ­¤ï¼ä¸ä½¿ç¨åè´¹ææ ·å¹¶éæ©ææ ·3çæ¶ï¼å®ä¹ä¼å¤§æ¦çç»ä½ å5çè¿æ¥ï¼å°èµä¹æ¯èµ¢ï¼\r\n\r\næä½¿ç¨äºESP321uæ¨¡ç»å»åï¼è¿ä¸ªæ¨¡ç»æ¯åä½èçé£ä¸ªè¦ä¾¿å®å¾å¤ï¼å·®ä¸å¤9åé±å§ï¼åçé£ä¸ªstackæ¨¡ç»è¦70+äºç°å¨ãä½ å¯ä»¥ç´æ¥æ¿æçGerberæä»¶å»ææ¿ï¼\r\n\r\n![Top](https://pic1.imgdb.cn/item/68d3aa7bc5157e1a882cc294.png)\r\n\r\n![Bottom](https://pic1.imgdb.cn/item/68d3af79c5157e1a882cc4a2.png)\r\n\r\nä¸è½½é¾æ¥å¨è¿éï¼\r\n\r\n\r\n\r\nå¦å¯¹äºï¼PCBææ ·ççµè·¯æ¿ååº¦æå¥½æ¯é1.00mmï¼å¦æä½ ééäºä¹æ²¡å³ç³»ï¼ä½ å¯ä»¥ä¸è½½Fustion360å»ä¿®æ¹éé¢çåºå£³æ¿æPCBçæ±çé«åº¦ã\r\n\r\nè¿æä¸ç¹éè¦æ³¨æçæ¯ï¼å°è´§æ¶ä¸å®è¦æ£æ¥ä¸ä¸PCBææ²¡æèªèº«å­å¨çç¼ºé·é®é¢ï¼å°¤å¶æ¯æ£æ¥å±å¹æçº¿çå¼èæ¯å¦æ­£å¸¸ï¼æå°±æ¶è¿ä¹åééäºä¸åæé®é¢çå»çæ¥ï¼åºä»¶æåç§å½ååç°å±å¹åªæå¾®å¼±èåï¼å°æåæ£æ¥é®é¢æåç°RSTå¼èç´æ¥æ¥å°äºï¼æ è¯­...\r\n\r\n## çæ¥çç®ææå\r\n\r\nå¦æä½ ç¨çæ¯æé£ä¸ªçæ¬çï¼é£ä¹ä½ å¯ä»¥å¨è¿éä½¿ç¨äº¤äºå¼BOMè¿è¡è´´çãååºä½ ççé¡èï¼å¯¹é¤äºå±å¹å¼èçåºåå¶ä»ä½ç½®ççé½ååçä¸ä¸äºé¡è/é¡æµãæå¨åæ¹å®¤åæ¸©åº¦27Â°æ¶ï¼é¡èçåæä¸»è¦è¿æ¯åºæï¼åªè¦æ§å¶å¥½é¡èå¨ççä¸çéå³å¯ã\r\n\r\n### éæ¿ç§ï¼\r\n\r\nææ¨èæ¯å±å¹è¿ä¸é¢è¿è¡éæ¿ç§ï¼è¿éåå«äºç»å¤§å¤æ°çåä»¶ï¼\r\n\r\n![InteractBom](https://pic1.imgdb.cn/item/68d3b202c5157e1a882cc58d.png)\r\n\r\nå¨çæ¥æ¶ï¼ä½ å¯ä»¥å¾éå·¦è¾¹çcheckboxæ¥ç¡®è®¤åä»¶å·²ææ¾è³PCBãå½è¿ä¸é¢çåä»¶ï¼é¤äºå±å¹ï¼é½æ¾å¥½åï¼å°å¶æ¾ç½®å¨å ç­å°ä¸ï¼å¹¶å¼å§å ç­ãè¿ä¸ªè¿ç¨ä¼æä¸äºé¾é»çå³éé",
    "æ¾ï¼å¶å®æ¯æ·»å å¨é¡èéé¢çå©çåè¸åçå³éï¼ï¼éè¦æ³¨æéé£ãè¿ä¸ªäº¤äºå¼BOMè¡¨å¯ä»¥å¨è¿éè¿è¡è®¿é®å¦ã\r\n\r\n### è¯çä¸typecæ¯åº§\r\n\r\nå®æè¿ä¸é¢ççæ¥åï¼å°æ¿å­åè¿æ¥ï¼æ¥ä¸æ¥ä¸»è¦å°±æ¯çæ¥Typecæ¯åº§åä¸»æ§è¯çäºãå¶å®æ­¥éª¤ä¹æ¯ä¸æ ·çï¼ä¾ç¶æ¯å¨ççä¸æ¤ä¸é¡èï¼ä½è¿æ¬¡ä½¿ç¨ç­é£æªå»å¹ãæ³¨æï¼è¯ççä¸å»ä¼¼ä¹å¾é¾çæ¥ï¼ä½å¶å®åªè¦ä½ çé¡èä¸è¦ç»å¤ªå°ï¼åºæ¬ä¸ä¼æé®é¢ï¼æ¯çï¼å¾é¾åºç°è¿é¡çæåµï¼åèæ¯é¡å çä¸å¤å¤ï¼ãTypeCççæ¥å°±æ´ç®åäºï¼ä½ å¯ä»¥èç¼è§å¯å°æ¯å¦æå¼èä¼æè¿é¡çæåµï¼å¦ææçè¯ä¹ä¸ç¨æå¿ï¼å¨çæ¥å®æåï¼å¨æè¿é¡çæåµçå¼èæ¤ä¸ä¸ç¹ç¹çå©çèï¼ç¶åç¨åå¤´çéæ²¿çå¼èçæ¹åå¾å¤å®ï¼ä½ ä¼åç°è¿é¡çå°æ¹æ¶å¤±äºï¼å°±æ¯è¿ä¹ç®åã\r\n\r\n### å±å¹\r\n\r\nå±å¹ççæ¥æä¸äºå°æå·§ãæççæ¥æ¹æ³æ¯è¿æ ·çï¼PCBä¸çå±å¹å¼èæ¤ä¸é¡èï¼ä½æ¤é¡èçæ¹ååæçº¿å¼èæ¹ååç´ï¼ä¹å°±æ¯ç±»ä¼¼ä¸ä¸ªâæâçå¨ä½ææ¯ä¸ªå¼èé½ä¸ä¸é¨åé¡ãè¿ä¸æ­¥å®æåï¼å°å±å¹çéææå¯¹åPCBä¸çççå¼èï¼ç¶åç¨åå¤´çéè½»è§¦ä¸¤ä¸ªè¾¹è§çä½ç½®è®©éææåççéè¿é¡è¿æ¥èµ·æ¥åºå®ä½ï¼éåå¯¹å©ä½çå¼èé½æ¯è¿æ ·æä½ãä¸»ä¹è§å¯ææ²¡æè¿é¡çæåµï¼å¦ææçè¯ï¼å ä¸ç¹å©çåç¶åæå¨åå¤´å¾å¤æå¤ä½çé¡å®èµ°ã\r\n\r\n## åºä»¶ç§å½\r\n\r\nåºä»¶ç§å½åä½ å¯ä»¥åä¸äºç®åççæ¥æ£æ¥ãä½ å¯ä»¥åæä¸USBçº¿ï¼è§å¯LED1çç»¿ç¯æ¯å¦æäº®ï¼å¦ææ²¡äº®ï¼è¯´æä½ TypeCå­å¨çæ¥é®é¢ï¼å¦æäº®äºï¼é£ä¹æ­åä½ ï¼è³å°ä½ çTypeCçVBUSçæ¥æ¯æ­£å¸¸ç~æ¥ä¸æ¥ä½ å¯ä»¥å°è¯ç§å½åºä»¶äºãç§å½çå·¥å·ä½ å¯ä»¥å¨ä¹é«çå®ç½æ¾å°ï¼åºä»¶å¯ä»¥éæ©ä½¿ç¨æç¼è¯çåºä»¶ï¼ä¹å¯ä»¥èªå·±ç¼è¯ãè¿éæå¤§çé®é¢æ¯ï¼ä½ æ æ³ç§å½åºä»¶ï¼æä»¬ç°å¨æ¥åæä¸ä¸è¿ä¸ªé®é¢ã\r\n\r\n### çµèè¯å«ä¸å°è®¾å¤/ä¸²å£",
    "\r\n\r\nè¯å«ä¸å°è®¾å¤å°±æ¯è¯´æä½ ççæ¥æé®é¢ï¼ä½ éè¦åçæ¯æ£æ¥Typcç`D-`å`D+`å¼èåè¯çä¹é´çè¿æ¥æ¯å¦æ­£å¸¸ï¼æéä¸­çæ³ç½çé¨åï¼ï¼\r\n\r\n![TypeC](https://pic1.imgdb.cn/item/68d3c545c5157e1a882cce78.png)\r\n\r\n### å¯ä»¥è¯å«è®¾å¤ï¼ä½æ¯ç§å½binæ²¡ååº\r\n\r\nå¨ç§å½è½¯ä»¶ä¸­ï¼ä½ éè¦ä½¿ç¨è¿æ ·çéç½®ï¼\r\n\r\n![step1](https://pic1.imgdb.cn/item/68d3c584c5157e1a882cceb4.png)\r\n\r\n![step2](https://pic1.imgdb.cn/item/68d3c5bcc5157e1a882cced6.png)\r\n\r\n### åå§æ§½æç§å½æ¾ç¤ºæåäºä½å±å¹ä¸äº®\r\n\r\nè¿æ¯ä¸ªéå¸¸ç¥ç§çé®é¢ï¼éå°è¿ä¸ªæåµå°±éè¦ä½ æ¿ä¸ç¨è¡¨å»æ£æµå±å¹æçº¿ãæéå°çæåµæ¯å¶æ¿é®é¢ï¼RSTå¼èæ¥å°å°äºï¼å¯¼è´å±å¹åªæéå¸¸å¾®å¼±çèåæ¾ç¤ºã**èåï¼BL/ LEDK/ LED-ï¼** æ¯ä¸ä¸ªç¸å¯¹ç¬ç«ççµè·¯ãéå¸¸åªéè¦ç»å®ä¾çµï¼æ¯å¦æ¥3.3Vï¼åå°ï¼æ¯å¦éè¿ä¸ä¸ªéæµçµé»æ¥å°ï¼ï¼èåLEDå°±ä¼äº®ãæä»¥èåäº®äºï¼è³å°è¯´æå±å¹çèåé¨åååºç¡ä¾çµå¯è½æ²¡é®é¢ãæä»¬ä½¿ç¨çå±å¹ä½¿ç¨çé©±å¨æ¯ST7789(V)ï¼**ä¸æ¾ç¤ºå¾å** æå³çä¸»é©±å¨è¯çST7789æ²¡ææ­£ç¡®æ¥æ¶å°æ°æ®ææ æ³å·¥ä½ãé®é¢å¤§æ¦çåºå¨ï¼**çµæºãå¤ä½ãéä¿¡æ»çº¿ï¼SPIï¼ãå³é®æ§å¶å¼è** ççæ¥ä¸ã\r\n\r\n- RSTå¼èéå¸¸æ¯**ä½çµå¹³å¤ä½**ãä¹å°±æ¯è¯´ï¼å¨åå§åæ¶ï¼åçæºéè¦åå°å¶æä½ä¸æ®µæ¶é´ï¼å¦10msï¼ï¼ç¶åç½®ä¸ºé«çµå¹³ãå¦æRSTå¼èæ ¹æ¬æ²¡æè¿æ¥å°åçæºï¼å®å¯è½å¤äº**æ¬ç©ºï¼Floatingï¼** ç¶æãæ¬ç©ºçå¼èçµå¹³ä¸ç¡®å®ï¼å¯è½å¯¼è´è¯çä¸ç´å¤äºå¤ä½ç¶æèæ æ³å·¥ä½ã\r\n- ä¸»çµæºå¼èéå¸¸æ¯3.3Vãç¨ä¸ç¨è¡¨æµéè¿ä¸ªå¼èå **GND** ä¹é´ççµåï¼ç¡®è®¤æ¯å¦æ¯ç¨³å®ç **3.3V** å·¦å³ã\r\n\r\nä¸è¬æ¥è¯´ä¸»è¦ææ¥è¿ä¸¤é¡¹å³å¯ãå¦å¤ä¸ç§å±å¹ä¸äº®çæåµå°±æ¯ï¼FPCè½¯æçº¿å¼¯æç¨åº¦è¿å¤§å¯¼è´å±å¹æ²¡ææ­£å¸¸ç¹äº®ãè¿ä¸ªå¤§æ¦çæ¯å±å¹çæ¥ä¸å°ä½å¯¼è´çï¼éæ°è¡¥ä¸ç¹çé¡å°±å¥½ã\r\n\r\n## 3Dæå°å¤å£³\r\n\r\n### éè²éæ©\r\n\r\n",
    "éè²éæ©ççç¦æ­»äºï¼æçç½è²æææ¯ç¹ç»´çï¼ç¹å«å®¹æç¿è¾¹ï¼ é»è²ç¨çæ¯å¤§ç®çPLAååï¼æå°ææéå¸¸å¥½ãå¶å®ä¹å¯ä»¥èèä½¿ç¨éæå£³ä½çï¼ä½æ¯æè§å¾éé¢ççµè·¯æ¿ç»çä¸æ¯ç¹å«å¥½çï¼æä»¥è¿æ¯ä¸éç¨éæäºã\r\n\r\nå¨æçè®¾æ³éé¢ï¼åºè¯¥æ¯éç¨ç½è²ä¸»ä½+æ©è²æé®ï¼ä½æ¯æçç½è²æå°ææå¤ªå·®äºã\r\n\r\n### æå°åè\r\n\r\n## ä»£ç ç¼è¯\r\n\r\nOKï¼æè¦è¯´ä¸ä¸ªäºå¿ï¼ä½èç»çä»£ç æä¸äºå°æ¹æ¯æé®é¢çï¼æä¸ç¥éæ¯ä½èçç»è¯»èçæèé¢è¿æ¯çæ¬æ´æ°ä½æ²¡æåæ­¥ï¼æ»ä¹ææä¿®å¤çä»£ç ç»æ¾å°æçGithubä»åºäºï¼ä½ å¯ä»¥ç´æ¥ä¸è½½å¹¶å¨VSCodeä¸­ä½¿ç¨PlatformIOæä»¶è¿è¡éç½®ãæå»ºãç§å½åºä»¶çæä½ã\r\n\r\n\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000"
]

  initializeLubyTransformEncoder();
  debug_load.value = true;
}
function stringToReadOnlyByteArray(str) {
  // 使用TextEncoder将字符串编码为UTF-8字节序列
  const encoder = new TextEncoder();
  const bytes = encoder.encode(str);

  // 将Uint8Array转换为只读字节数组
  const readOnlyByteArray = new Uint8Array(bytes.buffer, bytes.byteOffset, bytes.byteLength);
  console.log("str byteLength: ",bytes.byteLength)
  return readOnlyByteArray;
}
const test_render_next_frame = () => {
    if (!isWasmInitialized || chunks.value.length === 0) {
      console.warn('Cannot initialize encoder: WASM not initialized or no chunks available');
      return;
    }
    const encodedBlock = encoder.generate_block();
    currentEncodedBlock.value = encodedBlock;
    
    // 按照要求组织数据格式：seed(4字节) + degree(4字节) + filename(32字节) + encodedBlock.data(2048字节)
    const bufferSize = 4 + 4 + 32 + 2048; // 总大小为2088字节
    const buffer = new ArrayBuffer(bufferSize);
    const byteView = new Uint8Array(buffer);
    
    // 1. 写入seed（4字节，小端序）
    const seedValue = parseInt(encodedBlock.seed);
    byteView[0] = seedValue & 0xFF;
    byteView[1] = (seedValue >> 8) & 0xFF;
    byteView[2] = (seedValue >> 16) & 0xFF;
    byteView[3] = (seedValue >> 24) & 0xFF;
    
    // 2. 写入degree（4字节，小端序）
    byteView[4] = encodedBlock.degree & 0xFF;
    byteView[5] = (encodedBlock.degree >> 8) & 0xFF;
    byteView[6] = (encodedBlock.degree >> 16) & 0xFF;
    byteView[7] = (encodedBlock.degree >> 24) & 0xFF;
    
    // 3. 写入文件名（从第8字节开始，固定32字节）
    const filename = "text.txt";
    for (let i = 0; i < 32; i++) {
      if (i < filename.length) {
        byteView[i + 8] = filename.charCodeAt(i);
      } else {
        byteView[i + 8] = 0; // 剩余空间用0填充
      }
    }
    
    // 4. 写入encodedBlock.data（从第40字节开始，共2048字节）
    const dataArray = stringToReadOnlyByteArray(encodedBlock.data);
    for (let i = 0; i < 2048; i++) {
      if (i < dataArray.length) {
        byteView[i + 40] = dataArray[i];
      } else {
        byteView[i + 40] = 0; // 不足部分用0填充
      }
    }
    
    // 将 Uint8Array 转换为只读的 number[] 数组
    let readonlyByteArray = Object.freeze(Array.from(byteView));
      svgg.value = renderSVG(readonlyByteArray, {
        pixelSize: 12,
        whiteColor: '#1D1E1F',
        blackColor: '#f5eddc',
      });
      transBlockIndices.value.push({
          seed: encodedBlock.seed.toString(),
          degree: encodedBlock.degree,
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
  
  // 检查是否有可用的编码块数据（现在使用readonlyByteArray而不是dataToEncode）
  // 从currentEncodedBlock获取最新的编码块信息
  if (!currentEncodedBlock.value) {
    console.warn('No encoded block data available. Please click "Next" first.');
    decodeStatus.value = '错误：无编码块数据，请先点击Next';
    return;
  }
  
  try {
    // 直接使用currentEncodedBlock中的数据，模拟从字节数组解析的过程
    const seedValue = parseInt(currentEncodedBlock.value.seed);
    const degreeValue = currentEncodedBlock.value.degree;
    const dataContent = currentEncodedBlock.value.data;
    
    console.log(`添加编码块 #${decode_counts.value + 1}`, {
      seed: seedValue,
      degree: degreeValue,
      dataLength: dataContent.length
    });
    
    // 将编码块添加到解码器
    decoderInstance.value.add_encoded_block(
      BigInt(seedValue), 
      degreeValue, 
      dataContent
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
      console.log(results)
      
      // 验证解码结果 - 针对字节数组的特殊比较方式
      let isAllCorrect = true;
      let errorCount = 0;
      
      // 确保结果数量匹配
      if (results.length !== chunks.value.length) {
        isAllCorrect = false;
        console.error(`解码块数量不匹配：期望 ${chunks.value.length}，实际 ${results.length}`);
      } else {
        // 逐块比较
        const after_mapping = chunks.value.map(chunk => {
          // 如果需要字符串形式，可以将字节数组转换为字符串
          // 否则直接传递字节数组
          
          return String.fromCharCode(...chunk);
        });
        for (let i = 0; i < results.length; i++) {
          const decodedBlock = results[i];
          const originalBlock = after_mapping[i];
          
          // 验证块大小
          if (decodedBlock.length !== originalBlock.length) {
            isAllCorrect = false;
            errorCount++;
            console.error(`解码块 ${i} 大小不匹配：期望 ${originalBlock.length}，实际 ${decodedBlock.length}`);
            continue;
          }
          
          // 逐字节比较
          let blockMatch = true;
          for (let j = 0; j < decodedBlock.length; j++) {
            if (decodedBlock[j] !== originalBlock[j]) {
              blockMatch = false;
              break;
            }
          }
          
          if (!blockMatch) {
            isAllCorrect = false;
            errorCount++;
            console.error(`解码块 ${i} 内容不匹配`);
            // 为了性能，最多只显示前5个错误块
            if (errorCount >= 5) {
              console.error('已发现5个错误块，停止进一步验证');
              break;
            }
          }
        }
      }
      
      decodeStatus.value = isAllCorrect 
        ? `解码成功！共使用 ${decode_counts.value} 个编码块`
        : `解码完成，但 ${errorCount} 个块验证失败`;
      
      console.log("解码完成！");
      console.log(`总使用 ${decode_counts.value} 个编码块`);
      console.log(`解码结果验证：${isAllCorrect ? '全部正确' : `存在 ${errorCount} 个错误块`}`);
      
      // 如果需要，可以在这里重置解码器和计数
      // resetDecoder();
    }
  } catch (error) {
    console.error('解码过程中发生错误:', error);
    decodeStatus.value = `解码错误: ${error.message || '未知错误'}`;
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