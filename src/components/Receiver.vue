<script setup lang="ts">
// encode("123")
// ES6 import
import jsQR from "jsqr";
import { encode, decode } from 'js-base64';


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
        <div id="TODO" class="w-[50vmin] h-[10%]  items-center  flex text-green font-bold bg-theme text-left text-4xl font-display">
            PROJECT OPHICULUS [R]
        </div>
        <div id="status" class=" w-[50vmin] flex justify-center flex-col" >

            <div class="card-header font-display text-3xl bg-orange px-4 mt-6">
                RECEIVING BLOCKS
            </div>
            <div id="blocks-display" class="grid grid-cols-16 gap-1 mt-2 px-4">
                    <div v-for="item in 63" class="rounded text-sm bg-theme text-green w-[0.8vmax] h-[0.8vmax] flex justify-center items-center">
                        {{ item }}
                    </div>
                    <div>...</div>
            </div>
                
           
        </div>
        <div id="progress" class="w-[50vmin] mt-[2%] flex items-center justify-center">
                <el-progress :percentage="43" class="text-white w-[95%]" color="pink"/>
        </div>
        <div id="img" class=" w-[50vmin] h-[50vmin]  items-center justify-center flex">
            <el-card style="width: 95%">
                <img
                    src="https://shadow.elemecdn.com/app/element/hamburger.9cf7b091-55e9-11e9-a976-7f4d0b07eef6.png"
                    style="width: 100%"
                    />
            </el-card>
        </div>
        <div id="control" class=" w-[50vmin] h-[5%]  items-center justify-center flex rounded">
            <el-button type="success" color="#5c7f71">Turn on Camera</el-button>
            <el-button type="warning" color="#b2653b">Back to Trans</el-button>
        </div>
    </div>

</template>