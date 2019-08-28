<template>
  <div id="app" oncontextmenu="return false">
    <img alt="Vue logo" src="./assets/logo.png">
    <input v-on:change="onFileChange" type="file"/>
    <HelloWorld v-bind:myFile="myFile" msg="Welcome to Your Vue.js App"/>
  </div>
</template>

<script>
import HelloWorld from './components/HelloWorld.vue'

export default {
  name: 'app',
  components: {
    HelloWorld
  },
  data() { return {
      myFile: null
    }
  },
  methods: {
    onFileChange(e) {
      var files = e.target.files || e.dataTransfer.files
      if (files.length > 0) {
        this.myFile=files[0]
      }
    }
  },
  watch: {
    myFile(newVal, oldVal) {
//      console.log(newVal, oldVal)
    }
  }
}
</script>

<style>
#app {
  font-family: 'Avenir', Helvetica, Arial, sans-serif;
  -webkit-font-smoothing: antialiased;
  -moz-osx-font-smoothing: grayscale;
  text-align: center;
  color: #2c3e50;
  margin-top: 60px;
}
</style>
