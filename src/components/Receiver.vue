<script setup lang="ts">
// encode("123")
// ES6 import
import jsQR from "jsqr";
import { encode, decode } from 'js-base64';
console.log('Origin:123')
console.log(decode("MTIz"))
console.log(encode("123"))


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
        <div id="TODO" class="w-[50vmin] h-[10%]  items-center  flex text-theme text-left text-4xl font-display">PROJECT OPHICULUS</div>
        <div id="status" class=" w-[50vmin] inline-block" >
             <el-collapse v-model="activeNames" @change="handleChange">
                <el-collapse-item title="Blocks" name="1">
                    <el-space wrap><div v-for="item in 64" class="rounded bg-orange w-[0.8vmax] h-[0.8vmax] items-center flex justify-center">
                    {{ item }}
                    </div></el-space>
                    
                </el-collapse-item>
            </el-collapse>
        </div>
        <div id="progress" class="w-[50vmin]">
            <el-progress :percentage="50" class="text-white"/>
        </div>
        <div id="img" class=" w-[50vmin] h-[50vmin]  items-center justify-center flex">
            <el-empty>
            </el-empty>

        </div>
        <div id="control" class=" w-[50vmin] h-[5%]  items-center justify-center flex rounded">
            <el-button type="success" color="#5c7f71">Turn on Camera</el-button>
            <el-button type="warning" color="#b2653b">Back to Trans</el-button>
        </div>
    </div>

</template>