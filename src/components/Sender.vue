<script setup lang="ts">
// encode("123")
// ES6 import
import jsQR from "jsqr";
import {
  renderSVG,
} from 'uqr'


const svgg = renderSVG("PROJECT OPHICULUS PROJECT OPHICULUSPROJECT OPHICULUSPROJECT OPHICULUSPROJECT OPHICULUSPROJECT OPHICULUSPROJECT OPHICULUSPROJECT OPHICULUSPROJECT OPHICULUSPROJECT OPHICULUSPROJECT OPHICULUSPROJECT OPHICULUSPROJECT OPHICULUSPROJECT OPHICULUSPROJECT OPHICULUSPROJECT OPHICULUSPROJECT OPHICULUSPROJECT OPHICULUSPROJECT OPHICULUSPROJECT OPHICULUSPROJECT OPHICULUSPROJECT OPHICULUSPROJECT OPHICULUSPROJECT OPHICULUSPROJECT OPHICULUSPROJECT OPHICULUSPROJECT OPHICULUSPROJECT OPHICULUSPROJECT OPHICULUSPROJECT OPHICULUSPROJECT OPHICULUSPROJECT OPHICULUSPROJECT OPHICULUSPROJECT OPHICULUSPROJECT OPHICULUSPROJECT OPHICULUSPROJECT OPHICULUSPROJECT OPHICULUSPROJECT OPHICULUSPROJECT OPHICULUSPROJECT OPHICULUSPROJECT OPHICULUSPROJECT OPHICULUSPROJECT OPHICULUSPROJECT OPHICULUSPROJECT OPHICULUSPROJECT OPHICULUSPROJECT OPHICULUSPROJECT OPHICULUSPROJECT OPHICULUSPROJECT OPHICULUS", {
    pixelSize : 12,
    whiteColor : '#1D1E1F',
    blackColor :'#f5eddc',
    }
)
import { ref } from 'vue';
const file = ref(null);
const chunks = ref([]);
const transBlockIndices = ref([]);
const tranFPS = ref(0);
const bitRATE = ref(0);


const handleFileClick = () => {
  const input = document.createElement('input');
  input.type = 'file';
  input.onchange = (event) => {
    file.value = event.target.files[0];
    const chunkSize = 1024 * 1024; // 1MB
    chunks.value = [];
    let start = 0;

    while (start < file.value.size) {
      const end = Math.min(start + chunkSize, file.value.size);
      chunks.value.push(file.value.slice(start, end));
      start = end;
    }

    console.log('文件切片:', chunks.value);
  };
  input.click();
};

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
    <div id="con" class="flex h-[100vh] flex-col items-center bg-[#202020] p-2">
        <div id="TODO" class="w-[50vmin] h-[10%]  items-center  flex text-green text-left text-4xl font-display px-2.5 font-bold bg-theme">
            PROJECT OPHICULUS [T]
        </div>
        <div id="status" class=" w-[50vmin] flex justify-center font-display flex-col " >
            <div class="card-header font-display text-3xl bg-orange px-4 mt-6">
                ▧ ENCODE STATUS▸
            </div>
            <div id="details" class="grid grid-cols-5 mt-2 px-4 gap-2 ">
                <p class="bg-green text-theme px-1 col-span-2">▣ FILENAME:</p> <p class="col-span-3 " v-if="file">{{ file.name }}</p><p class="col-span-3 " v-else> No File Selected yet :)</p>
                <p class="bg-green text-theme px-1 col-span-2">▣ BYTES</p><p class="col-span-3 " v-if="file" >{{ file.size }} BYTES</p><p class="col-span-3 " v-else></p>
                <p class="bg-green text-theme px-1 col-span-2">▣ TOTAL BLOCKS</p><p class="col-span-3 " v-if="file">{{ chunks.length }}</p><p class="col-span-3 " v-else></p>
                <p class="bg-green text-theme px-1 col-span-2">▣ BLOCKS INDICES</p><p class="col-span-3 " v-if="file">{{ transBlockIndices }}</p><p class="col-span-3 " v-else></p>
                <p class="bg-green text-theme px-1 col-span-2">▣ BITRATE</p ><p class="col-span-3 " v-if="file">{{ bitRATE }}</p><p class="col-span-3 " v-else></p>
                <p class="bg-green text-theme px-1 col-span-2">▣ FPS</p><p class="col-span-3 " v-if="file">{{ tranFPS}}</p><p class="col-span-3 " v-else></p>
            </div>
            
        </div>
        <div id="img" class=" w-[50vmin] h-[50vmin]  items-center flex mt-2.5  justify-center font-mono">
            <el-card style="width: 95%; height: 95%;">
                <div v-html="svgg" class="qrcode"></div>
            </el-card>
        </div>
        <div id="control" class=" w-[50vmin] h-[5%] grid-cols-2 grid px-4 text-theme">
            <el-button type="success" color="#5c7f71" @click="handleFileSlice">START STRAMING</el-button>
            <el-button type="warning" color="#ba8530"  @click="handleFileClick">SELECT FILE </el-button>
        </div>
    </div>

</template>

<style>

</style>