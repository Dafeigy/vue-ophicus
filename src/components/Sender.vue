<script setup lang="ts">
// encode("123")
// ES6 import
import jsQR from "jsqr";
import {
  encode,
  renderANSI,
  renderSVG,
  renderUnicode,
  renderUnicodeCompact,
} from 'uqr'


const svgg = renderSVG("PROJECT OPHICULUS PROJECT OPHICULUSPROJECT OPHICULUSPROJECT OPHICULUSPROJECT OPHICULUSPROJECT OPHICULUSPROJECT OPHICULUSPROJECT OPHICULUSPROJECT OPHICULUSPROJECT OPHICULUSPROJECT OPHICULUSPROJECT OPHICULUSPROJECT OPHICULUSPROJECT OPHICULUSPROJECT OPHICULUSPROJECT OPHICULUSPROJECT OPHICULUSPROJECT OPHICULUSPROJECT OPHICULUSPROJECT OPHICULUSPROJECT OPHICULUSPROJECT OPHICULUSPROJECT OPHICULUSPROJECT OPHICULUSPROJECT OPHICULUSPROJECT OPHICULUSPROJECT OPHICULUSPROJECT OPHICULUSPROJECT OPHICULUSPROJECT OPHICULUSPROJECT OPHICULUSPROJECT OPHICULUSPROJECT OPHICULUSPROJECT OPHICULUSPROJECT OPHICULUSPROJECT OPHICULUSPROJECT OPHICULUSPROJECT OPHICULUSPROJECT OPHICULUSPROJECT OPHICULUSPROJECT OPHICULUSPROJECT OPHICULUSPROJECT OPHICULUSPROJECT OPHICULUSPROJECT OPHICULUSPROJECT OPHICULUSPROJECT OPHICULUSPROJECT OPHICULUSPROJECT OPHICULUSPROJECT OPHICULUSPROJECT OPHICULUS", {
    pixelSize : 12,
    whiteColor : '#1D1E1F',
    blackColor :'#f5eddc',
    }
)

const ansi = renderUnicodeCompact('PROJECT OPHICULUS', {
  ecc: 'L',
  // Border width
  border: 1,
})

// display QR Code in terminal
console.log(ansi)
import { ImageProps } from 'element-plus'

const fits = [
  'fill',
  'contain',
  'cover',
  'none',
  'scale-down',
] as ImageProps['fit'][]
const url =
  'https://fuss10.elemecdn.com/e/5d/4a731a905s94a4af544c0c25941171jpeg.jpeg'

</script>

<template>
    <div id="con" class="flex h-[100vh] flex-col items-center bg-[#202020] p-2">
        <div id="TODO" class="w-[50vmin] h-[10%]  items-center  flex text-green text-left text-4xl font-display px-2.5 font-bold bg-theme">
            PROJECT OPHICULUS [T]
        </div>
        <div id="status" class=" w-[50vmin] flex justify-center font-display flex-col " >
            <div class="card-header font-display text-3xl bg-orange px-4 mt-6">
                ENCODE STATUS
            </div>
            <div id="details" class="grid grid-cols-3 mt-2 px-4 gap-2 ">
                <p class="bg-green text-theme px-1">FILENAME:</p> <p class="col-span-2"> Helloworld.md</p>
                <p class="bg-green text-theme px-1">BLOCKS INDICES</p><p class="col-span-2">[1,2,3,4,5]</p>
                <p class="bg-green text-theme px-1">TOTAL BLOCKS</p><p class="col-span-2">68</p>
                <p class="bg-green text-theme px-1">BYTES</p><p class="col-span-2">529.21KB</p>
                <p class="bg-green text-theme px-1">BITRATE</p ><p class="col-span-2">20</p>
                <p class="bg-green text-theme px-1" >FPS</p><p class="col-span-2">23.1</p>
            </div>
            
        </div>
        <div id="img" class=" w-[50vmin] h-[50vmin]  items-center flex mt-2.5  justify-center font-mono">
            <el-card style="width: 95%; height: 95%;">
                <div v-html="svgg" class="qrcode"></div>
            </el-card>
        </div>
        <div id="control" class=" w-[50vmin] h-[5%]  items-center justify-center flex mt-6">
            <el-button type="success" color="#5c7f71">START STRAMING</el-button>
            <el-button type="warning" color="#802520">SELECT FILE </el-button>
        </div>
    </div>

</template>

<style>

</style>