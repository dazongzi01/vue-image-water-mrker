<template>
  <div id="app">
    <h1>赵总的专属应用：输入手机号才能正常使用哦！</h1>
    <PhoneVerification v-if="!isOk" @phone-verified="onPhoneVerified"/>
    <!-- <img alt="Vue logo" src="./assets/logo.png">
    <HelloWorld msg="Welcome to Your Vue.js App"/> -->
    <ImageMarker v-else></ImageMarker>
  </div>
</template>

<script>
import HelloWorld from './components/HelloWorld.vue'
import ImageMarker from './components/ImageMarker.vue'
import PhoneVerification from './components/PhoneVerification.vue'

export default {
  name: 'App',
  components: {
    HelloWorld,
    ImageMarker,
    PhoneVerification
  },
  data(){
    return{
      isOk: false,
      }
  },
  methods: {
    onPhoneVerified(phoneNumber) {
      console.log('验证通过的手机号：', phoneNumber);
      // 这里添加您的后续处理逻辑
      if('18135773868' == phoneNumber){
        this.isOk = true;
      }else{
        alert('暗号不对')
      }
    }
  }
}
</script>

<style>
#app {
  font-family: Avenir, Helvetica, Arial, sans-serif;
  -webkit-font-smoothing: antialiased;
  -moz-osx-font-smoothing: grayscale;
  text-align: center;
  color: #2c3e50;
  margin-top: 60px;
}
</style>
