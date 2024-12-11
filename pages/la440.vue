<template>
    <div class="audio-generator" :class="{
        'left-active': currentChannel === 'Left',
        'right-active': currentChannel === 'Right'
    }">
        <div class="controls">
            <button @click="toggleAudio">
                {{ isPlaying ? 'Stop' : 'Start' }}
            </button>
            <div class="status">
                Current Channel: {{ currentChannel }}
                <div v-if="isPlaying" class="countdown">
                    Next switch in: {{ formattedCountdown }}
                </div>
            </div>
        </div>
    </div>
</template>

<script setup>
import { ref, onUnmounted, computed, onMounted } from 'vue'

const frequency = 440
const currentChannel = ref('Stopped')
const isPlaying = ref(false)
const remainingTime = ref(5000)

let audioContext = null
let oscillator = null
let stereoPanner = null
let sequenceTimeout = null

const formattedCountdown = computed(() => {
    const seconds = Math.floor(remainingTime.value / 1000)
    const milliseconds = remainingTime.value % 1000
    return `${seconds}.${milliseconds.toString().padStart(3, '0')}s`
})

const initAudioContext = () => {
    audioContext = new (window.AudioContext || window.webkitAudioContext)()
}

const stopSequence = () => {
    if (sequenceTimeout) clearTimeout(sequenceTimeout)
    if (oscillator) {
        oscillator.stop()
        oscillator.disconnect()
    }
    currentChannel.value = 'Stopped'
    isPlaying.value = false
    remainingTime.value = 5000
}

const playStereoSequence = () => {
    initAudioContext()
    isPlaying.value = true

    const playChannel = (side, panValue) => {
        return new Promise((resolve) => {
            const startTime = audioContext.currentTime

            // Update channel and start countdown
            currentChannel.value = side

            // Create audio components
            oscillator = audioContext.createOscillator()
            stereoPanner = audioContext.createStereoPanner()
            const gainNode = audioContext.createGain()

            // Complex stereo panning simulation
            oscillator.type = 'sine'
            oscillator.frequency.setValueAtTime(frequency, startTime)

            // Enhanced panning with amplitude difference
            const leftGain = side === 'Left' ? 1 : 0.1
            const rightGain = side === 'Right' ? 1 : 0.1

            const leftPanner = audioContext.createStereoPanner()
            const rightPanner = audioContext.createStereoPanner()

            leftPanner.pan.setValueAtTime(-1, startTime)
            rightPanner.pan.setValueAtTime(1, startTime)

            const leftGainNode = audioContext.createGain()
            const rightGainNode = audioContext.createGain()

            leftGainNode.gain.setValueAtTime(leftGain, startTime)
            rightGainNode.gain.setValueAtTime(rightGain, startTime)

            oscillator.connect(leftPanner)
            oscillator.connect(rightPanner)

            leftPanner.connect(leftGainNode)
            rightPanner.connect(rightGainNode)

            leftGainNode.connect(audioContext.destination)
            rightGainNode.connect(audioContext.destination)

            // Countdown logic
            const countdownStart = Date.now()
            const updateCountdown = () => {
                const elapsed = Date.now() - countdownStart
                remainingTime.value = Math.max(5000 - elapsed, 0)

                if (elapsed < 5000) {
                    requestAnimationFrame(updateCountdown)
                }
            }
            requestAnimationFrame(updateCountdown)

            // Play and stop audio
            oscillator.start(startTime)
            oscillator.stop(startTime + 5)

            // Resolve after 5 seconds
            sequenceTimeout = setTimeout(() => {
                oscillator.disconnect()
                resolve()
            }, 5000)
        })
    }

    const runSequence = async () => {
        while (isPlaying.value) {
            await playChannel('Left', -1)
            if (!isPlaying.value) break
            await playChannel('Right', 1)
        }
    }

    runSequence()
}

const toggleAudio = () => {
    if (isPlaying.value) {
        stopSequence()
    } else {
        playStereoSequence()
    }
}

onMounted(() => {
    playStereoSequence()
})

onUnmounted(() => {
    stopSequence()
    if (audioContext) {
        audioContext.close()
    }
})
</script>

<style scoped>
.audio-generator {
    position: fixed;
    top: 0;
    left: 0;
    width: 100%;
    height: 100%;
    background-color: white;
    transition: background-color 0.125s;
    display: flex;
    flex-direction: column;
    justify-content: center;
    align-items: center;
}

.left-active {
    background-color: rgba(0, 255, 0, 0.2);
}

.right-active {
    background-color: rgba(255, 0, 0, 0.2);
}

.controls {
    display: flex;
    flex-direction: column;
    align-items: center;
    gap: 10px;
    z-index: 10;
}

button {
    padding: 10px 20px;
    background-color: #4CAF50;
    color: white;
    border: none;
    border-radius: 5px;
}

.countdown {
    margin-top: 10px;
    font-weight: bold;
}
</style>