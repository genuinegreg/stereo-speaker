<template>
    <div class="video-player">
        <video ref="videoRef" :src="videoSrc" @play="onPlay" @pause="onPause" controls></video>

        <div class="pan-control">
            <label>Stereo Pan: {{ panValue.toFixed(2) }}</label>
            <input type="range" min="-1" max="1" step="0.01" :value="panValue" @input="manualPan" class="w-full" />

            <div class="resume-control" v-if="!isAutoPanning">
                <button @click="resumeAutoPan" class="resume-button">
                    Resume Auto Balance
                </button>
            </div>
        </div>

        <div class="countdown-bar" v-if="isAutoPanning">
            <div class="countdown-progress"
                :style="{ width: `${countdownProgress}%`, backgroundColor: countdownColor }"></div>
        </div>

        <div class="record-controls">
            <button @click="startRecording" :disabled="isRecording" class="record-button">
                Start Recording
            </button>
            <button @click="stopRecording" :disabled="!isRecording" class="stop-button">
                Stop Recording
            </button>
            <a ref="downloadLink" style="display: none;">Download Recording</a>
        </div>
    </div>
</template>

<script setup>
import { ref, onMounted, onUnmounted } from 'vue';

useHead({
    title: 'BBB stereo pan'
})

// Video source
const videoSrc = ref('./bbb_sunflower_1080p_30fps_normal.mp4.30sec.webm');
const videoRef = ref(null);
const audioContext = ref(null);
const sourceNode = ref(null);
const pannerNode = ref(null);
const panValue = ref(0);
const isAutoPanning = ref(false);

// Countdown-related refs
const countdownProgress = ref(0);
const countdownColor = ref('#4CAF50');
const animationFrameId = ref(null);
const panSwitchTime = ref(0);
const SWITCH_INTERVAL = 4000; // 4 seconds

// Recording-related refs
const isRecording = ref(false);
const mediaRecorder = ref(null);
const recordedChunks = ref([]);
const downloadLink = ref(null);

onMounted(() => {
    // Initialize Web Audio API
    audioContext.value = new (window.AudioContext || window.webkitAudioContext)();

    if (videoRef.value) {
        // Create audio source from video
        sourceNode.value = audioContext.value.createMediaElementSource(videoRef.value);

        // Create stereo panner
        pannerNode.value = audioContext.value.createStereoPanner();

        // Connect audio graph
        sourceNode.value.connect(pannerNode.value);
        pannerNode.value.connect(audioContext.value.destination);
    }
});

const startCountdown = () => {
    panSwitchTime.value = Date.now() + SWITCH_INTERVAL;

    const updateCountdown = () => {
        const remainingTime = panSwitchTime.value - Date.now();

        if (remainingTime <= 0) {
            let currentPan = panValue.value === -1 ? 1 : -1;

            panValue.value = currentPan;
            pannerNode.value.pan.setValueAtTime(currentPan, audioContext.value.currentTime);

            panSwitchTime.value = Date.now() + SWITCH_INTERVAL;
            countdownProgress.value = 0;

            countdownColor.value = currentPan === -1 ? '#FF5722' : '#2196F3';
        } else {
            countdownProgress.value = ((SWITCH_INTERVAL - remainingTime) / SWITCH_INTERVAL) * 100;
        }

        if (isAutoPanning.value) {
            animationFrameId.value = requestAnimationFrame(updateCountdown);
        }
    };

    animationFrameId.value = requestAnimationFrame(updateCountdown);
};

const onPlay = () => {
    isAutoPanning.value = true;
    panValue.value = -1;
    pannerNode.value.pan.setValueAtTime(-1, audioContext.value.currentTime);
    startCountdown();
};

const onPause = () => {
    if (animationFrameId.value) {
        cancelAnimationFrame(animationFrameId.value);
    }

    countdownProgress.value = 0;
    isAutoPanning.value = false;
};

const manualPan = (event) => {
    isAutoPanning.value = false;

    if (animationFrameId.value) {
        cancelAnimationFrame(animationFrameId.value);
    }

    const newPanValue = parseFloat(event.target.value);
    panValue.value = newPanValue;

    if (pannerNode.value) {
        pannerNode.value.pan.setValueAtTime(newPanValue, audioContext.value.currentTime);
    }
};

const resumeAutoPan = () => {
    if (videoRef.value && videoRef.value.paused) {
        videoRef.value.play();
    }

    isAutoPanning.value = true;
    panValue.value = -1;
    pannerNode.value.pan.setValueAtTime(-1, audioContext.value.currentTime);
    startCountdown();
};

const startRecording = async () => {
    const stream = await navigator.mediaDevices.getDisplayMedia({ video: true, audio: true });
    recordedChunks.value = [];

    mediaRecorder.value = new MediaRecorder(stream, { mimeType: 'video/webm' });

    mediaRecorder.value.ondataavailable = (event) => {
        if (event.data.size > 0) {
            recordedChunks.value.push(event.data);
        }
    };

    mediaRecorder.value.onstop = () => {
        const blob = new Blob(recordedChunks.value, { type: 'video/webm' });
        const url = URL.createObjectURL(blob);

        downloadLink.value.href = url;
        downloadLink.value.download = 'recording.webm';
        downloadLink.value.style.display = 'block';
    };

    mediaRecorder.value.start();
    isRecording.value = true;
};

const stopRecording = () => {
    if (mediaRecorder.value) {
        mediaRecorder.value.stop();
    }
    isRecording.value = false;
};

onUnmounted(() => {
    if (animationFrameId.value) {
        cancelAnimationFrame(animationFrameId.value);
    }

    if (sourceNode.value && pannerNode.value) {
        sourceNode.value.disconnect();
        pannerNode.value.disconnect();
    }
});
</script>

<style scoped>
.video-player {
    max-width: 600px;
    margin: 0 auto;
}

video {
    width: 100%;
}

.pan-control {
    margin-top: 20px;
    padding: 10px;
    background-color: #f0f0f0;
    border-radius: 8px;
}

.pan-control label {
    display: block;
    margin-bottom: 10px;
    text-align: center;
}

.countdown-bar {
    height: 10px;
    background-color: #e0e0e0;
    margin-top: 10px;
}

.countdown-progress {
    height: 100%;
    transition: width 0.1s linear;
}

.resume-control {
    display: flex;
    justify-content: center;
    margin-top: 10px;
}

.resume-button {
    background-color: #4CAF50;
    color: white;
    border: none;
    padding: 10px 15px;
    border-radius: 5px;
    cursor: pointer;
    transition: background-color 0.3s ease;
}

.resume-button:hover {
    background-color: #45a049;
}

.record-controls {
    margin-top: 20px;
    text-align: center;
}

.record-button,
.stop-button {
    background-color: #f44336;
    color: white;
    border: none;
    padding: 10px 15px;
    border-radius: 5px;
    cursor: pointer;
    margin: 5px;
    transition: background-color 0.3s ease;
}

.record-button:hover {
    background-color: #e53935;
}

.stop-button {
    background-color: #2196F3;
}

.stop-button:hover {
    background-color: #1e88e5;
}
</style>
