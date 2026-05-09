<script setup>
import { nextTick, onBeforeUnmount, ref, watch } from 'vue'

defineOptions({
  name: 'VideoRater'
})

const props = defineProps({
  trial: {
    required: true
  },
  lang: {
    required: true
  },
  isExample: {
    default: false
  },
  readonlyExample: {
    default: false
  }
})

const dataURL = import.meta.env.VITE_DATA_URL

const temporalConsistency = ref(null)
const spatialAccuracy = ref(null)
const inputVideo = ref(null)
const outputAVideo = ref(null)
const outputBVideo = ref(null)
const inputFocusMap = ref(null)
const targetFocusMap = ref(null)

const emit = defineEmits(['submit'])
const showWarning = ref(false)
const isPlaying = ref(false)
const isSyncing = ref(false)
let syncIntervalId = null
let syncGeneration = 0

const DRIFT_THRESHOLD_SECONDS = 0.08
const SYNC_CHECK_MS = 160
const LOOP_EDGE_SECONDS = 0.05

const answerOptions = [
  { value: 'A', en: 'A is better', zh: 'A 較好' },
  { value: 'B', en: 'B is better', zh: 'B 較好' },
  { value: 'C', en: 'None, difficult to judge', zh: '無法判斷' }
]

function videoUrl(path) {
  return `${dataURL}/${path}`
}

function getVideoRefs() {
  return [inputVideo, inputFocusMap, targetFocusMap, outputAVideo, outputBVideo]
}

function getVideos() {
  return getVideoRefs()
    .map((videoRef) => videoRef.value)
    .filter(Boolean)
}

function clearSyncMonitor() {
  if (syncIntervalId !== null) {
    window.clearInterval(syncIntervalId)
    syncIntervalId = null
  }
}

function pauseVideos() {
  clearSyncMonitor()
  getVideos().forEach((video) => video.pause())
  isPlaying.value = false
}

function waitForVideoEvent(video, eventName, generation, timeoutMs = 5000) {
  return new Promise((resolve) => {
    if (generation !== syncGeneration) {
      resolve(false)
      return
    }

    let timeoutId = null
    const cleanup = (result) => {
      video.removeEventListener(eventName, onReady)
      video.removeEventListener('error', onError)
      if (timeoutId !== null) {
        window.clearTimeout(timeoutId)
      }
      resolve(result && generation === syncGeneration)
    }
    const onReady = () => cleanup(true)
    const onError = () => cleanup(false)

    video.addEventListener(eventName, onReady, { once: true })
    video.addEventListener('error', onError, { once: true })
    timeoutId = window.setTimeout(() => cleanup(false), timeoutMs)
  })
}

function waitForMetadata(video, generation) {
  if (video.readyState >= 1) {
    return Promise.resolve(true)
  }
  return waitForVideoEvent(video, 'loadedmetadata', generation)
}

function waitForCanPlay(video, generation) {
  if (video.readyState >= 3) {
    return Promise.resolve(true)
  }
  return waitForVideoEvent(video, 'canplay', generation)
}

function seekVideo(video, time, generation) {
  return new Promise((resolve) => {
    if (generation !== syncGeneration || !Number.isFinite(video.duration)) {
      resolve(false)
      return
    }

    const maxTime = Math.max(0, video.duration - LOOP_EDGE_SECONDS)
    const targetTime = Math.min(Math.max(0, time), maxTime)
    if (Math.abs(video.currentTime - targetTime) < 0.02) {
      resolve(true)
      return
    }

    let timeoutId = null
    const cleanup = (result) => {
      video.removeEventListener('seeked', onSeeked)
      video.removeEventListener('error', onError)
      if (timeoutId !== null) {
        window.clearTimeout(timeoutId)
      }
      resolve(result && generation === syncGeneration)
    }
    const onSeeked = () => cleanup(true)
    const onError = () => cleanup(false)

    video.addEventListener('seeked', onSeeked, { once: true })
    video.addEventListener('error', onError, { once: true })
    timeoutId = window.setTimeout(() => cleanup(false), 2500)
    video.currentTime = targetTime
  })
}

function getLoopDuration(videos) {
  const durations = videos
    .map((video) => video.duration)
    .filter((duration) => Number.isFinite(duration) && duration > 0)
  return durations.length > 0 ? Math.min(...durations) : null
}

function getLeaderVideo(videos) {
  return inputVideo.value || videos[0]
}

function startSyncMonitor() {
  clearSyncMonitor()
  syncIntervalId = window.setInterval(() => {
    if (!isPlaying.value || isSyncing.value) return

    const videos = getVideos()
    const leader = getLeaderVideo(videos)
    if (!leader) return

    const loopDuration = getLoopDuration(videos)
    if (loopDuration !== null && leader.currentTime >= loopDuration - LOOP_EDGE_SECONDS) {
      replayVideos()
      return
    }

    const targetTime = leader.currentTime
    videos.forEach((video) => {
      if (video === leader || video.paused || !Number.isFinite(video.duration)) return

      const boundedTarget = Math.min(targetTime, Math.max(0, video.duration - LOOP_EDGE_SECONDS))
      if (Math.abs(video.currentTime - boundedTarget) > DRIFT_THRESHOLD_SECONDS) {
        video.currentTime = boundedTarget
      }
    })
  }, SYNC_CHECK_MS)
}

async function playVideos(generation = syncGeneration, realign = true) {
  const videos = getVideos()
  if (videos.length === 0) return

  clearSyncMonitor()
  isSyncing.value = true

  if (realign) {
    const leader = getLeaderVideo(videos)
    const targetTime = leader?.currentTime || 0
    await Promise.all(videos.map((video) => seekVideo(video, targetTime, generation)))
  }

  if (generation !== syncGeneration) return

  const playResults = await Promise.all(
    videos.map((video) => {
      video.muted = true
      return video.play().then(
        () => true,
        () => false
      )
    })
  )

  if (generation !== syncGeneration) return

  isSyncing.value = false
  isPlaying.value = playResults.every(Boolean)
  if (isPlaying.value) {
    startSyncMonitor()
  }
}

async function replayVideos() {
  const generation = ++syncGeneration
  const videos = getVideos()
  if (videos.length === 0) return

  clearSyncMonitor()
  isSyncing.value = true
  isPlaying.value = false
  videos.forEach((video) => video.pause())

  await Promise.all(videos.map((video) => waitForMetadata(video, generation)))
  await Promise.all(videos.map((video) => seekVideo(video, 0, generation)))
  await Promise.all(videos.map((video) => waitForCanPlay(video, generation)))

  if (generation !== syncGeneration) return

  isSyncing.value = false
  await playVideos(generation, false)
}

function togglePlayback() {
  if (isPlaying.value) {
    pauseVideos()
    return
  }
  playVideos(syncGeneration)
}

function handleVideoEnded() {
  if (isPlaying.value && !isSyncing.value) {
    replayVideos()
  }
}

async function restartVideos() {
  const generation = ++syncGeneration
  clearSyncMonitor()
  isSyncing.value = true
  isPlaying.value = false

  await nextTick()

  const videos = getVideos()
  videos.forEach((video) => {
    video.pause()
    video.loop = false
    video.muted = true
    video.preload = 'auto'
    video.load()
  })

  await Promise.all(videos.map((video) => waitForMetadata(video, generation)))
  await Promise.all(videos.map((video) => seekVideo(video, 0, generation)))
  await Promise.all(videos.map((video) => waitForCanPlay(video, generation)))

  if (generation !== syncGeneration) return

  isSyncing.value = false
  await playVideos(generation, false)
}

watch(
  () => [
    props.trial.inputVideo,
    props.trial.inputFocusMap,
    props.trial.targetFocusMap,
    props.trial.outputA,
    props.trial.outputB
  ],
  () => {
    temporalConsistency.value = props.readonlyExample ? 'A' : null
    spatialAccuracy.value = props.readonlyExample ? 'A' : null
    showWarning.value = false
    restartVideos()
  },
  { immediate: true }
)

onBeforeUnmount(() => {
  clearSyncMonitor()
})

function submit() {
  if (props.readonlyExample) {
    emit('submit')
    return
  }

  if (temporalConsistency.value === null || spatialAccuracy.value === null) {
    showWarning.value = true
    return
  }
  showWarning.value = false
  emit('submit', {
    temporalConsistency: temporalConsistency.value,
    spatialAccuracy: spatialAccuracy.value
  })
}
</script>

<template>
  <div class="rater">
    <h2 class="green">
      <template v-if="isExample && lang === 'en-US'"> Practice question </template>
      <template v-else-if="isExample"> 練習題 </template>
      <template v-else-if="lang === 'en-US'"> Which output video is better? </template>
      <template v-else> 哪一個輸出影片較好？ </template>
    </h2>

    <div class="sync-controls">
      <button type="button" @click="replayVideos" :disabled="isSyncing">
        <template v-if="lang === 'en-US'"> Replay </template>
        <template v-else> 重新播放 </template>
      </button>
      <button type="button" @click="togglePlayback" :disabled="isSyncing">
        <template v-if="isPlaying && lang === 'en-US'"> Pause </template>
        <template v-else-if="isPlaying"> 暫停 </template>
        <template v-else-if="lang === 'en-US'"> Play </template>
        <template v-else> 播放 </template>
      </button>
      <span class="sync-status">
        <template v-if="isSyncing && lang === 'en-US'"> Synchronizing videos... </template>
        <template v-else-if="isSyncing"> 影片同步中... </template>
        <template v-else-if="isPlaying && lang === 'en-US'"> Synchronized playback </template>
        <template v-else-if="isPlaying"> 同步播放中 </template>
        <template v-else-if="lang === 'en-US'"> Paused </template>
        <template v-else> 已暫停 </template>
      </span>
    </div>

    <div class="video-grid reference-grid">
      <figure>
        <figcaption>
          <template v-if="lang === 'en-US'"> Input video </template>
          <template v-else> 輸入影片 </template>
        </figcaption>
        <video
          ref="inputVideo"
          :src="videoUrl(trial.inputVideo)"
          muted
          playsinline
          preload="auto"
          @ended="handleVideoEnded"
        ></video>
      </figure>

      <figure>
        <figcaption>
          <template v-if="lang === 'en-US'"> Input focus map </template>
          <template v-else> 輸入 focus map </template>
        </figcaption>
        <video
          ref="inputFocusMap"
          :src="videoUrl(trial.inputFocusMap)"
          muted
          playsinline
          preload="auto"
          @ended="handleVideoEnded"
        ></video>
      </figure>

      <figure>
        <figcaption>
          <template v-if="lang === 'en-US'"> Target focus map </template>
          <template v-else> 目標 focus map </template>
        </figcaption>
        <video
          ref="targetFocusMap"
          :src="videoUrl(trial.targetFocusMap)"
          muted
          playsinline
          preload="auto"
          @ended="handleVideoEnded"
        ></video>
      </figure>
    </div>

    <div class="video-grid output-grid">
      <figure>
        <figcaption>Video A</figcaption>
        <video
          ref="outputAVideo"
          :src="videoUrl(trial.outputA)"
          muted
          playsinline
          preload="auto"
          @ended="handleVideoEnded"
        ></video>
      </figure>

      <figure>
        <figcaption>Video B</figcaption>
        <video
          ref="outputBVideo"
          :src="videoUrl(trial.outputB)"
          muted
          playsinline
          preload="auto"
          @ended="handleVideoEnded"
        ></video>
      </figure>
    </div>

    <div class="questions-grid">
      <div class="question-block">
        <h3>
          <template v-if="lang === 'en-US'"> Temporal consistency </template>
          <template v-else> 時間與內容穩定性 </template>
        </h3>
        <p>
          <template v-if="lang === 'en-US'">
            Which output has better temporal consistency?
          </template>
          <template v-else> 哪個輸出影片的時間與內容穩定性較好？ </template>
        </p>
        <div class="radio-container">
          <div class="radio-item" v-for="option in answerOptions" :key="`temporal-${option.value}`">
            <input
              type="radio"
              :id="`temporal-${option.value}`"
              :value="option.value"
              v-model="temporalConsistency"
              :disabled="readonlyExample"
            />
            <label :for="`temporal-${option.value}`">
              <template v-if="lang === 'en-US'">{{ option.en }}</template>
              <template v-else>{{ option.zh }}</template>
            </label>
          </div>
        </div>
      </div>

      <div class="question-block">
        <h3>
          <template v-if="lang === 'en-US'"> Spatial refocusing accuracy </template>
          <template v-else> 空間對焦準確度 </template>
        </h3>
        <p>
          <template v-if="lang === 'en-US'">
            Which output follows the target focus map better?
          </template>
          <template v-else> 哪個輸出影片較符合目標 focus map？ </template>
        </p>
        <div class="radio-container">
          <div class="radio-item" v-for="option in answerOptions" :key="`spatial-${option.value}`">
            <input
              type="radio"
              :id="`spatial-${option.value}`"
              :value="option.value"
              v-model="spatialAccuracy"
              :disabled="readonlyExample"
            />
            <label :for="`spatial-${option.value}`">
              <template v-if="lang === 'en-US'">{{ option.en }}</template>
              <template v-else>{{ option.zh }}</template>
            </label>
          </div>
        </div>
      </div>
    </div>

    <div class="submit-row">
      <p class="green" v-if="showWarning">
        <template v-if="isExample && lang === 'en-US'">
          Please answer both practice questions before continuing
        </template>
        <template v-else-if="isExample"> 請回答兩個練習問題後再繼續 </template>
        <template v-else-if="lang === 'en-US'">
          Please answer both questions before submitting
        </template>
        <template v-else> 請回答兩個問題後再送出 </template>
      </p>
      <button @click="submit">
        <template v-if="isExample && lang === 'en-US'"> Start study </template>
        <template v-else-if="isExample"> 開始正式作答 </template>
        <template v-else-if="lang === 'en-US'"> Submit </template>
        <template v-else> 送出 </template>
      </button>
    </div>
  </div>
</template>

<style scoped>
p {
  margin: 4px 0 8px 0;
}

.rater {
  display: grid;
  gap: 8px;
}

h2 {
  font-size: 22px;
  line-height: 1.2;
  margin: 0;
}

.sync-controls {
  align-items: center;
  display: flex;
  flex-wrap: wrap;
  gap: 8px;
}

.sync-status {
  color: var(--color-text);
  font-size: 12px;
}

.video-grid {
  display: grid;
  gap: 8px;
}

.reference-grid {
  grid-template-columns: repeat(3, minmax(0, 1fr));
}

.output-grid {
  grid-template-columns: repeat(2, minmax(0, 1fr));
}

figure {
  margin: 0;
  min-width: 0;
}

figcaption {
  color: var(--color-heading);
  font-size: 12px;
  margin-bottom: 4px;
}

video {
  background: #000;
  border: 1px solid #888;
  display: block;
  height: auto;
  margin: 0 auto;
  object-fit: contain;
  width: 100%;
}

.questions-grid {
  display: grid;
  gap: 10px;
}

.question-block {
  border-top: 1px solid var(--color-border);
  padding: 8px 0 0 0;
}

.question-block h3 {
  color: var(--color-heading);
  font-size: 15px;
  line-height: 1.2;
  margin-bottom: 2px;
}

.radio-container {
  display: flex;
  flex-wrap: wrap;
  gap: 6px 12px;
}

.radio-item {
  display: flex;
  align-items: center;
  gap: 8px;
}

label {
  font-size: 14px;
  cursor: pointer;
}

input {
  -webkit-appearance: none;
  -moz-appearance: none;
  appearance: none;

  border-radius: 50%;
  width: 18px;
  height: 18px;

  border: 2px solid var(--color-text);
  transition: 0.2s all linear;
  margin-bottom: 5px;

  cursor: pointer;
}

input:checked {
  border: 6px solid var(--color-text);
  outline: unset !important;
  /* I added this one for Edge (chromium) support */
}

input:disabled,
input:disabled + label {
  cursor: default;
}

input:disabled:not(:checked) {
  opacity: 0.45;
}

button {
  min-height: 32px;
  padding: 0 14px;
}

.submit-row {
  align-items: center;
  display: flex;
  gap: 12px;
  justify-content: space-between;
}

.submit-row p {
  margin: 0;
}

@media (min-width: 1024px) {
  .rater {
    gap: 6px;
  }

  .video-grid {
    gap: 6px;
  }

  .reference-grid video {
    max-height: 18vh;
  }

  .output-grid video {
    max-height: 24vh;
  }

  .questions-grid {
    grid-template-columns: repeat(2, minmax(0, 1fr));
  }
}

@media (max-width: 1023px) {
  .reference-grid,
  .output-grid {
    grid-template-columns: 1fr;
  }
}
</style>
