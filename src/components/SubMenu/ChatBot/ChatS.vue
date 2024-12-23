<template>
  <div className="voice_chat">
      <!-- 开始聊天 -->
      <!-- <div v-if="!onReco" className="voice_chat_wrapper">
        <div className="voice_chat_btn"
            @click="startRecorder()"
        ></div>
        <div className="voice_chat_btn_title">点击开始聊天</div>
        <div className="voice_chat_btn_prompt">聊天前请允许浏览器获取麦克风权限</div>
      </div> -->
        <!-- 结束聊天 -->

      <div className="voice_chat_dialog_wrapper">
          <div className="dialog_box" >
            <ul className="dialog_content" >
              <li id="speech_list" :key="index">
                  <div className="dialog_content_img_pp"></div>
                  <div className="dialog_content_dialogue_pp">
                      {{ nlpResult }}
                  </div>
              </li>
              <li id="speech_list" className="move_dialogue">
                  
                  <div className="dialog_content_dialogue_user">
                      {{ asrResult }}
                  </div>
                  <div className="dialog_content_img_user"></div>
              </li>
            </ul>
          </div>
          <div className="btn_end_dialog"
            @click="startRecorder()"
          >
            <span></span>
            <span>{{ onReco ? "结束说话" : "开始说话"}}</span>
          </div>

        </div>

  </div>
</template>

<script>
import { asrCollentEnv, asrOffline, asrResumeRecord, asrStopRecord } from '../../../api/ApiASR'
import { apiURL } from '../../../api/API'
import Recorder from 'js-audio-recorder'
import { nlpChat } from '../../../api/ApiNLP';


// const audioCtx = new (window.AudioContext || window.webkitAudioContext)({
//  latencyHint: 'interactive',
//  sampleRate: 24000,
// });

const recorder = new Recorder({
sampleBits: 16,                 // 采样位数，支持 8 或 16，默认是16
sampleRate: 16000,              // 采样率，支持 11025、16000、22050、24000、44100、48000，根据浏览器默认值，我的chrome是48000
numChannels: 1,                 // 声道，支持 1 或 2， 默认是1
compiling: true
})


export default {
  data () {
      return {
          onReco: false,
          onEvaluate: false,
          allResultList: [],
          asrResult: "",
          nlpResult: "",

          emotionList: [],
          //ws:"",

          initChatText: "欢迎使用语音情感助手，试试和我说话吧",
          speakingText: "我正在说话...",
          stopText: "点击下面的按钮开始说话吧！",
          nlpLoadingText: "等我想一想^-^",
          asrLoadingText: "等待小助手为您语音识别...",
      }
  },
  mounted () {
    // 初始化ws
    //this.ws = new WebSocket(apiURL.CHAT_SOCKET_RECORD);
    var _that = this
    // this.ws.addEventListener('message', function (event) {
    //     _that.allResultList.push(
    //       {
    //         value : event.data,
    //         name : "asr"
    //       }
    //     )
    //     _that.asrResult = event.data
    //     _that.$nextTick(()=>{})
    //     _that.getNlp(event.data)
    // })

    this.asrResult = this.stopText
    this.nlpResult = this.initChatText

  },
  methods: {
      // 开始录音
      startRecorder(){
        this.allResultList = []

        if (this.onEvaluate){
          return;
        }

        if (!this.onReco) { // 判断是否处于录音状态
          console.log("开始录音");
          this.asrResult = this.speakingText; // 设置当前语音识别的状态文本为“我正在说话...”
          //this.resumeRecordOnline(); // 恢复后端录音

          // if (!this.audioCtx) {
          //   this.audioCtx = new (window.AudioContext || window.webkitAudioContext)({
          //     latencyHint: 'interactive',
          //     sampleRate: 24000
          //   });
          // }

          // 获取麦克风权限
          navigator.mediaDevices.getUserMedia({ audio: true }).then(stream => {
            // 创建MediaRecorder实例
            this.mediaRecorder = new MediaRecorder(stream);
            this.audioChunks = []; // 用于存储录音数据
            this.mediaRecorder.start(); // 开始录音

            // 监听音频数据
            this.mediaRecorder.ondataavailable = (event) => {
              this.audioChunks.push(event.data); // 收集录音数据
            };

            this.mediaRecorder.onstop = () => {
              const audioBlob = new Blob(this.audioChunks, { type: 'audio/wav' }); // 转换为WAV格式
              this.audioUrl = URL.createObjectURL(audioBlob); // 创建音频URL
              this.uploadAudioFile(audioBlob); // 上传音频文件
            };

            this.onReco = true; // 设置录音状态
          }).catch(err => {
            console.error('获取麦克风权限失败:', err);
          });
          //this.initNLP(); // 初始化NLP
        }
        else {
          // 结束录音
          console.log("结束录音");
          this.asrResult = this.asrLoadingText;
          this.mediaRecorder.stop();
          this.onReco = false;

          //this.stopRecordOnline()
        }
      },

      uploadAudioFile(audioBlob) {
        const formData = new FormData();
        formData.append('audio', audioBlob, 'audio.wav'); // 将Blob添加到FormData中，文件名为“audio.wav”

        this.onEvaluate = true;
        // 使用外部API上传音频文件
        this.$http.post('http://127.0.0.1:5000/asr/comprehensive_recognition', formData)
        .then(response => {
          console.log('音频上传成功:', response.data);

          // 后续操作
          this.asrResult = response.data.text;
          this.clearEmotion();
          this.emotionList = response.data.emotion;

          this.nlpResult = this.nlpLoadingText;

          console.log('语音识别结果', this.asrResult);
          console.log('情感分析结果', this.emotionList);

          // 调用大语言模型生成答复
          let requestData = {
            text: this.asrResult,
            emotion: this.emotionList,
          }
          this.$http.post('http://127.0.0.1:5000/llm/voice_emotion_assistant',requestData)
          .then(response => {
            console.log('情感分析生成回复', response.data);
            this.nlpResult = response.data.result;
            this.onEvaluate = false;
          })
          .catch(error => {
            console.log('接口调用失败', error);
            this.nlpResult = "抱歉，网络出错请稍后再试--"
            this.onEvaluate = false;
          })

        })
        .catch(error => {
          console.error('音频上传失败:', error);
          this.asrResult = "很抱歉，网络出错，请试试重新说话吧！";
          this.onEvaluate = false;
        })
      },

      // 录音数据上传
      // uploadChunk(chunkDatas){
      //   chunkDatas.forEach((chunkData) => {
      //     this.ws.send(chunkData)
      //   })
      // },

      // 恢复后端录音
      async resumeRecordOnline(){
          const result = await asrResumeRecord();
      },
      // 暂停后端录音
      async stopRecordOnline(){
          const result = await asrStopRecord();
      },
      // 清空录音
      clearChat(){
          this.allResultList = []
      },

      // 清空情感分析结果
      clearEmotion(){
        this.emotionList = [];
      },
      // 采集环境录音


      // 初始化NLP
      initNLP(){
          // 录音暂停
          this.onRecoPause = true
          recorder.pause()
          this.stopRecordOnline()
          console.log('录音暂停')
          this.asrResult = this.stopText

          // 开场语句
          // this.allResultList.push(
          //     {
          //         value:this.initChatText,
          //         name: "nlp"
          //     }
          // )
          this.nlpResult = this.initChatText
          this.getTts(this.initChatText)

      },

      // 获得NLP聊天结果
      async getNlp(text){
          
          // 录音暂停
          this.onRecoPause = true
          recorder.pause()
          this.stopRecordOnline()
          console.log('录音暂停')

          const result = await nlpChat(text);
          if (result.data.code === 0) {
              // this.allResultList.push(
              //     {
              //         value:result.data.result,
              //         name: "nlp"
              //     }
              // )
              this.nlpResult = result.data.result
              this.getTts(result.data.result)
          } else {
              this.$message.error(result.data.message);
          }
      },


      // 获得TTS录音
      async getTts(nlpText){
          // base64
          var result = await this.$http.post("/api/tts/offline", { text : nlpText});
          if (result.data.code === 0) {
              var typedArray = this.base64ToUint8Array(result.data.result)
              this.playAudioData( typedArray.buffer )
          } else {
              this.$message.error(result.data.message)
          }
          
      },

      // bs64解码
       base64ToUint8Array(base64String) {
          const padding = '='.repeat((4 - base64String.length % 4) % 4);
          const base64 = (base64String + padding)
                          .replace(/-/g, '+')
                          .replace(/_/g, '/');

          const rawData = window.atob(base64);
          const outputArray = new Uint8Array(rawData.length);

          for (let i = 0; i < rawData.length; ++i) {
                  outputArray[i] = rawData.charCodeAt(i);
          }
          return outputArray;
          },

      // 播放音频
      playAudioData( wav_buffer ) {
          var _that = this
          audioCtx.decodeAudioData(wav_buffer, buffer => {
          var source = audioCtx.createBufferSource();
          source.onended = () => {
            // 如果被暂停
            if(_that.onRecoPause){
              console.log("恢复录音")
              // 客户端录音恢复
              this.onRecoPause = false
              recorder.resume()
              this.asrResult = this.speakingText

              // 服务器录音恢复
              this.resumeRecordOnline()
            }
            
          }
          source.buffer = buffer;
          source.connect(audioCtx.destination);
          source.start();
      }, function(e) {
          Recorder.throwError(e);
      });
  },

  }
}
</script>

<style lang="less" scoped>
@import "./style.less";
</style>