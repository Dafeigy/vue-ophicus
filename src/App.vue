<script setup>
import HelloWorld from './components/HelloWorld.vue'
import Receiver from './components/Receiver.vue'
import Sender from './components/Sender.vue'
// 

</script>

<template>
  <link rel="stylesheet" type='text/css' href="https://cdn.jsdelivr.net/gh/devicons/devicon@latest/devicon.min.css" />
  <div id="container" class="flex items-center justify-center bg-[#202020] h-[100vh] flex-col">
    <!-- <Receiver/> -->
     <Sender/>
     <div id="author"
      class="absolute bottom-[2%] hidden md:flex font-display rounded-lg md:h-[5%] md:w-[80vmin] text-md justify-center items-center bg-primary mt-[0.5%] text-[#3f3f3f]">
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
