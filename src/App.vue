<script setup>
import Receiver from './components/Receiver.vue'
import Sender from './components/Sender.vue'
import Camera from './components/Camera.vue'
import { ref, provide } from 'vue'

// 当前模式：'sender' 或 'receiver'
const currentMode = ref('receiver')

// 提供切换模式的函数给子组件
const handleSwitchMode = () => {
  currentMode.value = currentMode.value === 'sender' ? 'receiver' : 'sender'
}

// 存储二维码扫描结果
const scanResult = ref('')
// 处理二维码扫描事件
const handleQRScanned = (content) => {
  console.log('接收到扫描结果:', content)
  scanResult.value = content
}

provide('handleSwitchMode', handleSwitchMode)
</script>

<template>
  <link rel="stylesheet" type='text/css' href="https://cdn.jsdelivr.net/gh/devicons/devicon@latest/devicon.min.css" />
  <div id="container" class="flex items-center justify-center bg-[#202020] h-[100vh] flex-col">
    <!-- 根据当前模式条件渲染组件 -->
    <Sender v-if="currentMode === 'sender'" />
    <Receiver v-else />
     <!-- <div class="flex items-center justify-center w-2/3 md:w-1/3 aspect-square flex-col">
      <Camera @qr-scanned="handleQRScanned" @camera-error="error => scanResult.value = `错误: ${error}`"/>
      <div id="debug" class="flex justify-center mt-2">
        <div id="debug-info" class="text-xs truncate text-[#eeeddd] bg-[#333] px-3 py-1.5 rounded-lg min-w-[200px] w-1/4 lg:w-1/4 text-center">
          {{ scanResult || 'Somthing just like this with a long string' }}
        </div>
      </div>
     </div> -->
     <div id="author"
      class="absolute bottom-[2%] hidden md:flex font-display rounded-lg md:h-[5%] md:w-[80vmin] text-md justify-center items-center bg-primary  text-[#3f3f3f]">
      <div class="text-[1.5vmin] text-center items-center flex justify-center">Build by<a href="https://github.com/Dafeigy" class="text-[#5f5f5f] hover:scale-105 hover:text-green transition-all ">&nbsp;@NUL4I </a>&nbsp;with:
        [&nbsp;
        <a href="" class="hover:-translate-y-1 hover:scale-120 transition duration-200 delay-50"><i class="devicon-typescript-plain colored"></i></a>&nbsp;+&nbsp;
        <a href="" class="hover:-translate-y-1 hover:scale-120 transition duration-200 delay-50"><i class="devicon-vuejs-plain colored"></i></a>&nbsp;+&nbsp;
        <a href="" class="hover:-translate-y-1 hover:scale-120 transition duration-200 delay-50"><i class="devicon-tailwindcss-original colored"></i></a>&nbsp;+&nbsp;
        <a href="" class="hover:-translate-y-1 hover:scale-120 transition duration-200 delay-50"><i class="devicon-rust-plain"></i></a>&nbsp;+&nbsp;
        <a href="" class="hover:-translate-y-1 hover:scale-120 transition duration-200 delay-50"><i class="devicon-wasm-original colored"></i></a>&nbsp;+&nbsp;
        <a href="" class="hover:-translate-y-1 hover:scale-120 transition duration-200 delay-50"><i class="devicon-archlinux-plain colored"></i></a>&nbsp;], deployed in <a href="">&nbsp;<i
            class="devicon-vercel-original"></i>Vercel</a>
      </div>
    </div>
  </div>


</template>

<style scoped>

</style>
