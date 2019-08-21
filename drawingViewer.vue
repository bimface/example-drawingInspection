<template>
  <div>
    <div :id="`viewer${shareToken}`" class="viewer-wrap" @click="onCountClick"></div>
    <div class="side" :class="{'side-isedit':isEdit}">
      <!-- 叠图 -->
      <div class="title" v-if="!isEdit">
        <span>叠图</span>
        <div class="new" v-if="!isEdit && loaded">
          <a href="javascript:void(0)" class="btn btn-sm btn-primary" :class="{'btn-disable2':isView}" @click="popupAddDwg">添加图纸</a>
        </div>
      </div>
      <ul class="addedlist" v-if="!isEdit">
        <li v-for="(item, i) in dwglist[dwgIndex].addList" :key="i">
          {{item.name}}
          <div class="icon">
            <span class="iconfont icon-move" @click="align(item.shareToken)"></span>
            <span class="iconfont icon-hide-s" @click="dwgHide(item.shareToken)"></span>
            <span class="iconfont icon-delete16" @click="dwgDel(item.shareToken)"></span>
          </div>
        </li>
      </ul>
      <!-- 批注 -->
      <div class="title">
        <span>批注</span>
        <div class="new" v-if="!isEdit && loaded">
          <a href="javascript:void(0)" class="btn btn-sm btn-primary" :class="{'btn-disable2':isView}" @click="createAnnotation">添加批注</a>
        </div>
      </div>
      <div class="edit" v-if="isEdit">
        <textarea placeholder="添加批注描述" v-model="comment"></textarea>
        <a href="javascript:void(0)" class="btn btn-sm btn-primary" @click="annotationSave">保存</a>
        <a href="javascript:void(0)" class="btn btn-sm btn-default" @click="annotationCancel">取消</a>
      </div>
      <ul class="annotationlist" v-if="!isEdit">
        <li v-for="(item, i) in annotationlist" 
            :title="item.name" 
            @click="annotationGetImage(item)" 
            :class="{'on':item.isClick}" 
            :key="i">
          <img :src="item.src" alt="" />
          <div class="name" v-if="item.name.length>0">{{item.name.substring(0,8)}}</div>
          <div class="name" v-else> - </div>
          <div class="note">{{item.time}}</div>
        </li>
      </ul>
    </div>
    <div class="mask" v-if="isView">
      <a href="javascript:void(0)" class="btn btn-exit" @click="maskExit">退出</a>
    </div>
    <!-- alert -->
    <div class='alert-tips alert-align' :class="{'show':isAlign}">
      <i class="iconfont icon-information--fill"></i>{{alertTips}}
    </div>
    <div class='alert-tips' :class="{'show':alignSuccess}">
      <i class="iconfont icon-success--fill"></i>图纸叠加完成
    </div>
    <!-- popup -->
    <div class="popup-mask" v-if="isPopup"></div>
    <div class="popup popup-adddwg" v-if="isPopup">
      <div class="head">添加图纸<span class="iconfont icon-close16" @click="popupCancel"></span></div>
      <div class="content">
        请选择要添加的图纸
        <ul class=popup-adddwg-list>
          <li v-for="(item, i) in dwglist" 
            :class="{ 'disable':item.isActive || item.isAdd,
                      'on':i == addDwgIndex && !item.isAdd && !item.isActive}" 
            :key="i"
            @click="chooseDwg(i)" >
              {{item.name}}<span>已添加</span>
          </li>
        </ul>
        <div class="button">
          <a href="javascript:void(0)" class="btn btn-sm btn-primary" @click="addDwg">添加</a><a href="javascript:void(0)" class="btn btn-sm btn-default" @click="popupCancel">取消</a>
        </div>
        <div class="tips">
          <i class="iconfont icon-information-"></i>
          <p>叠图使用说明</p>
          <p>1、选择要添加的图纸；</p>
          <p>2、在需要移动的图纸上选择对齐起始点；</p>
          <p>3、选择需要对齐的终点。</p>
        </div>
      </div>
    </div>
  </div>
</template>

<script>
  import Vue from 'vue'
  import '../../../page/css/example-common.css'
  import '../../assets/css/btn.less'
  import './less/drawingInspection.less'

  export default {
    name:'drawingViewer',
    props: {
      shareToken: String,
      dwglist: Array,
      dwgIndex: Number,
    },
    data(){
      return {
        viewToken: null,
        app: null,
        viewer2D: null,
        loaded:false,
        isEdit: false,
        isView:false,
        state:null,
        comment:'',
        annotationlist:[],
        dwgViewer: {
          toolbar: null,  // 工具条
          annotationManager: null,  // annotation的绘制管理器
          annotationToolbar: null,
          annotationControl: null,  // 重写annotation的保存、取消
        },
        isPopup: false,
        addedList:[],
        addDwgIndex: null,
        isAlign: false,
        alignSuccess: false,
        countClick: 0,
        alertTips:'请在叠加图纸上选择对齐起点',
        savedState: null,
        delItemIndex: null,
      }
    },

    mounted(){
      var me = this;
      me.$http.get(`//bimface.com/api/console/share/preview/viewtoken?token=${me.shareToken}`).then((res)=>{
        if(res.data.code == 'success'){
          var viewToken = res.data.data;
          var BimfaceLoaderConfig = new BimfaceSDKLoaderConfig();
          BimfaceLoaderConfig.viewToken = viewToken;
          BimfaceSDKLoader.load(BimfaceLoaderConfig,successCallback,failureCallback);

          function successCallback() {
            // 获取DOM元素
            var domId = `viewer${me.shareToken}`;
            var dom4Show = document.getElementById(domId);
            var WebDwgConfig = new Glodon.Bimface.Application.WebApplicationDrawingSetConfig();
            WebDwgConfig.domElement = dom4Show;
            WebDwgConfig.Buttons = ['Home', "RectZoom", "DrawingMeasure", "FullScreen"];
            
            me.app = new Glodon.Bimface.Application.WebApplicationDrawingSet(WebDwgConfig);
            me.viewer2D = me.app.getViewer(); 
            //加载图纸
            me.app.addDrawing(me.shareToken, viewToken);
            drawingViewExtend(me.viewer2D);
          }

          function failureCallback(error) {
            console.log(error);
          };

          function drawingViewExtend(viewer2D) {
            var viewerEvents = Glodon.Bimface.Viewer.ViewerDrawingEvent;
            viewer2D.addEventListener(viewerEvents.Loaded, function () {
              me.loaded = true;
              me.setActiveDwg();
              //加载默认数据
              if(me.shareToken == '93a18efe' && !me.dwglist[me.dwgIndex].addList) {
                me.setDefaulShow();
              }
              //自适应屏幕大小
              window.onresize=function(){
                viewer2D.resize(document.documentElement.clientWidth,document.documentElement.clientHeight-40)
              } 
              // 添加批注按钮
              var viewerDom = viewer2D.getRootElement();
              me.dwgViewer.toolbar = viewerDom.querySelector('.bf-toolbar-bottom');
              me.defaultAnnotation();
            });
          }
        }
      })
    },

    methods: {
      //第一张图纸加载默认信息
      setDefaulShow() {
        this.loadDwg('746f3392');
        this.addedList=[{
          name: '灯具布置图',
          shareToken: '746f3392',
        }];
        this.saveState();
        this.setIsAdd();
      },
      //
      saveState(){
        this.savedState = this.viewer2D.getCurrentState();
        this.$emit('func',this.dwgIndex,this.savedState,this.addedList);
      },
      setIsAdd() {
        var addList = this.dwglist[this.dwgIndex].addList;
        if(addList) {
          addList.map((item,i) => {
            this.$emit('setAdd',item.shareToken,true);
          })
        }
       
      },
      setActiveDwg() {
        var me = this;
        var state = me.dwglist[me.dwgIndex].state;
        var addList = me.dwglist[me.dwgIndex].addList;
        state ? me.viewer2D.setCurrentState(state) : null;
        if(addList && addList.length > 0) {
          addList[0].shareToken ? me.loadDwg(addList[0].shareToken) : null;
          me.addedList = addList;
        }
        me.viewer2D.setActiveDrawing(me.shareToken);
        var activeDwgId = me.viewer2D.sets[0].id;
        me.dwglist.map((item, i) =>{
          item.isActive = false;
          if(item.shareToken == activeDwgId) {
            item.isActive = true;
          }
        })
        me.dwglist.map((item,i) => {
          me.$emit('setAdd',item.shareToken,false);
        });
        me.setIsAdd();
      },

      chooseDwg: function(index) {
        this.addDwgIndex = index;   
      },
      loadDwg(shareToken) {
        this.$http.get(`//bimface.com/api/console/share/preview/viewtoken?token=${shareToken}`).then((res)=>{  
          if(res.data.code == 'success'){
            var viewToken = res.data.data;
            this.app.addDrawing(shareToken,viewToken);
          }
        })
      },
      addDwg: function() {
        var me = this;
        var item = me.dwglist[me.addDwgIndex]
        var shareToken = item.shareToken;
        if (!item.isActive && !item.isAdd){
          me.loadDwg(shareToken);
          me.createAddedList();
          me.align(shareToken);
          me.saveState();
          me.setIsAdd();
        }
        me.popupCancel();
      },
      createAddedList() {
        var i = this.addDwgIndex;
        var item = this.dwglist[i];
        var obj = {
          name:item.name,
          shareToken:item.shareToken,
        }
        this.addedList.push(obj);
        this.addDwgIndex = null;
      },

      onCountClick(countClick) {
        if(this.isAlign) {
          this.countClick++;
          switch(this.countClick) {
            case 1:
              this.alertTips = '请选择对齐终点';
              break;
            case 2:
              this.viewer2D.endMoving();
              this.saveState();
              this.isAlign = false;
              this.alignSuccess = true;
              setTimeout(() => {
                this.alignSuccess = false;
              }, 1500);
              this.countClick = 0;
              break;
          }
        }
      },

      align(id) {
        this.isAlign = true;
        this.viewer2D.selectDrawing(id)
        this.viewer2D.startMoving();
        this.alertTips = '请在叠加图纸上选择对齐起点';
      },  
      dwgDel(id) {
        this.delItemIndex = this.dwglist.findIndex(item => item.shareToken ==id);
        this.viewer2D.removeDrawing(id);
        var arr = this.addedList;
        arr.splice(arr.findIndex(item => item.shareToken === id), 1);
        this.$emit('setAdd',id,false);
        this.saveState();
      },
      dwgHide(id) {
        var hideItem = this.viewer2D.sets.filter(item => item.id ==id);
        hideItem[0].visible ? this.viewer2D.hideDrawing(id) :this.viewer2D.showDrawing(id)
        this.saveState();
      },
      popupAddDwg: function() {
        this.isPopup = true;
      },
      popupCancel: function(){
        this.isPopup = false;
      },
      // 批注
      defaultAnnotation() {
        if (!this.dwgViewer.annotationManager) {
          var config = new Glodon.Bimface.Plugins.Annotation.AnnotationToolbarConfig();
          config.viewer = this.viewer2D.getActiveDrawing().viewerDrawing;
          var annotationToolbar = new Glodon.Bimface.Plugins.Annotation.AnnotationToolbar(config);
          
        this.dwgViewer.annotationToolbar = annotationToolbar;
        this.dwgViewer.annotationManager = annotationToolbar.getAnnotationManager();
        }
      },

      createAnnotation: function(){
        if(!this.isView){
          this.isEdit = true;
          this.$emit('edit',this.isEdit);
          this.state = [];
          this.comment = '';
          this.changeModel(1);
          this.dwgViewer.toolbar.hidden = true; 
          
          this.dwgViewer.annotationToolbar.show();
          document.querySelector('.bf-toolbar-control').style.display = 'none';
          
        }
      },

      annotationSave: function(){
        var me = this;
        me.dwgViewer.annotationManager.createSnapshot(function(data){
          var date = new Date();
          var seperator1 = "-";
          var seperator2 = ":";
          var month = date.getMonth() + 1;
          var strDate = date.getDate();
          let currentdate = date.getFullYear() + seperator1 + month + seperator1 + strDate + " " + date.getHours() + seperator2 + date.getMinutes() + seperator2 + date.getSeconds();
          let _state = me.dwgViewer.annotationManager.getCurrentState();
          let _img = data;
          var obj = {
            name:me.comment,
            time:currentdate,
            src:_img,
            state:_state,
            isClick:false
          }
          me.annotationlist.push(obj);
        });
        me.annotationCancel();
      },

      annotationCancel: function(){
        this.dwgViewer.annotationManager.endDrawing();
        this.isEdit = false;
         this.$emit('edit',this.isEdit);
        this.changeModel(0);
        this.dwgViewer.toolbar.hidden = false;
        document.querySelector('.bf-annotation').addClass('bf-hide');
      },

      annotationGetImage: function(data){
        if(!this.isEdit){
          this.isView = true;
          this.clearListStyle();
          this.changeModel(1);
          data.isClick = true;
          this.dwgViewer.annotationManager.setState(data.state);
        }
      },

      clearListStyle: function(){
        for(let i=0;i<this.annotationlist.length;i++){
          this.annotationlist[i].isClick = false;
        }
      },

      maskExit: function(){
        this.isView = false;
        this.clearListStyle();
        this.changeModel(0);
        this.dwgViewer.annotationManager.startDrawing();
        this.dwgViewer.annotationManager.endDrawing();
      },

      changeModel: function(num){
        if(num == 0){
          this.app.getToolbar("MainToolbar").show();
        } else {
          this.app.getToolbar("MainToolbar").hide();
        }
      },
      
    }
  }
</script>
