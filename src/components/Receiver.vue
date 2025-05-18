<script setup lang="ts">
import jsQR from "jsqr";
import { encode, decode } from 'js-base64';
import {
  renderSVG,
} from 'uqr'


const svgg = renderSVG("PROJECT OPHICULUS PROJECT OPHICULUSPROJECT OPHICULUSPROJECT OPHICULUSPROJECT OPHICULUSPROJECT OPHICULUSPROJECT OPHICULUSPROJECT OPHICULUSPROJECT OPHICULUSPROJECT OPHICULUSPROJECT OPHICULUSPROJECT OPHICULUSPROJECT OPHICULUSPROJECT OPHICULUSPROJECT OPHICULUSPROJECT OPHICULUSPROJECT OPHICULUSPROJECT OPHICULUSPROJECT OPHICULUSPROJECT OPHICULUSPROJECT OPHICULUSPROJECT OPHICULUSPROJECT OPHICULUSPROJECT OPHICULUSPROJECT OPHICULUSPROJECT OPHICULUSPROJECT OPHICULUSPROJECT OPHICULUSPROJECT OPHICULUSPROJECT OPHICULUSPROJECT OPHICULUSPROJECT OPHICULUSPROJECT OPHICULUSPROJECT OPHICULUSPROJECT OPHICULUSPROJECT OPHICULUSPROJECT OPHICULUSPROJECT OPHICULUSPROJECT OPHICULUSPROJECT OPHICULUSPROJECT OPHICULUSPROJECT OPHICULUSPROJECT OPHICULUSPROJECT OPHICULUSPROJECT OPHICULUSPROJECT OPHICULUSPROJECT OPHICULUSPROJECT OPHICULUSPROJECT OPHICULUSPROJECT OPHICULUSPROJECT OPHICULUS", {
    pixelSize : 12,
    whiteColor : '#1D1E1F',
    blackColor :'#f5eddc',
    }
)

</script>

<template>
    <div id="con" class="flex h-[100vh] flex-col items-center bg-[#202020] p-2">
        <div id="TODO" class="w-[50vmin] h-[10%]  items-center  flex text-green font-bold bg-theme text-left text-4xl px-2.5 font-display">
            PROJECT OPHICULUS [R]
        </div>
        <div id="status" class=" w-[50vmin] flex justify-center flex-col" >

            <div class="card-header font-display text-3xl bg-orange px-4 mt-6">
                ▧ RECEIVING STATUS▸
            </div>
            <div id="blocks-display" class="grid grid-cols-16 gap-1 mt-2 px-4 grid-rows-6">
                    <div v-for="item in 95" class="cursor-pointer rounded text-sm  border-1 text-theme w-[0.8vmax] h-[0.8vmax] flex justify-center items-center">
                        {{ item }}
                    </div>
                    <div>...</div>
            </div>
                
        </div>
        <div id="progress" class="w-[50vmin] mt-4 flex items-center justify-center">
                <el-progress :percentage="43" class="text-white w-[95%]" color="pink"/>
        </div>
        <div id="img" class=" w-[50vmin] h-[50vmin]  items-center justify-center flex">
            <el-card style="width: 95%">
                 <div v-html="svgg" class="qrcode"></div>
            </el-card>
        </div>
        <div id="control" class=" w-[50vmin] h-[5%] grid-cols-2 grid px-4 text-theme">
            <el-button type="success" color="#5c7f71">TURN ON CAMERA</el-button>
            <el-button type="warning" color="#ba8530">DOWNLOAD</el-button>
        </div>
    </div>

</template>