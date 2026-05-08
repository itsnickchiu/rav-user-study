<script setup>
import { nextTick, ref, watch } from 'vue'

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

const answerOptions = [
  { value: 'A', en: 'A is better', zh: 'A 較好' },
  { value: 'B', en: 'B is better', zh: 'B 較好' },
  { value: 'C', en: 'None, difficult to judge', zh: '無法判斷' }
]

function videoUrl(path) {
  return `${dataURL}/${path}`
}

function restartVideos() {
  nextTick(() => {
    ;[inputVideo, outputAVideo, outputBVideo, inputFocusMap, targetFocusMap].forEach((videoRef) => {
      const video = videoRef.value
      if (!video) return

      video.currentTime = 0
      const playPromise = video.play()
      if (playPromise) {
        playPromise.catch(() => {})
      }
    })
  })
}

watch(
  () => props.trial.videoId,
  () => {
    temporalConsistency.value = props.readonlyExample ? 'A' : null
    spatialAccuracy.value = props.readonlyExample ? 'A' : null
    showWarning.value = false
    restartVideos()
  },
  { immediate: true }
)

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

    <div class="video-grid reference-grid">
      <figure>
        <figcaption>
          <template v-if="lang === 'en-US'"> Input video </template>
          <template v-else> 輸入影片 </template>
        </figcaption>
        <video
          ref="inputVideo"
          :src="videoUrl(trial.inputVideo)"
          autoplay
          muted
          loop
          playsinline
          controls
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
          autoplay
          muted
          loop
          playsinline
          controls
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
          autoplay
          muted
          loop
          playsinline
          controls
        ></video>
      </figure>
    </div>

    <div class="video-grid output-grid">
      <figure>
        <figcaption>Video A</figcaption>
        <video
          ref="outputAVideo"
          :src="videoUrl(trial.outputA)"
          autoplay
          muted
          loop
          playsinline
          controls
        ></video>
      </figure>

      <figure>
        <figcaption>Video B</figcaption>
        <video
          ref="outputBVideo"
          :src="videoUrl(trial.outputB)"
          autoplay
          muted
          loop
          playsinline
          controls
        ></video>
      </figure>
    </div>

    <div class="questions-grid">
      <div class="question-block">
        <h3>
          <template v-if="lang === 'en-US'"> Temporal consistency </template>
          <template v-else> 時間一致性 </template>
        </h3>
        <p>
          <template v-if="lang === 'en-US'">
            Which output has better temporal consistency?
          </template>
          <template v-else> 哪個輸出影片的時間一致性較好？ </template>
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
}

figcaption {
  color: var(--color-heading);
  font-size: 12px;
  margin-bottom: 4px;
}

video {
  aspect-ratio: 16 / 9;
  background: #000;
  border: 1px solid #888;
  display: block;
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
