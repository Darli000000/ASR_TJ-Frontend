<template>
    <div class="voice_chat">
      <!-- 开始/结束按钮 -->
      <div className="voice_chat_wrapper">
        <div className="voice_chat_btn"  
            @click="toggleRecorder()">            
        </div>
        <div className="voice_chat_btn_title">点击开始聊天</div>
      </div>
      <!-- 聊天内容 -->
      <div class="voice_chat_dialog_wrapper">
        <div v-for="(message, index) in messages" :key="index" class="dialog_box">
          <div v-if="message.role === 'user'" class="dialog_user">
            <div class="dialog_content_img_user"></div>
            <div class="dialog_content_dialogue_user">{{ message.text }}</div>
          </div>
          <div v-else class="dialog_assistant">
            <div class="dialog_content_img_pp"></div>
            <div class="dialog_content_dialogue_pp">{{ message.text }}</div>
          </div>
        </div>
      </div>
    </div>
</template>

<script>
import Recorder from "js-audio-recorder";
import { nlpChat } from "../../../api/ApiNLP"; // NLP API 调用

export default {
  data() {
    return {
      onReco: false, // 是否在录音中
      recorder: null, // 录音实例
      messages: [], // 聊天记录
    };
  },
  methods: {
    // 开始/结束录音
    toggleRecorder() {
      if (this.onReco) {
        this.stopRecorder();
      } else {
        this.startRecorder();
      }
    },

    // 开始录音
    startRecorder() {
      this.recorder = new Recorder({
        sampleBits: 16,
        sampleRate: 16000,
        numChannels: 1,
      });

      this.recorder.start().then(() => {
        console.log("开始录音");
        this.onReco = true;
      });
    },

    // 停止录音并发送到 API
    stopRecorder() {
      this.recorder.stop().then((blob) => {
        console.log("录音结束");
        this.onReco = false;

        // 上传录音文件
        this.uploadAudio(blob);
      });
    },

    // 上传录音文件到 API
    async uploadAudio(blob) {
      const formData = new FormData();
      formData.append("file", blob, "audio.wav");

      try {
        const response = await this.$http.post("/api/asr", formData); // 替换为你的 API 路径
        const { userText, assistantText } = response.data;

        // 更新聊天记录
        this.messages.push({ role: "user", text: userText });
        this.messages.push({ role: "assistant", text: assistantText });
      } catch (error) {
        console.error("上传录音失败：", error);
      }
    },
  },
};
</script>


<style lang="less" scoped>
@import "./style.less";
</style>


