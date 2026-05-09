<script setup>
import { ref, reactive, watch, onMounted, onBeforeUnmount } from 'vue'
import Rater from './components/Rater.vue'

const email = ref('')
const round = ref(null)
const showExample = ref(false)
const proposedMethod = 'rav'
const baselineMethods = ['genfocus', 'diffcamera', 'learn2refocus']
const videoRoot = import.meta.env.VITE_VIDEO_ROOT || 'videos_15fps'

function videoPath(path) {
  return `${videoRoot.replace(/\/$/, '')}/${path.replace(/^\//, '')}`
}

const studyVideos = [
  {
    dataset: 'UltraVideo',
    videoId: '12749f9c-c018-4190-9950-425780029a93',
    videoType: 'tc',
    basePath: videoPath('UltraVideo/tc/12749f9c-c018-4190-9950-425780029a93')
  },
  {
    dataset: 'UltraVideo',
    videoId: 'a292e712-f9ea-4763-8e1a-ac56621a5d1b',
    videoType: 'tc',
    basePath: videoPath('UltraVideo/tc/a292e712-f9ea-4763-8e1a-ac56621a5d1b')
  },
  {
    dataset: 'UltraVideo',
    videoId: '04b9b8a5-4527-4d16-8eda-3daf06a2fb50',
    videoType: 'tv',
    basePath: videoPath('UltraVideo/tv/04b9b8a5-4527-4d16-8eda-3daf06a2fb50')
  },
  {
    dataset: 'UltraVideo',
    videoId: 'b27e6f4c-fb68-489d-92d1-3180882e4266',
    videoType: 'tv',
    basePath: videoPath('UltraVideo/tv/b27e6f4c-fb68-489d-92d1-3180882e4266')
  },
  {
    dataset: 'CelebV-HQ',
    videoId: 'jBvjrK-pgrA_45_1',
    videoType: 'tc',
    basePath: videoPath('CelebV-HQ/tc/jBvjrK-pgrA_45_1')
  },
  {
    dataset: 'CelebV-HQ',
    videoId: 'mt1R5svwMEc_1',
    videoType: 'tc',
    basePath: videoPath('CelebV-HQ/tc/mt1R5svwMEc_1')
  },
  {
    dataset: 'CelebV-HQ',
    videoId: 'jvBp6TqoHWw_2_2',
    videoType: 'tv',
    basePath: videoPath('CelebV-HQ/tv/jvBp6TqoHWw_2_2')
  },
  {
    dataset: 'CelebV-HQ',
    videoId: 'k0sEhT8n6VM_6',
    videoType: 'tv',
    basePath: videoPath('CelebV-HQ/tv/k0sEhT8n6VM_6')
  },
  {
    dataset: 'InTheWild',
    videoId: 'american_psycho4',
    videoType: 'real',
    basePath: videoPath('itw/american_psycho4')
  },
  {
    dataset: 'InTheWild',
    videoId: 'breaking_bad2',
    videoType: 'real',
    basePath: videoPath('itw/breaking_bad2')
  }
]
const requestedRoundCount = Number(import.meta.env.VITE_ROUNDS_PER_RECORD)
const totalTrialCount = studyVideos.length * baselineMethods.length
const totalRound =
  Number.isFinite(requestedRoundCount) && requestedRoundCount > 0
    ? Math.min(requestedRoundCount, totalTrialCount)
    : totalTrialCount
const estimatedTime = '10 to 15 minutes'
const exampleBasePath = videoPath('UltraVideo/tc/13d46d36-a42c-48f3-91c0-a9977bb48a8c')
const exampleTrial = reactive({
  dataset: 'UltraVideo',
  videoId: '13d46d36-a42c-48f3-91c0-a9977bb48a8c',
  videoType: 'tc',
  inputVideo: `${exampleBasePath}/input.mp4`,
  inputFocusMap: `${exampleBasePath}/input_map.mp4`,
  targetFocusMap: `${exampleBasePath}/output_map.mp4`,
  methodA: 'rav',
  methodB: 'genfocus',
  outputA: `${exampleBasePath}/rav.mp4`,
  outputB: `${exampleBasePath}/genfocus.mp4`
})

function getRandomInt(max) {
  return Math.floor(Math.random() * max)
}

function shuffle(items) {
  const shuffled = [...items]
  for (let i = shuffled.length - 1; i > 0; i--) {
    const j = getRandomInt(i + 1)
    ;[shuffled[i], shuffled[j]] = [shuffled[j], shuffled[i]]
  }
  return shuffled
}

function shuffleBlockWithoutBoundaryRepeat(items, previousVideoId) {
  const shuffled = shuffle(items)
  if (!previousVideoId || shuffled[0]?.video.videoId !== previousVideoId) {
    return shuffled
  }

  const swapIndex = shuffled.findIndex((item) => item.video.videoId !== previousVideoId)
  if (swapIndex > 0) {
    ;[shuffled[0], shuffled[swapIndex]] = [shuffled[swapIndex], shuffled[0]]
  }
  return shuffled
}

function createTrialPlans() {
  const baselineOrderByVideo = studyVideos.map((video) => ({
    video,
    baselineOrder: shuffle(baselineMethods)
  }))

  const trialPlans = []
  let previousVideoId = null

  for (let baselineIndex = 0; baselineIndex < baselineMethods.length; baselineIndex++) {
    const block = baselineOrderByVideo.map(({ video, baselineOrder }) => ({
      video,
      comparedBaseline: baselineOrder[baselineIndex]
    }))
    const shuffledBlock = shuffleBlockWithoutBoundaryRepeat(block, previousVideoId)
    trialPlans.push(...shuffledBlock)
    previousVideoId = shuffledBlock[shuffledBlock.length - 1]?.video.videoId
  }

  return trialPlans
}

const checked = ref(false)
const showWarning = ref(false)
function start() {
  if (checked.value === false) {
    showWarning.value = true
    return
  }
  showWarning.value = false
  showExample.value = true
}

function beginStudy() {
  showExample.value = false
  round.value = 1
  setTrial(1)
  window.scrollTo({ top: 0 })
}

const trial = reactive({})
const trialQueue = createTrialPlans()
  .slice(0, totalRound)
  .map(({ video, comparedBaseline }) => {
    const [methodA, methodB] = shuffle([comparedBaseline, proposedMethod])

    return {
      ...video,
      inputVideo: `${video.basePath}/input.mp4`,
      inputFocusMap: `${video.basePath}/input_map.mp4`,
      targetFocusMap: `${video.basePath}/output_map.mp4`,
      comparedBaseline,
      methodA,
      methodB,
      outputA: `${video.basePath}/${methodA}.mp4`,
      outputB: `${video.basePath}/${methodB}.mp4`
    }
  })

function setTrial(nextRound) {
  Object.assign(trial, trialQueue[nextRound - 1])
}

const thankYou = reactive({
  show: false,
  countdown: 10
})

function createSessionId() {
  if (window.crypto?.randomUUID) {
    return window.crypto.randomUUID()
  }
  return `${Date.now()}-${Math.random().toString(36).slice(2)}`
}
const sessionId = createSessionId()

function getGoogleFormEntries() {
  try {
    return JSON.parse(import.meta.env.VITE_GOOGLE_FORM_ENTRIES || '[]')
  } catch (error) {
    console.warn('Could not parse VITE_GOOGLE_FORM_ENTRIES.', error)
    return []
  }
}

function logTrial(answers) {
  const keys = getGoogleFormEntries()
  const values = [
    email.value,
    sessionId,
    round.value,
    trial.dataset,
    trial.videoId,
    trial.videoType,
    trial.inputVideo,
    trial.inputFocusMap,
    trial.targetFocusMap,
    trial.comparedBaseline,
    trial.methodA,
    trial.methodB,
    trial.outputA,
    trial.outputB,
    answers.temporalConsistency,
    answers.spatialAccuracy
  ]

  if (!import.meta.env.VITE_GOOGLE_FORM_URL || keys.length !== values.length) {
    console.warn('Google Form is not configured yet. Skipping submission.', {
      expectedEntryCount: values.length,
      actualEntryCount: keys.length,
      values
    })
    return
  }

  const params = new URLSearchParams()
  keys.forEach((key, index) => {
    params.append(key, values[index] ?? '')
  })
  const formResponse = `${import.meta.env.VITE_GOOGLE_FORM_URL}?${params.toString()}`
  fetch(formResponse, { mode: 'no-cors' })
}

function submit(answers) {
  logTrial(answers)

  if (round.value >= totalRound) {
    thankYou.show = true
    setInterval(() => thankYou.countdown--, 1000)
    return
  }

  round.value += 1
  setTrial(round.value)
}
watch(thankYou, (newThankYou) => {
  if (newThankYou.countdown === 0) {
    location.reload()
  }
})

const lang = ref('zh-TW')

// sync width of the header and main content on mobile device
function onResize() {
  const appDOM = document.getElementById('app')
  const headerDiv = document.getElementById('header-div')
  if (headerDiv !== null) {
    if (window.matchMedia('(max-width: 1023px)').matches) {
      headerDiv.style.width = appDOM.offsetWidth + 'px'
    } else {
      headerDiv.style.width = null
    }
  }
}

const resizeObserver = new ResizeObserver(onResize)
onMounted(() => {
  resizeObserver.observe(document.getElementById('app'))
})
onBeforeUnmount(() => {
  resizeObserver.unobserve(document.getElementById('app'))
})
</script>

<template>
  <div v-if="thankYou.show">
    <template v-if="lang === 'zh-TW'">
      感謝您的填寫，我們已收到您的意見。 本頁面將在 {{ thankYou.countdown }} 秒後重新載入，
      您可以再次使用同一個 email 填寫。
    </template>
    <template v-else>
      We have received your submission. Thank you! This page will reload in
      {{ thankYou.countdown }} seconds. You can enter the same email for a new submission.
    </template>
  </div>
  <template v-else>
    <div class="main">
      <div v-if="round === null && !showExample">
        <div class="lang-selector">
          <span :class="{ green: lang === 'zh-TW' }" @click="lang = 'zh-TW'">中文</span> /
          <span :class="{ green: lang !== 'zh-TW' }" @click="lang = 'en-US'">Eng</span>
        </div>
        <h1 class="green">RAV</h1>
        <p>
          <template v-if="lang === 'zh-TW'">
            <!-- 感謝您參加本次測驗，本測驗共30題，我們將抽出五位完成作答的幸運兒，贈與超商禮券100元。
            您可以提供 email 讓我們在抽到您時通知您。如中途離開，本測驗不保留您未完成的作答。 -->

            大家好，我們是中央研究院資訊創新研究中心陳駿丞副研究員的研究團隊。
            <br /><br />
            我們正在進行 AI
            生成影像與影片之使用者偏好評量相關研究。本研究旨在評估現有影像與影片生成模型的生成品質，通過主觀和客觀的評估方法，分析模型生成影像的品質，進一步了解並分析生成模型的特性。
            <br /><br />
            本研究將透過此網站進行問卷調查，提供您由模型預先生成的影片，並請您比較影片的時間與內容穩定性與重新對焦準確度。本問卷共有
            {{ totalRound }} 題，預計需要約 10 至 15 分鐘。 <br /><br />
            有效填寫問卷者可選擇留下email參加抽獎，我們將抽出有效問卷的 1/20 贏取7-11百元禮券。
          </template>
          <template v-else>
            <!-- Thanks for your interest in participating this test. The test contains 30 questions.
            We have $100 NTD coupon giveaway to 5 random selected participates who complete the test.
            You're welcome to fill in your email address so we can reach you if you won the lottery.
            If you leave the test half-way, this site will not keep your record. -->

            Hello everyone, we are the research team of Associate Research Fellow Jun-Cheng Chen
            from the Research Center for Information Technology Innovation at Academia Sinica.
            <br /><br />
            We are currently conducting research on user preference evaluation of AI-generated
            videos. The aim of this study is to assess the temporal consistency and refocusing
            accuracy of video generation and editing methods.
            <br /><br />
            In each question, you will compare two output videos against the same input video and
            focus-map conditions.
            <br /><br />
            This user study contains {{ totalRound }} questions and usually takes about
            {{ estimatedTime }}. <br /><br />
            Participants who complete the survey accurately can choose to provide their email
            address for a chance to enter a raffle. We will randomly select 1 out of every 20 valid
            survey submissions to win a 100 NT$ 7-11 gift voucher.
          </template>
        </p>

        <div>
          <br />
          <template v-if="lang === 'zh-TW'">
            <a
              href="https://drive.google.com/file/d/1B3iZ2qKKpcfQexR-aIo1ZNPC1SNIT4Mm/view"
              target="_blank"
              >中央研究院人文社會科學研究對象說明同意書</a
            >
          </template>
          <template v-else>
            <a
              href="https://drive.google.com/file/d/1B3iZ2qKKpcfQexR-aIo1ZNPC1SNIT4Mm/view"
              target="_blank"
              >Academia Sinica Humanities and Social Sciences Research Subject Description and
              Consent Form</a
            >
          </template>
          <br />
          <input type="checkbox" id="checkbox" v-model="checked" />
          <template v-if="lang === 'zh-TW'"> 我已閱讀並同意 </template>
          <template v-else> I have read and agree </template>
        </div>

        <label for="email" class="green">
          <br />
          <template v-if="lang === 'zh-TW'">
            <!-- 請輸入您的 email 來開始作答 -->
            抽獎用信箱（選填，僅為抽獎通知用）
          </template>
          <template v-else>
            <!-- Please enter your email to get started. -->
            E-mail (optional, only for the lottery notification)
          </template>
        </label>

        <div>
          <input type="email" id="email" v-model="email" v-on:keyup.enter="start" />
          <button @click="start">
            <template v-if="lang === 'zh-TW'"> 下一步 </template>
            <template v-else> Next </template>
          </button>
        </div>
        <p class="green" for="" v-if="showWarning">
          <template v-if="lang === 'en-US'">
            Please confirm that you've read the consent form and agree
          </template>
          <template v-else> 請確認已閱讀以上同意書並同意 </template>
        </p>
      </div>
      <div v-else-if="showExample" class="example-intro">
        <div class="lang-selector">
          <span :class="{ green: lang === 'zh-TW' }" @click="lang = 'zh-TW'">中文</span> /
          <span :class="{ green: lang !== 'zh-TW' }" @click="lang = 'en-US'">Eng</span>
        </div>
        <h1 class="green">RAV</h1>
        <h2>
          <template v-if="lang === 'zh-TW'"> 作答範例 </template>
          <template v-else> Example </template>
        </h2>
        <div class="example-copy">
          <template v-if="lang === 'zh-TW'">
            <p>重新對焦是指根據新的目標焦平面，改變影片中應該清楚的區域。</p>
            <ul class="example-list">
              <li>輸入影片：顯示原始場景。</li>
              <li>輸入 focus map：顯示原本的對焦狀態。</li>
              <li>目標 focus map：顯示希望輸出影片對焦的位置。</li>
              <li>
                Focus map 中，<span class="highlight-green">白色表示應該對焦的位置</span
                >，<span class="highlight-red">黑色表示不應該對焦的位置</span>。
              </li>
            </ul>
            <p>
              在正式題目中，請把 Video A 與 Video B 和目標 focus map
              比較。好的輸出應該把清楚的焦點移到目標區域，並且在時間上穩定，不應出現明顯閃爍、亮度跳動或內容抖動。
            </p>
            <p>每題會有兩個問題：</p>
            <ul class="example-list">
              <li>
                <span class="highlight-blue">時間與內容穩定性</span
                >：請判斷哪個影片比較少閃爍、亮度跳動或內容不穩定。
              </li>
              <li>
                <span class="highlight-blue">空間對焦準確度</span
                >：請判斷哪個影片比較準確地依照目標 focus map 對焦到正確區域。
              </li>
            </ul>
            <p>每題會有三個選項：</p>
            <ul class="example-list">
              <li>如果 Video A 較好，請選 A。</li>
              <li>如果 Video B 較好，請選 B。</li>
              <li>如果兩者很接近、都不好，或很難判斷，請選 C。</li>
            </ul>
            <div class="green example-answer">
              <p>
                在這個範例中，input video 原先對焦在前景的男人，因此未對焦的背景（海洋、天空、山崖）會呈現模糊。你可以和輸入
                focus map 交叉比對：前景的男人與船體呈白灰色，背景則呈黑色。
              </p>
              <p>
                現在我們希望把焦點重新移到畫面的背景，因此目標 focus map
                會變成背景為白色、前景為黑灰色。觀察輸出結果時，Video A 的背景比 Video B
                更清晰，因此「空間對焦準確度」的標準答案是 A。
              </p>
              <p>
                此外，Video B 在時間軸上較不穩定，會出現閃爍、亮度跳動或內容不穩定；Video A
                則較穩定。因此「時間與內容穩定性」的標準答案也是 A。
              </p>
            </div>
          </template>
          <template v-else>
            <p>
              Refocusing means changing which region or focal plane should appear sharp in the
              output video.
            </p>
            <ul class="example-list">
              <li>The input video shows the original scene.</li>
              <li>The input focus map shows the original focus condition.</li>
              <li>The target focus map shows where the output should focus.</li>
              <li>
                In a focus map,
                <span class="highlight-green">
                  white indicates the focus point or focused region</span
                >, while
                <span class="highlight-red">
                  black indicates regions that should not be in focus</span
                >.
              </li>
            </ul>
            <p>
              In the formal questions, compare Video A and Video B against the target focus map. A
              good output should move the sharp focus to the target region while staying temporally
              stable, without obvious flickering, brightness jumps, or content jitter.
            </p>
            <p> Each trial has two questions. </p>
            <ul class="example-list">
              <li>
                <span class="highlight-blue">Temporal consistency</span>: which video has less
                flickering, brightness jumping, or unstable content?
              </li>
              <li>
                <span class="highlight-blue">Spatial refocusing accuracy</span>: which video better
                focuses on the correct region according to the target focus map?
              </li>
            </ul>
            <p>Each question has three options: </p>
            <ul class="example-list">
              <li>Choose A if Video A is better.</li>
              <li>Choose B if Video B is better.</li>
              <li>
                Choose C if the difference is too small, both videos are poor, or it is difficult to
                judge.
              </li>
            </ul>
            <div class="green example-answer">
              <p>
                In this example, the input video is originally focused on the man in the
                foreground, so the out-of-focus background, including the ocean, sky, and cliff,
                appears blurred. You can cross-check this with the input focus map: the foreground
                man and boat appear white or gray, while the background appears black.
              </p>
              <p>
                Now we want to refocus the video onto the background. In the target focus map, the
                background becomes white, while the foreground becomes dark gray or black. In the
                outputs, the background in Video A is clearer than in Video B, so the standard
                answer for spatial refocusing accuracy is A.
              </p>
              <p>
                Also, Video B is less stable over time, with flickering, brightness changes, or
                unstable content, while Video A remains more stable. Therefore, the standard answer
                for temporal consistency is also A.
              </p>
            </div>
          </template>
        </div>
        <p class="green">
          <template v-if="lang === 'zh-TW'">
            請參考右側範例答案，理解判斷標準後開始正式作答。
          </template>
          <template v-else>
            Review the example answers on the right, then start the formal study.
          </template>
        </p>
      </div>
      <div v-else>
        <header>
          <div id="header-div">
            <div class="lang-selector">
              <span :class="{ green: lang === 'zh-TW' }" @click="lang = 'zh-TW'">中文</span> /
              <span :class="{ green: lang !== 'zh-TW' }" @click="lang = 'en-US'">Eng</span>
            </div>
            <h1 class="green">RAV</h1>
            <h3>
              <template v-if="lang === 'zh-TW'"> 問題 </template>
              <template v-else> Question </template>
              {{ round }}/{{ totalRound }}
            </h3>
          </div>
        </header>

        <p class="instruction">
          <template v-if="lang === 'zh-TW'">
            請根據輸入影片與 focus map，比較影片 A 與影片 B，並回答兩個問題。
          </template>
          <template v-else>
            Compare video A and video B using the input video and focus maps, then answer both
            questions.
          </template>
        </p>
        <div class="formal-reminder">
          <template v-if="lang === 'zh-TW'">
            <h3>判斷提醒</h3>
            <p>重新對焦是指根據新的目標焦平面，改變影片中應該清楚的區域。</p>
            <ul class="example-list">
              <li>輸入影片：顯示原始場景。</li>
              <li>輸入 focus map：顯示原本的對焦狀態。</li>
              <li>目標 focus map：顯示希望輸出影片對焦的位置。</li>
              <li>
                Focus map 中，<span class="highlight-green">白色表示應該對焦的位置</span
                >，<span class="highlight-red">黑色表示不應該對焦的位置</span>。
              </li>
            </ul>
            <p>
              判斷時，請把 Video A 與 Video B 和目標 focus map
              比較。好的輸出應該把清楚的焦點移到目標區域，並且在時間上穩定，不應出現明顯閃爍、亮度跳動或內容抖動。
            </p>
            <p>每題會有兩個問題：</p>
            <ul class="example-list">
              <li>
                <span class="highlight-blue">時間與內容穩定性</span
                >：請判斷哪個影片比較少閃爍、亮度跳動或內容不穩定。
              </li>
              <li>
                <span class="highlight-blue">空間對焦準確度</span
                >：請判斷哪個影片比較準確地依照目標 focus map 對焦到正確區域。
              </li>
            </ul>
            <p>每題會有三個選項：</p>
            <ul class="example-list">
              <li>如果 Video A 較好，請選 A。</li>
              <li>如果 Video B 較好，請選 B。</li>
              <li>如果兩者很接近、都不好，或很難判斷，請選 C。</li>
            </ul>
          </template>
          <template v-else>
            <h3>Rating reminder</h3>
            <p>
              Refocusing means changing which region or focal plane should appear sharp in the
              output video.
            </p>
            <ul class="example-list">
              <li>The input video shows the original scene.</li>
              <li>The input focus map shows the original focus condition.</li>
              <li>The target focus map shows where the output should focus.</li>
              <li>
                In a focus map,
                <span class="highlight-green">
                  white indicates the focus point or focused region</span
                >, while
                <span class="highlight-red">
                  black indicates regions that should not be in focus</span
                >.
              </li>
            </ul>
            <p>
              Compare Video A and Video B against the target focus map. A good output should move
              the sharp focus to the target region while staying temporally stable, without obvious
              flickering, brightness jumps, or content jitter.
            </p>
            <p>Each trial has two questions.</p>
            <ul class="example-list">
              <li>
                <span class="highlight-blue">Temporal consistency</span>: which video has less
                flickering, brightness jumping, or unstable content?
              </li>
              <li>
                <span class="highlight-blue">Spatial refocusing accuracy</span>: which video better
                focuses on the correct region according to the target focus map?
              </li>
            </ul>
            <p>Each question has three options:</p>
            <ul class="example-list">
              <li>Choose A if Video A is better.</li>
              <li>Choose B if Video B is better.</li>
              <li>
                Choose C if the difference is too small, both videos are poor, or it is difficult to
                judge.
              </li>
            </ul>
          </template>
        </div>
      </div>
    </div>
    <div v-if="round === null && !showExample">
      <br />
      <div class="scrollable">
        <template v-if="lang === 'zh-TW'">
          <h3>研究參與資訊</h3>
          <ul>
            <li>
              資料收集：我們計劃收集100-200名受試者的生成影像或影片評分資料。
            </li>
            <li>遞補機制：若有人中途退出或數據無法使用，將遞補至收集完所需受試者為止。</li>
            <li>目標對象：已滿 18 歲且未受監護宣告之成年人。</li>
            <li>經費來源：中研院資創中心智慧優網。</li>
          </ul>
          <br />
          <h3>資料管理與隱私保障</h3>
          <ul>
            <li>資料用途：本問卷內容僅供學術研究之用。</li>
            <li>
              聯絡資訊：您的 email 等聯絡資訊僅供 7-11 禮券抽獎使用，不會留下任何身份等敏感資訊。
            </li>
            <li>資料刪除：所有聯絡資料將於問卷調查與抽獎完畢後立即刪除。</li>
            <li>
              退出與刪除權利：您可在填寫問卷時隨時退出，填寫完成後也可來信 (<a
                href="nickchiu@citi.sinica.edu.tw"
                >nickchiu@citi.sinica.edu.tw</a
              >
              或 <a href="pullpull@citi.sinica.edu.tw">pullpull@citi.sinica.edu.tw</a>)
              要求刪除相關資料。
            </li>
          </ul>
          <br />
          <h3>資料管理細節</h3>
          <ul>
            <li>
              不收集敏感資料：本計畫實驗不收集任何關於您的個人敏感資料，僅收集生成影像或影片的評分。
            </li>
            <li>
              資料儲存與加密：實驗收集的資料將儲存於實驗地點（中央研究院資訊創新研究中心）的資料處理電腦，並會進一步加密處理。除了評分結果與抽獎用
              email，研究人員分析資料時不會取得您的身分資料。
            </li>
            <li>資料使用限制：所收集的資料僅供本實驗室人員學術研究使用。</li>
            <li>抽獎後刪除：email 聯絡資訊將於抽獎完成後（問卷結束後一至兩週內進行）立即刪除。</li>
            <li>
              資料庫管理：此計畫所收集的資料庫（生成影像或影片評分結果）為不記名資料庫，放置在主持人實驗室伺服器，資料保存和使用時間至計畫結束（2026/12/31）。若被挪作其他用途，本計畫團隊將追溯非法使用來源，以確保您的權益。計畫結束後，實驗室將銷毀相關問卷評量結果。
            </li>
          </ul>
          <br />
          <h3>參與權益聲明</h3>
          <ul>
            <li>研究計畫名稱：AI 生成影像與影片之使用者偏好評量</li>
            <li>編號：ASIRB-HS-24031</li>
            <li>自願參與：參加此項計畫是完全自願性質，無論參加與否均不影響您的權益。</li>
            <li>
              權益諮詢：若對參與研究的相關權益有疑問，可來電查詢中央研究院人文社會科學研究倫理委員會
              IRB on Humanities & Social Science Research / IRB-HS，查詢電話：02-27898722。
            </li>
          </ul>
          <br />
          <h3>實驗進行方式</h3>
          您將對預先使用不同方法產生的影片進行個人主觀的品質比較，並評估其時間與內容穩定性與是否正確對焦到目標
          focus map 所指定的區域。您將進行 {{ totalRound }} 組實驗，大約花費 10 至 15
          分鐘，但您可隨時依自己的狀態停止實驗。
        </template>
        <template v-else>
          <h3>Research Participation Information</h3>
          <ul>
            <li>
              Data Collection: We will collect image or video evaluation data from 100-200
              participants. If any participant withdraws or their data becomes unusable, we will
              recruit additional participants until the required number is reached.
            </li>
            <li>Target Subject: Adults aged 18 and above who are not under guardianship.</li>
            <li>
              Funding Source: Intelligent Networks Program at the Research Center for Information
              Technology Innovation, Academia Sinica
            </li>
          </ul>
          <br />
          <h3>Data Management and Privacy Protection</h3>
          <ul>
            <li>
              Data Usage: The content of this questionnaire is solely for academic research
              purposes.
            </li>
            <li>
              Contact Information: Your email and other contact information will only be used for a
              7-11 gift voucher lottery and will not be used to record any personal identification
              or sensitive information.
            </li>
            <li>
              Data Deletion: All contact information will be deleted immediately after the
              questionnaire survey and lottery are completed.
            </li>
            <li>
              Withdrawal and Deletion Rights: You can withdraw from the questionnaire at any time
              during the process. After completion, you can also request the deletion of relevant
              data by sending an email to
              <a href="nickchiu@citi.sinica.edu.tw">nickchiu@citi.sinica.edu.tw</a> or
              <a href="pullpull@citi.sinica.edu.tw">pullpull@citi.sinica.edu.tw</a>.
            </li>
          </ul>
          <br />
          <h3>Details of Data Management</h3>
          <ul>
            <li>
              No Collection of Sensitive Data: This project does not collect any of your personal
              sensitive data, only evaluations of the generated images or videos.
            </li>
            <li>
              Data Storage and Encryption: Collected data will be stored on the data processing
              computers at the research site (Academia Sinica Research Center for Information
              Technology Innovation) and will be further encrypted. Researchers analyzing the data
              will not have access to your identity information, except for the evaluation results
              and the email used for the lottery.
            </li>
            <li>
              Data Usage Restrictions: The collected data will only be used for academic research by
              the personnel of this laboratory.
            </li>
            <li>
              Deletion After Lottery: Email contact information will be deleted immediately after
              the lottery is completed (within one to two weeks after the questionnaire ends).
            </li>
            <li>
              Database Management: The database collected in this project (evaluation results of
              generated images or videos) is anonymous and will be stored on the principal
              investigator's laboratory server. Data storage and usage will continue until the end
              of the project (2026/12/31). If the data is used for any other purpose, our team will
              track the source of unauthorized use to protect your rights. After the project
              concludes, the laboratory will destroy the relevant questionnaire evaluation results.
            </li>
          </ul>
          <br />
          <h3>Statement of Participation Rights</h3>
          <ul>
            <li>
              Voluntary Participation: Participation in this project is entirely voluntary, and your
              decision to participate or not will not affect your rights in any way.
            </li>
            <li>ID: ASIRB-HS-24031</li>
            <li>
              Voluntary Participation: Participation in this project is completely voluntary, and
              your rights will not be affected whether you choose to participate or not.
            </li>
            <li>
              Inquiry of Rights: If you have any questions about your rights related to research
              participation, please call the Academia Sinica Institutional Review Board on
              Humanities & Social Science Research (IRB-HS) at 02-27898722.
            </li>
          </ul>
          <br />
          <h3>Experimental Procedure</h3>
          You will be asked to compare pairs of generated videos. For each trial, you will see one
          input video, two output videos, one input focus map, and one target focus map. You will
          answer which output video is better for temporal consistency and which output video is
          better for spatial refocusing accuracy. The study contains {{ totalRound }} questions and
          takes about {{ estimatedTime }}. You can stop the experiment at any time based on your own
          condition.
        </template>
      </div>
    </div>
    <div v-else-if="showExample" class="study-panel example-panel">
      <div class="example-note">
        <template v-if="lang === 'zh-TW'">
          正式題目不會顯示方法名稱，只會顯示 Video A 與 Video B。請根據你看到的結果作答。
        </template>
        <template v-else>
          Formal questions will only show Video A and Video B, not method names. Please answer based
          on what you observe.
        </template>
      </div>
      <Rater
        :trial="exampleTrial"
        :lang="lang"
        :is-example="true"
        :readonly-example="true"
        @submit="beginStudy"
      />
    </div>
    <div v-else class="study-panel">
      <Rater :trial="trial" :lang="lang" @submit="submit" />
    </div>
  </template>
</template>

<style scoped>
.lang-selector {
  float: right;
  /*
  position: fixed;
  top: 10px;
  right: 20px;
  z-index: 999;
*/
}

.lang-selector span:hover {
  cursor: pointer;
}

input {
  border: 1px solid #666;
  height: 25px;
  border-radius: 5px;
  margin: 0 5px 0 0;
  font-size: 16px;
}

button {
  height: 25px;
}

p {
  margin: 10px 0 10px 0;
}

.example-copy p {
  margin-bottom: 14px;
}

.example-list {
  margin: 4px 0 14px 0;
  padding-left: 1.25rem;
}

.example-list li {
  margin-bottom: 6px;
}

.highlight-green {
  color: #148a43;
  font-weight: 600;
}

.highlight-red {
  color: #c43c35;
  font-weight: 600;
}

.highlight-blue {
  color: #2563eb;
  font-weight: 600;
}

.formal-reminder {
  border-top: 1px solid var(--color-border);
  margin-top: 16px;
  padding-top: 14px;
}

.formal-reminder h3 {
  color: var(--color-heading);
  margin-bottom: 8px;
}

.example-panel {
  width: 100%;
}

.example-note {
  border-top: 1px solid var(--color-border);
  margin-bottom: 12px;
  padding-top: 12px;
}

@media (min-width: 1024px) {
  .main {
    padding-right: 2rem;
  }
  .scrollable {
    width: 500px;
    height: 430px;
    overflow-y: auto; /* Vertical scrollbar */
    overflow-x: hidden; /* Hide horizontal scrollbar if not needed */
  }
}

@media (max-width: 1023px) {
  header {
    position: fixed;
    top: 0;
    right: 0;
    width: 100%;
    background: var(--color-background-soft);
    z-index: 1;
    display: flex;
    place-items: center;
  }

  header > div {
    padding: 0 2rem 0.5rem 2rem;
    margin: 0 auto;
  }

  p.instruction {
    margin-top: 150px;
  }
}
</style>
