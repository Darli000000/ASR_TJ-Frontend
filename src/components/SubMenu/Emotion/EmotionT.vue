<template>
    <div class="container_">
        <div class="left">
            <div class="icon-circle">
                <svg class="icon" viewBox="0 0 24 24">
                    <path
                        d="M12 14c1.66 0 3-1.34 3-3V5c0-1.66-1.34-3-3-3S9 3.34 9 5v6c0 1.66 1.34 3 3 3zm5.91-3c-.49 0-.9.36-.98.85C16.52 14.2 14.47 16 12 16s-4.52-1.8-4.93-4.15c-.08-.49-.49-.85-.98-.85-.61 0-1.09.54-1 1.14.49 3 2.89 5.35 5.91 5.78V20c0 .55.45 1 1 1s1-.45 1-1v-2.08c3.02-.43 5.42-2.78 5.91-5.78.1-.6-.39-1.14-1-1.14z" />
                </svg>
            </div>
            <div id="recordingTime" v-if="isRecording">{{ formattedTime }}</div>

            <div class="buttons-container">
                <button @click="startRecording" class="btn" v-if="!isRecording">{{ recordButtonText }}</button>
                <button @click="stopRecording" class="btn recording" v-if="isRecording">停止录音</button>
                <button @click="uploadRecording" class="btn save" v-if="canSave">情感分析</button>
            </div>

            <div class="recording-status" :class="{ active: isRecording }">
                <div class="pulse"></div>
                正在录音...
            </div>

            <p class="tip-text">
                点击"开始识别"开始录音，请对准您想识别的语音。<br />
                我们以协助您准确的分析。<br />
                请先许可录音权限
            </p>
        </div>

        <div class="right">
            <div class="result-section">
                <h2 class="section-title">识别结果</h2>
                <div class="content">
                    <ul v-if="topThreeEmotions.length > 0" class="emotion-list">
                        <li v-for="(emotion, index) in topThreeEmotions" :key="index" class="emotion-item">
                            <span class="emotion-label">
                                <span v-if="emotion.label === '<unk>'">未知/unknown</span>
                                <span v-else>{{ emotion.label }}</span>
                            </span>
                            <span class="emotion-score">{{ (emotion.score * 100).toFixed(2) }}%</span>
                        </li>
                    </ul>
                </div>


            </div>
        </div>
    </div>
</template>

<script setup>
import { ref } from 'vue'
import axios from 'axios'

const isRecording = ref(false)
const canSave = ref(false)
const recordButtonText = ref('开始识别')
const recordingTime = ref(0)
const audioChunks = ref([])
let mediaRecorder = null
let timerInterval = null
const formattedTime = ref()

const updateTimer = () => {
    const minutes = Math.floor(recordingTime.value / 60)
    const seconds = recordingTime.value % 60
    return `${minutes.toString().padStart(2, '0')}:${seconds.toString().padStart(2, '0')}`
}

const startTimer = () => {
    recordingTime.value = 0
    timerInterval = setInterval(() => {
        recordingTime.value++
        formattedTime.value = updateTimer()
    }, 1000)
}

const stopTimer = () => {
    clearInterval(timerInterval)
}

const startRecording = async () => {
    try {
        const stream = await navigator.mediaDevices.getUserMedia({ audio: true })
        mediaRecorder = new MediaRecorder(stream)
        audioChunks.value.length = 0

        mediaRecorder.addEventListener('dataavailable', (event) => {
            audioChunks.value.push(event.data)
        })

        mediaRecorder.start()
        startTimer()

        isRecording.value = true
        canSave.value = false
        recordButtonText.value = '重新录音'
    } catch (error) {
        console.error('Error accessing microphone:', error)
        alert('无法访问麦克风，请确保已授予录音权限。')
    }
}

const stopRecording = () => {
    if (mediaRecorder && mediaRecorder.state === 'recording') {
        mediaRecorder.stop()
        stopTimer()

        isRecording.value = false
        canSave.value = true
    }
}

const topThreeEmotions = ref([]); // 存储前三高的情绪

const uploadRecording = async () => {
    const audioBlob = new Blob(audioChunks.value, { type: 'audio/wav' });
    const formData = new FormData();
    formData.append('audio', audioBlob, 'recording.wav');

    try {
        const response = await axios.post('http://127.0.0.1:5000/asr/emotion_evaluate', formData, {
            headers: {
                'Content-Type': 'multipart/form-data',
            },
        });

        // 确保返回数据中存在 result 并且是数组
        const resultArray = response.data.result;
        if (Array.isArray(resultArray) && resultArray.length > 0) {
            const result = resultArray[0]; // 获取第一个结果对象
            const labels = result.labels; // 获取情绪标签
            const scores = result.scores; // 获取情绪分数

            // 将 labels 和 scores 合并，并按分数排序
            const emotions = labels.map((label, index) => ({
                label,
                score: scores[index],
            }));
            const sortedEmotions = emotions.sort((a, b) => b.score - a.score); // 按分数降序排序

            // 提取前三高的情绪
            topThreeEmotions.value = sortedEmotions.slice(0, 3);
            console.log('前三高的情绪:', topThreeEmotions.value);
        } else {
            console.error('后端返回的 result 数组为空或格式不正确:', response.data);
        }
    } catch (error) {
        console.error('上传失败:', error);
    }
};



</script>


<style lang="less">
.container_ {
    width: 1200px;
    height: 410px;
    display: flex;
    align-items: center;
    justify-content: space-between;
    background: linear-gradient(135deg, #f5f7fa 0%, #e4e8ec 100%);

    .left {
        width: 50%;
        height: 100%;
        text-align: center;
        background-color: #fff;
        border-radius: 16px;
        box-shadow: 0 8px 30px rgba(0, 0, 0, 0.08);
        animation: fadeIn 0.5s ease-out;


        .icon-circle {
            width: 80px;
            height: 80px;
            background: #f3f7ff;
            border-radius: 50%;
            display: flex;
            align-items: center;
            justify-content: center;
            margin: 100px auto 30px;
            /* 设置 top 的间距为 20px */
        }

        .icon {
            width: 32px;
            height: 32px;
            fill: #4169e1;
        }

        .btn {
            background: #4169e1;
            color: white;
            border: none;
            padding: 12px 32px;
            border-radius: 8px;
            font-size: 16px;
            cursor: pointer;
            transition: all 0.3s ease;
            margin: 10px 5px;
        }

        .btn:hover {
            background: #3154b3;
            transform: translateY(-1px);
        }

        .btn.recording {
            background: #dc3545;
        }

        .btn.recording:hover {
            background: #bb2d3b;
        }

        .btn.save {
            background: #28a745;
        }

        .btn.save:hover {
            background: #218838;
        }

        .btn:disabled {
            background: #ccc;
            cursor: not-allowed;
            transform: none;
        }

        .tip-text {
            color: #666;
            font-size: 14px;
            line-height: 1.6;
            margin: 20px 0;
        }

        .recording-status {
            margin-top: 20px;
            display: flex;
            align-items: center;
            justify-content: center;
            gap: 8px;
            color: #dc3545;
            font-size: 14px;
            display: none;
        }

        .recording-status.active {
            display: flex;
        }

        .pulse {
            width: 8px;
            height: 8px;
            background: #dc3545;
            border-radius: 50%;
            animation: pulse 1s infinite;
        }

        @keyframes pulse {
            0% {
                transform: scale(1);
                opacity: 1;
            }

            50% {
                transform: scale(1.5);
                opacity: 0.5;
            }

            100% {
                transform: scale(1);
                opacity: 1;
            }
        }

        .buttons-container {
            display: flex;
            justify-content: center;
            gap: 10px;
            flex-wrap: wrap;
        }

        #recordingTime {
            font-size: 24px;
            font-weight: bold;
            color: #333;
            margin: 20px 0;
        }
    }

    .right {
        width: 50%;
        height: 100%;
        text-align: left;
        background-color: transparent;
        border-radius: 16px;
        box-shadow: 0 8px 30px rgba(0, 0, 0, 0.08);
        animation: fadeIn 0.5s ease-out;

        .result-section {
            height: 100%;
            width: 100%;
            display: flex;
            flex-direction: column;
            align-items: center;
            background: white;
            border-radius: 16px;
            padding: 30px;
            box-shadow: 0 4px 20px rgba(0, 0, 0, 0.05);
        }

        .section-title {
            height: 30px;
            font-size: 18px;
            color: #333;
            margin-bottom: 30px;
            padding-bottom: 15px;
            border-bottom: 1px solid #eee;
        }

        .content {
            width: 100%;
            flex: 1;
            background-color: rgb(234, 234, 234);
            padding: 10px;
            border-radius: 8px;
            overflow-y: auto;
            /* 如果内容较多，添加滚动 */
            font-size: 16px;
            color: #333;
            list-style: none;

            .emotion-list {
                padding: 0;
                margin: 0;
            }

            .emotion-item {
                display: flex;
                justify-content: flex-start;
                /* 让内容从左对齐 */
                gap: 10px;
                /* 标签与分数之间的间距 */
                margin-bottom: 10px;
            }

            .emotion-label {
                flex: 0 0 auto;
                text-align: left;
            }

            .emotion-score {
                flex: 0 0 auto;
                /* 固定宽度，避免分数偏移 */
                text-align: left;
                /* 左对齐分数 */
            }
        }


        .content ul {
            padding: 0;
            margin: 0;
        }

        .content li {
            margin-bottom: 10px;
        }

        .content p {
            text-align: center;
            color: #666;
        }

    }
}
</style>