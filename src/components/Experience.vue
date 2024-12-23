<script setup>

import ChatT from './SubMenu/ChatBot/ChatT.vue'
import ChatS from './SubMenu/ChatBot/ChatS.vue'
import ASRT from './SubMenu/ASR/ASRT.vue'
import TTST from './SubMenu/TTS/TTST.vue'
import VPRT from './SubMenu/VPR/VPRT.vue'
import IET from './SubMenu/IE/IET.vue'
import EmotionT from './SubMenu/Emotion/EmotionT.vue'

</script>

<template>
    <div className="experience">
      <div className="experience_wrapper">
        <div className="experience_title">
          语音情感小助手
        </div>
        <div className="experience_describe">
          体验前，请允许浏览器获取麦克风权限
        </div>
        <div className="experience_content" >
          <el-tabs
            className="experience_tabs"
            type="border-card"
          >
            <el-tab-pane label="语音聊天" key="1">
              <ChatT></ChatT>
            </el-tab-pane>
            <el-tab-pane label="语音情感助手" key="2">
              <ChatS></ChatS>
            </el-tab-pane>
            <el-tab-pane label="情感分析" key="3">
            <EmotionT></EmotionT>
            </el-tab-pane>
          </el-tabs>
        </div>
      </div>
    </div>
</template>


<style lang="less">
@import "./style.less";

</style>