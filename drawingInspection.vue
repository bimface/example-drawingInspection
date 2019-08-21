<template>
  <div id="app">
    <header>
      <a href="#" class="logo"></a>
      <h2>基于BIM的设计协同平台<i>官方示例</i></h2>
      <div class="btn-box">
        <a href="https://github.com/bimface/example-team" target="_blank" class="btn btn-sm btn-primary">源代码</a>
      </div>
    </header>

    <div class="container" :style="{'height':cHeight + 'px'}">
      <ul class="dwglist" v-if="!isEditHide">
        <li v-for="(item, i) in dwgDefault" 
        :key="i"
        :class="{ 'main':i==nowIndex}" 
        @click="toggleDwg(i)" >
          {{item.name}}
        </li>
      </ul>
      <div v-for="(item, i) in dwgDefault" :key="i" >
        <drawingViewer :shareToken = "item.shareToken" 
                       :dwglist = "dwgDefault"
                       :dwgIndex = "i"
                       v-if = "nowIndex == i"
                       @func = "getState"
                       @edit = "editState"
                       @setAdd = "setAdd">
        </drawingViewer>
      </div>
    </div>

    <footer>
      <div class="w1200">
        <div class="copyright">
          Copyright ©2016-2019 <a href="http://bimface.com" target="_blank">BIMFACE</a> 京ICP备10021606号-19 <a href="http://www.glodon.com/" target="_blank">广联达</a>旗下产品
        </div>
      </div>
    </footer>
  </div>
</template>

<script>
  import Vue from 'vue'
  import '../../../page/css/example-common.css'
  import '../../assets/css/btn.less'
  import './less/drawingInspection.less'
  import drawingViewer from './drawingViewer.vue'
  Vue.component('drawingViewer', drawingViewer)

  export default {
    data(){
      return {
        cHeight:null,
        nowIndex:0,
        isEditHide: false,
        dwgDefault:[{
          name: '家具布置图',
          shareToken: '93a18efe',
          state: null,
          addList: null,
        }, {
          name: '灯具布置图',
          shareToken: '746f3392',
          state: null,
          addList: null,
        }, {
          name: '水路布置图',
          shareToken: '0b4859eb',
          state: null,
          addList: null,
        },],
      }
    },

    mounted(){
      this.cHeight = document.documentElement.clientHeight - 105 - 40;
    },

    methods: {
      toggleDwg(index) {
        this.nowIndex = index;
      }, 
      getState(i,savedata,addList,editState) {
        this.dwgDefault[i].state = savedata;
        this.dwgDefault[i].addList = addList;
      },
      editState(editState) {
        this.isEditHide = editState;
      },
      setAdd(shareToken,bool) {
        var i = this.dwgDefault.findIndex(item => item.shareToken ==shareToken);
        this.dwgDefault[i].isAdd = bool;
      }
    }
  }
</script>
