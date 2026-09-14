<script setup>
import { computed, inject, onBeforeUnmount, onMounted, ref, watch } from 'vue'
import * as Phaser from 'phaser'
import gardenBackgroundUrl from './assets/background/garden_bg.webp'
import leavesUrl from './assets/background/leafs.webp'
import flowersUrl from './assets/background/flowers.webp'
import butterfliesUrl from './assets/background/butterflies.webp'
import answerPanelUrl from './assets/ui/answer_panel.webp'
import gameLogoUrl from './assets/ui/game_logo.webp'
import mascotCorrectUrl from './assets/ui/mascot_correct.webp'
import mascotIdleUrl from './assets/ui/mascot_idle.webp'
import mascotWrongUrl from './assets/ui/mascot_wrong.webp'
import progressBarUrl from './assets/ui/progress_bar.webp'
import progressFlowerUrl from './assets/ui/progress_flower.webp'
import questionBoardUrl from './assets/ui/question_board.webp'

const props = defineProps({
  engine: { type: Object, required: true },
  params: { type: Object, required: true },
  pool: { type: Array, default: null },
})

const live = inject('liveSettings', { sizeScale: 1, elementCount: null, speed: 100 })

const DESIGN_WIDTH = 1672
const DESIGN_HEIGHT = 941

const LEAF_WIND_FRAGMENT_SHADER = `
#ifdef GL_FRAGMENT_PRECISION_HIGH
precision highp float;
#else
precision mediump float;
#endif

varying vec2 outTexCoord;
uniform sampler2D uLeaves;
uniform float uTime;

float softEllipse(vec2 uv, vec2 center, vec2 radius) {
  float distanceFromCenter = length((uv - center) / radius);
  return 1.0 - smoothstep(0.68, 1.0, distanceFromCenter);
}

void main() {
  vec2 uv = outTexCoord;
  float windTime = uTime * 1.22;

  float leftOuter = softEllipse(uv, vec2(0.075, 0.82), vec2(0.17, 0.22));
  float leftMiddle = softEllipse(uv, vec2(0.23, 0.875), vec2(0.19, 0.19));
  float leftInner = softEllipse(uv, vec2(0.385, 0.835), vec2(0.135, 0.19));
  float rightInner = softEllipse(uv, vec2(0.625, 0.835), vec2(0.135, 0.19));
  float rightMiddle = softEllipse(uv, vec2(0.78, 0.875), vec2(0.19, 0.19));
  float rightOuter = softEllipse(uv, vec2(0.935, 0.82), vec2(0.17, 0.22));

  float leftMask = max(leftOuter, max(leftMiddle, leftInner));
  float rightMask = max(rightOuter, max(rightMiddle, rightInner));
  float canopyMask = max(leftMask, rightMask);

  float leftWeight = leftOuter + leftMiddle + leftInner + 0.001;
  float rightWeight = rightOuter + rightMiddle + rightInner + 0.001;
  float leftResponse =
    (leftOuter * sin(windTime * 0.63 + 0.1) +
     leftMiddle * sin(windTime * 0.57 + 0.58) +
     leftInner * sin(windTime * 0.51 + 0.96)) / leftWeight;
  float rightResponse =
    (rightInner * sin(windTime * 0.49 + 1.42) +
     rightMiddle * sin(windTime * 0.55 + 1.02) +
     rightOuter * sin(windTime * 0.67 + 0.54)) / rightWeight;

  float globalWind =
    sin(windTime * 0.39) * 0.62 +
    sin(windTime * 0.17 + 1.9) * 0.25 +
    sin(windTime * 0.91 + 0.35) * 0.13;
  float gustWave = max(0.0, sin(windTime * 0.12 - 0.7));
  float gust = pow(gustWave, 8.0);

  float localResponse = leftResponse * leftMask + rightResponse * rightMask;
  float hangingLeafFade = smoothstep(0.64, 0.73, uv.y);
  float leftAttachment = softEllipse(uv, vec2(0.015, 0.94), vec2(0.09, 0.12));
  float rightAttachment = softEllipse(uv, vec2(0.985, 0.94), vec2(0.09, 0.12));
  float attachmentFlex = 1.0 - max(leftAttachment, rightAttachment) * 0.82;

  float outerFlutterMask = max(leftOuter, rightOuter);
  float middleMask = max(leftMiddle, rightMiddle);
  float innerMask = max(leftInner, rightInner);
  float motionStrength = max(
    outerFlutterMask * 1.2,
    max(middleMask * 0.92, innerMask * 0.62)
  );
  float fineFlutter =
    sin(uv.x * 31.0 + uv.y * 17.0 + windTime * 2.15) *
    outerFlutterMask * 0.00058;

  float horizontalOffset =
    ((globalWind * 0.0041 + gust * 0.0023) * motionStrength +
     localResponse * 0.00175 * canopyMask) *
    hangingLeafFade * attachmentFlex +
    fineFlutter;

  vec2 sampleUv = vec2(clamp(uv.x - horizontalOffset, 0.0, 1.0), uv.y);
  gl_FragColor = texture2D(uLeaves, sampleUv);
}
`

const FLOWER_WIND_FRAGMENT_SHADER = `
#ifdef GL_FRAGMENT_PRECISION_HIGH
precision highp float;
#else
precision mediump float;
#endif

varying vec2 outTexCoord;
uniform sampler2D uFlowers;
uniform float uTime;

const float ASPECT = 1.776833;

float softEllipse(vec2 uv, vec2 center, vec2 radius) {
  float distanceFromCenter = length((uv - center) / radius);
  return 1.0 - smoothstep(0.7, 1.0, distanceFromCenter);
}

vec2 rotateAroundPivot(vec2 uv, vec2 pivot, float angle) {
  vec2 point = (uv - pivot) * vec2(ASPECT, 1.0);
  float cosine = cos(-angle);
  float sine = sin(-angle);
  point = mat2(cosine, -sine, sine, cosine) * point;
  return clamp(pivot + point / vec2(ASPECT, 1.0), 0.0, 1.0);
}

void main() {
  vec2 uv = outTexCoord;
  vec4 baseColor = texture2D(uFlowers, uv);

  float rootFlex = smoothstep(0.045, 0.145, uv.y);
  float topFade = 1.0 - smoothstep(0.31, 0.39, uv.y);
  float leftStone = softEllipse(uv, vec2(0.225, 0.16), vec2(0.048, 0.062));
  float movable = rootFlex * topFade * (1.0 - leftStone);

  float w1 = softEllipse(uv, vec2(0.085, 0.155), vec2(0.085, 0.13)) * movable;
  float w2 = softEllipse(uv, vec2(0.19, 0.17), vec2(0.105, 0.145)) * movable;
  float w3 = softEllipse(uv, vec2(0.305, 0.15), vec2(0.095, 0.125)) * movable;
  float w4 = softEllipse(uv, vec2(0.695, 0.15), vec2(0.095, 0.125)) * movable;
  float w5 = softEllipse(uv, vec2(0.81, 0.17), vec2(0.105, 0.145)) * movable;
  float w6 = softEllipse(uv, vec2(0.92, 0.155), vec2(0.085, 0.13)) * movable;

  float sharedWind =
    sin(uTime * 0.62) * 0.035 +
    sin(uTime * 0.24 + 1.3) * 0.012;
  float gustWave = max(0.0, sin(uTime * 0.15 - 0.9));
  float gust = pow(gustWave, 8.0) * 0.018;

  float a1 = sharedWind * 1.12 + sin(uTime * 0.91 + 0.1) * 0.013 + gust;
  float a2 = sharedWind * 0.84 + sin(uTime * 0.78 + 0.7) * 0.01 + gust * 0.8;
  float a3 = sharedWind * 0.66 + sin(uTime * 0.69 + 1.15) * 0.008 + gust * 0.65;
  float a4 = sharedWind * 0.64 + sin(uTime * 0.71 + 1.65) * 0.008 + gust * 0.62;
  float a5 = sharedWind * 0.88 + sin(uTime * 0.8 + 2.05) * 0.01 + gust * 0.82;
  float a6 = sharedWind * 1.1 + sin(uTime * 0.94 + 2.5) * 0.013 + gust;

  float totalWeight = w1 + w2 + w3 + w4 + w5 + w6;
  float normalizer = max(1.0, totalWeight);
  w1 /= normalizer;
  w2 /= normalizer;
  w3 /= normalizer;
  w4 /= normalizer;
  w5 /= normalizer;
  w6 /= normalizer;
  float animatedWeight = min(1.0, totalWeight);

  vec2 warpedUv = uv;
  warpedUv += (rotateAroundPivot(uv, vec2(0.085, 0.055), a1) - uv) * w1;
  warpedUv += (rotateAroundPivot(uv, vec2(0.19, 0.055), a2) - uv) * w2;
  warpedUv += (rotateAroundPivot(uv, vec2(0.305, 0.055), a3) - uv) * w3;
  warpedUv += (rotateAroundPivot(uv, vec2(0.695, 0.055), a4) - uv) * w4;
  warpedUv += (rotateAroundPivot(uv, vec2(0.81, 0.055), a5) - uv) * w5;
  warpedUv += (rotateAroundPivot(uv, vec2(0.92, 0.055), a6) - uv) * w6;

  vec2 sampleUv = mix(uv, warpedUv, animatedWeight);
  gl_FragColor = texture2D(uFlowers, sampleUv);
}
`

const gameContainer = ref(null)
const selectedAnswer = ref(null)
const revealedCorrect = ref(null)
const currentQuestionIndex = ref(0)
const results = ref(Array(5).fill(null))
const mascotState = ref('idle')
const isTransitioning = ref(false)
let game = null
let transitionTimer = null

const questionSets = {
  kolay: [
    { type: 'ad', prompt: 'Bu şeklin adı nedir?', shape: 'square', correct: 'Kare', answers: ['Üçgen', 'Kare', 'Dikdörtgen'] },
    { type: 'ad', prompt: 'Bu şeklin adı nedir?', shape: 'triangle', correct: 'Üçgen', answers: ['Daire', 'Dikdörtgen', 'Üçgen'] },
    { type: 'ad', prompt: 'Bu şeklin adı nedir?', shape: 'rectangle', correct: 'Dikdörtgen', answers: ['Kare', 'Dikdörtgen', 'Daire'] },
    { type: 'ad', prompt: 'Bu şeklin adı nedir?', shape: 'circle', correct: 'Daire', answers: ['Üçgen', 'Daire', 'Kare'] },
    { type: 'ad', prompt: 'Bu şeklin adı nedir?', shape: 'diamond', correct: 'Eşkenar Dörtgen', answers: ['Dikdörtgen', 'Kare', 'Eşkenar Dörtgen'] },
  ],
  orta: [
    { type: 'kose', prompt: 'Bu şeklin kaç köşesi vardır?', shape: 'triangle', correct: '3', answers: ['4', '0', '3'] },
    { type: 'eslestirme', prompt: 'Hangisi aynı sayıda köşeye sahiptir?', shape: 'square', correct: 'Dikdörtgen', answers: ['Üçgen', 'Daire', 'Dikdörtgen'] },
    { type: 'kose', prompt: 'Bu şeklin kaç köşesi vardır?', shape: 'circle', correct: '0', answers: ['3', '0', '4'] },
    { type: 'eslestirme', prompt: 'Hangisi aynı sayıda köşeye sahiptir?', shape: 'diamond', correct: 'Kare', answers: ['Daire', 'Üçgen', 'Kare'] },
    { type: 'kose', prompt: 'Bu şeklin kaç köşesi vardır?', shape: 'rectangle', correct: '4', answers: ['0', '3', '4'] },
  ],
  zor: [
    { type: 'oruntu', prompt: 'Örüntüde sırada hangi şekil var?', pattern: ['square', 'triangle', 'square', 'triangle'], correct: 'Kare', answers: ['Daire', 'Kare', 'Üçgen'] },
    { type: 'oruntu', prompt: 'Eksik şekli bul.', pattern: ['circle', 'circle', 'diamond', 'circle', 'circle'], correct: 'Eşkenar Dörtgen', answers: ['Kare', 'Eşkenar Dörtgen', 'Üçgen'] },
    { type: 'oruntu', prompt: 'Örüntüyü tamamla.', pattern: ['triangle', 'square', 'circle', 'triangle', 'square'], correct: 'Daire', answers: ['Daire', 'Dikdörtgen', 'Üçgen'] },
    { type: 'oruntu', prompt: 'Sıradaki şekli seç.', pattern: ['rectangle', 'diamond', 'rectangle', 'diamond'], correct: 'Dikdörtgen', answers: ['Üçgen', 'Kare', 'Dikdörtgen'] },
    { type: 'oruntu', prompt: 'Eksik şekli bul.', pattern: ['square', 'square', 'triangle', 'square', 'square'], correct: 'Üçgen', answers: ['Daire', 'Üçgen', 'Kare'] },
  ],
}

const activeLevel = computed(() => (
  ['kolay', 'orta', 'zor'].includes(props.params.level) ? props.params.level : 'kolay'
))
const availableQuestions = computed(() => questionSets[activeLevel.value])
const currentQuestion = computed(() => availableQuestions.value[currentQuestionIndex.value])
const flowerSlots = ['40.8%', '45.75%', '50.7%', '55.65%', '60.6%']
const liveSpeed = computed(() => {
  const value = Number(live.speed)
  return Number.isFinite(value) ? Math.min(200, Math.max(50, value)) : 100
})
const feedbackDuration = computed(() => {
  const configured = Number(props.params.feedbackDurationMs)
  const base = Number.isFinite(configured) ? Math.min(4000, Math.max(500, configured)) : 1900
  return Math.round(base * (100 / liveSpeed.value))
})
const gameStyle = computed(() => ({
  '--game-font-scale': String(Number(live.sizeScale) || 1),
}))

const mascotUrl = computed(() => {
  if (mascotState.value === 'correct') return mascotCorrectUrl
  if (mascotState.value === 'wrong') return mascotWrongUrl
  return mascotIdleUrl
})

const chooseAnswer = (answer) => {
  if (isTransitioning.value || props.engine.finished.value) return

  const isCorrect = answer === currentQuestion.value.correct
  selectedAnswer.value = answer
  const progressIndex = Math.min(Math.max(props.engine.round.value - 1, 0), results.value.length - 1)
  results.value[progressIndex] = isCorrect
  mascotState.value = isCorrect ? 'correct' : 'wrong'
  isTransitioning.value = true

  if (!isCorrect) revealedCorrect.value = currentQuestion.value.correct

  transitionTimer = window.setTimeout(() => {
    props.engine.answer(isCorrect, {
      tip: isCorrect ? 'dogru' : 'yanlis',
      hedef: currentQuestion.value.correct,
      secilen: answer,
      sekil: currentQuestion.value.shape,
      soruTuru: currentQuestion.value.type,
      oruntu: currentQuestion.value.pattern,
    })
  }, feedbackDuration.value)
}

watch(
  [() => props.engine.round.value, availableQuestions],
  ([round]) => {
    if (round <= 0 || props.engine.finished.value) return
    if (transitionTimer) window.clearTimeout(transitionTimer)
    transitionTimer = null
    currentQuestionIndex.value = (round - 1) % availableQuestions.value.length
    if (round === 1) results.value = Array(5).fill(null)
    selectedAnswer.value = null
    revealedCorrect.value = null
    mascotState.value = 'idle'
    isTransitioning.value = false
  },
  { immediate: true },
)

class GardenScene extends Phaser.Scene {
  constructor() {
    super('GardenScene')
  }

  preload() {
    this.load.image('garden-background', gardenBackgroundUrl)
    this.load.image('leaves', leavesUrl)
    this.load.image('flowers', flowersUrl)
    this.load.image('answer-panel', answerPanelUrl)
    this.load.image('mascot-idle', mascotIdleUrl)
    this.load.image('mascot-correct', mascotCorrectUrl)
    this.load.image('progress-bar', progressBarUrl)
    this.load.image('progress-flower', progressFlowerUrl)
    this.load.image('question-board', questionBoardUrl)
  }

  createGlowTexture() {
    const size = 64
    const center = size / 2
    const texture = this.textures.createCanvas('garden-glow', size, size)
    const context = texture.context
    const glow = context.createRadialGradient(center, center, 0, center, center, center)

    glow.addColorStop(0, 'rgba(255, 255, 225, 1)')
    glow.addColorStop(0.09, 'rgba(255, 247, 145, 0.98)')
    glow.addColorStop(0.28, 'rgba(224, 255, 120, 0.5)')
    glow.addColorStop(1, 'rgba(190, 255, 100, 0)')
    context.fillStyle = glow
    context.fillRect(0, 0, size, size)
    texture.refresh()
  }

  createGardenLights() {
    const anchors = [
      [132, 552], [222, 690], [330, 515], [430, 735], [535, 585],
      [645, 665], [760, 490], [842, 760], [955, 605], [1065, 520],
      [1160, 705], [1275, 570], [1390, 670], [1518, 525], [1580, 755],
      [285, 420], [705, 400], [1010, 445], [1325, 405],
    ]

    return anchors.map(([x, y], index) => {
      const depth = 0.76 + (index % 4) * 0.1
      const glowScale = (0.25 + (index % 3) * 0.055) * depth
      const sprite = this.add
        .image(x, y, 'garden-glow')
        .setBlendMode(Phaser.BlendModes.ADD)
        .setScale(glowScale)

      return {
        sprite,
        baseX: x,
        baseY: y,
        driftX: 9 + (index % 5) * 3,
        driftY: 7 + ((index + 2) % 4) * 3,
        phase: index * 1.73,
        speed: 0.34 + (index % 6) * 0.045,
        pulseSpeed: 1.05 + (index % 5) * 0.19,
        baseAlpha: 0.48 + (index % 4) * 0.075,
        baseScale: glowScale,
      }
    })
  }

  createInterface() {
    this.interfaceLayers = this.backgroundLayers
    this.answerLocked = false

    const progressBar = this.add
      .image(895, 273, 'progress-bar')
      .setDisplaySize(1170, 169)

    const board = this.add
      .image(886, 494, 'question-board')
      .setDisplaySize(635, 386)

    const square = this.add
      .rectangle(886, 540, 126, 126, 0x000000, 0)
      .setStrokeStyle(14, 0x10a9aa, 1)

    this.mascot = this.add
      .image(0, 0, 'mascot-idle')
      .setDisplaySize(301, 339)
    this.mascotContainer = this.add.container(485, 475, [this.mascot])

    this.progressFlower = this.add
      .image(610, 266, 'progress-flower')
      .setDisplaySize(92, 92)
      .setVisible(false)

    this.interfaceLayers.add([
      progressBar,
      board,
      square,
      this.mascotContainer,
      this.progressFlower,
    ])

    const answers = [
      { key: 'A', label: 'Üçgen', x: 604 },
      { key: 'B', label: 'Kare', x: 919, correct: true },
      { key: 'C', label: 'Dikdörtgen', x: 1234 },
    ]

    this.answerButtons = answers.map((answer, index) => {
      const panel = this.add
        .image(0, 0, 'answer-panel')
        .setDisplaySize(289, 151)
        .setInteractive({ useHandCursor: true })
      const label = this.add
        .text(0, 4, answer.label, {
          color: '#5a260f',
          fontFamily: 'Arial Rounded MT Bold, Trebuchet MS, sans-serif',
          fontSize: '32px',
          fontStyle: 'bold',
          stroke: '#fff3d2',
          strokeThickness: 2,
        })
        .setOrigin(0.5)
      const button = this.add.container(answer.x, 711, [panel, label])

      button.setScale(0)
      this.interfaceLayers.add(button)

      this.tweens.add({
        targets: button,
        scale: 1,
        duration: 520,
        delay: 110 + index * 120,
        ease: 'Back.Out',
      })

      panel.on('pointerover', () => {
        if (this.answerLocked) return
        this.tweens.add({
          targets: button,
          scale: 1.07,
          duration: 160,
          ease: 'Sine.Out',
        })
      })
      panel.on('pointerout', () => {
        if (this.answerLocked) return
        this.tweens.add({
          targets: button,
          scale: 1,
          duration: 190,
          ease: 'Sine.Out',
        })
      })
      panel.on('pointerdown', () => {
        if (this.answerLocked) return
        this.tweens.add({
          targets: button,
          scale: 0.93,
          duration: 80,
          yoyo: true,
          ease: 'Quad.Out',
        })
      })
      panel.on('pointerup', () => this.selectAnswer(answer, button, panel))

      return { ...answer, button, panel }
    })
  }

  selectAnswer(answer, button, panel) {
    if (this.answerLocked) return

    if (!answer.correct) {
      panel.setTint(0xff8f8f)
      this.tweens.add({
        targets: button,
        scale: { from: 0.96, to: 1.08 },
        duration: 170,
        yoyo: true,
        ease: 'Back.Out',
      })
      return
    }

    this.answerLocked = true
    panel.setTint(0x8ff0bd)
    this.tweens.add({
      targets: button,
      scale: { from: 0.92, to: 1.16 },
      duration: 260,
      yoyo: true,
      ease: 'Back.Out',
    })

    this.createCorrectBurst(button.x, button.y)
    this.animateProgressFlower()
    this.showCorrectMascot()
  }

  createCorrectBurst(x, y) {
    const colors = [0xffe56b, 0x8ff0bd, 0xffffff]

    for (let index = 0; index < 12; index += 1) {
      const angle = (Math.PI * 2 * index) / 12
      const distance = 74 + (index % 3) * 18
      const sparkle = this.add.circle(x, y, 7 - (index % 3), colors[index % 3])

      this.interfaceLayers.add(sparkle)
      this.tweens.add({
        targets: sparkle,
        x: x + Math.cos(angle) * distance,
        y: y + Math.sin(angle) * distance,
        alpha: 0,
        scale: { from: 0.4, to: 1.35 },
        duration: 620,
        ease: 'Cubic.Out',
        onComplete: () => sparkle.destroy(),
      })
    }
  }

  animateProgressFlower() {
    const targetScale = 92 / this.progressFlower.width

    this.progressFlower
      .setVisible(true)
      .setPosition(886, 540)
      .setAlpha(0)
      .setScale(targetScale * 0.16)

    this.tweens.add({
      targets: this.progressFlower,
      x: 610,
      y: 266,
      alpha: 1,
      scale: targetScale,
      angle: 360,
      duration: 820,
      ease: 'Back.Out',
    })
  }

  showCorrectMascot() {
    this.tweens.add({
      targets: this.mascotContainer,
      alpha: 0,
      scale: 0.94,
      duration: 170,
      ease: 'Sine.In',
      onComplete: () => {
        this.mascot.setTexture('mascot-correct').setDisplaySize(301, 339)
        this.tweens.add({
          targets: this.mascotContainer,
          alpha: 1,
          scale: 1,
          duration: 260,
          ease: 'Back.Out',
        })

        this.time.delayedCall(1150, () => {
          this.tweens.add({
            targets: this.mascotContainer,
            alpha: 0,
            duration: 160,
            onComplete: () => {
              this.mascot.setTexture('mascot-idle').setDisplaySize(301, 339)
              this.tweens.add({
                targets: this.mascotContainer,
                alpha: 1,
                duration: 240,
                ease: 'Sine.Out',
              })
            },
          })
        })
      },
    })
  }

  create() {
    this.backgroundLayers = this.add.container(0, 0)

    this.createGlowTexture()

    const gardenBackground = this.add.image(
      DESIGN_WIDTH / 2,
      DESIGN_HEIGHT / 2,
      'garden-background',
    )
    const leaves = this.add.shader(
      {
        name: 'LeafWind',
        fragmentSource: LEAF_WIND_FRAGMENT_SHADER,
        setupUniforms: (setUniform) => {
          setUniform('uLeaves', 0)
          setUniform('uTime', this.game.loop.time / 1000)
        },
      },
      DESIGN_WIDTH / 2,
      DESIGN_HEIGHT / 2,
      DESIGN_WIDTH,
      DESIGN_HEIGHT,
      ['leaves'],
    )
    const flowers = this.add.shader(
      {
        name: 'FlowerWind',
        fragmentSource: FLOWER_WIND_FRAGMENT_SHADER,
        setupUniforms: (setUniform) => {
          setUniform('uFlowers', 0)
          setUniform('uTime', this.game.loop.time / 1000)
        },
      },
      DESIGN_WIDTH / 2,
      DESIGN_HEIGHT / 2,
      DESIGN_WIDTH,
      DESIGN_HEIGHT,
      ['flowers'],
    )
    this.gardenLights = this.createGardenLights()
    this.backgroundLayers.add([
      gardenBackground,
      leaves,
      flowers,
      ...this.gardenLights.map((light) => light.sprite),
    ])

    this.resizeBackground(this.scale.gameSize)
    this.scale.on('resize', this.resizeBackground, this)
    this.events.once(Phaser.Scenes.Events.SHUTDOWN, () => {
      this.scale.off('resize', this.resizeBackground, this)
    })
  }

  update(time) {
    const elapsed = time / 1000

    this.gardenLights?.forEach((light) => {
      const driftTime = elapsed * light.speed + light.phase
      const pulse = 0.5 + Math.sin(elapsed * light.pulseSpeed + light.phase) * 0.5
      const sparkle = Math.pow(
        Math.max(0, Math.sin(elapsed * light.pulseSpeed * 0.47 + light.phase * 1.7)),
        10,
      )

      light.sprite.x =
        light.baseX +
        Math.sin(driftTime) * light.driftX +
        Math.sin(driftTime * 0.43 + 1.2) * 4
      light.sprite.y =
        light.baseY +
        Math.cos(driftTime * 0.78) * light.driftY +
        Math.sin(driftTime * 1.31) * 3
      light.sprite.alpha = light.baseAlpha * (0.5 + pulse * 0.5) + sparkle * 0.34
      light.sprite.setScale(light.baseScale * (0.9 + pulse * 0.22 + sparkle * 0.14))
    })

  }

  resizeBackground(gameSize) {
    const screenWidth = gameSize.width
    const screenHeight = gameSize.height
    const coverScale = Math.max(
      screenWidth / DESIGN_WIDTH,
      screenHeight / DESIGN_HEIGHT,
    )
    const offsetX = (screenWidth - DESIGN_WIDTH * coverScale) / 2
    const offsetY = (screenHeight - DESIGN_HEIGHT * coverScale) / 2

    this.backgroundLayers
      .setPosition(offsetX, offsetY)
      .setScale(coverScale)
  }
}

onMounted(() => {
  game = new Phaser.Game({
    type: Phaser.WEBGL,
    parent: gameContainer.value,
    width: DESIGN_WIDTH,
    height: DESIGN_HEIGHT,
    backgroundColor: '#76cbe9',
    scene: GardenScene,
    scale: {
      mode: Phaser.Scale.RESIZE,
      autoRound: true,
    },
    render: {
      antialias: true,
      roundPixels: false,
      transparent: false,
    },
  })
})

onBeforeUnmount(() => {
  if (transitionTimer) window.clearTimeout(transitionTimer)
  game?.destroy(true)
  game = null
})
</script>

<template>
  <main
    class="garden-scene"
    :style="gameStyle"
    aria-label="Şekil Bahçesi şekil oyunu"
  >
    <div ref="gameContainer" class="game-container" />

    <section class="ui-stage" aria-label="Oyun arayüzü">
      <img class="game-logo" :src="gameLogoUrl" alt="Şekil Bahçesi" />

      <img class="question-progress" :src="progressBarUrl" alt="" />
      <template v-for="(result, index) in results" :key="index">
        <img
          v-if="result === true"
          class="progress-flower"
          :src="progressFlowerUrl"
          :style="{ left: flowerSlots[index] }"
          alt="Tamamlanan soru"
        />
      </template>

      <img
        :key="mascotState"
        class="mascot mascot--reaction"
        :src="mascotUrl"
        alt="Şekil Bahçesi maskotu"
      />

      <div class="question-card">
        <img class="question-card__board" :src="questionBoardUrl" alt="" />
        <p class="question-card__prompt game-prompt game-scale-text">{{ currentQuestion.prompt }}</p>
        <div
          v-if="currentQuestion.pattern"
          class="pattern-display"
          aria-label="Şekil örüntüsü"
        >
          <div
            v-for="(shape, index) in currentQuestion.pattern"
            :key="`${currentQuestionIndex}-${index}-${shape}`"
            class="pattern-shape"
            :class="`shape-display--${shape}`"
          />
          <span class="pattern-question">?</span>
        </div>
        <div
          v-else
          class="shape-display"
          :class="`shape-display--${currentQuestion.shape}`"
          :aria-label="currentQuestion.shape"
        />
      </div>

      <div class="answer-row" aria-label="Cevap seçenekleri">
        <button
          v-for="answer in currentQuestion.answers"
          :key="`${currentQuestionIndex}-${answer}`"
          class="answer game-tile jelly-tap"
          :class="{
            'answer--selected': selectedAnswer === answer && answer === currentQuestion.correct,
            'answer--correct': selectedAnswer === answer && answer === currentQuestion.correct,
            'answer--wrong': selectedAnswer === answer && answer !== currentQuestion.correct,
            'answer--reveal': revealedCorrect === answer,
          }"
          type="button"
          :disabled="isTransitioning"
          @click="chooseAnswer(answer)"
        >
          <img :src="answerPanelUrl" alt="" />
          <span>{{ answer }}</span>
        </button>
      </div>
    </section>

    <div class="butterfly-stage" aria-hidden="true">
      <div class="butterfly butterfly--left">
        <div class="butterfly__crop">
          <img class="butterfly__image butterfly__image--left" :src="butterfliesUrl" alt="" />
        </div>
      </div>
      <div class="butterfly butterfly--right">
        <div class="butterfly__crop">
          <img class="butterfly__image butterfly__image--right" :src="butterfliesUrl" alt="" />
        </div>
      </div>
    </div>
  </main>
</template>
<style>
:root {
  background: #76cbe9;
}

* {
  box-sizing: border-box;
}

html,
body,
#app {
  width: 100%;
  min-width: 0;
  height: 100%;
  margin: 0;
  overflow: hidden;
}

.garden-scene {
  position: relative;
  width: 100%;
  height: 100dvh;
  min-height: 100%;
  overflow: hidden;
  background: #76cbe9;
}

.game-container {
  width: 100%;
  height: 100%;
  min-width: 0;
  min-height: 0;
}

.game-container canvas {
  display: block;
}

.ui-stage {
  position: absolute;
  top: 50%;
  left: 50%;
  width: max(100vw, 177.683dvh);
  height: max(56.28vw, 100dvh);
  min-width: 100vw;
  min-height: 100dvh;
  overflow: hidden;
  color: #5a260f;
  font-family: "Arial Rounded MT Bold", "Trebuchet MS", system-ui, sans-serif;
  transform: translate(-50%, -50%);
  pointer-events: none;
  user-select: none;
  z-index: 1;
}

.butterfly-stage {
  position: absolute;
  z-index: 2;
  top: 50%;
  left: 50%;
  width: max(100vw, 177.683dvh);
  height: max(56.28vw, 100dvh);
  min-width: 100vw;
  min-height: 100dvh;
  overflow: hidden;
  transform: translate(-50%, -50%);
  pointer-events: none;
}

.game-logo {
  position: absolute;
}

.game-logo {
  top: 5.5%;
  left: 37%;
  width: 30%;
  height: 27%;
  object-fit: contain;
  filter: drop-shadow(0 4px 2px rgb(75 43 16 / 0.2));
  transform: scaleX(1.12);
  transform-origin: center;
}

.question-progress,
.progress-flower,
.mascot,
.question-card,
.answer-row {
  position: absolute;
}

.question-progress {
  top: 21%;
  left: 18.5%;
  width: 70%;
  height: 18%;
  object-fit: contain;
}

.progress-flower {
  z-index: 5;
  top: 25.1%;
  left: 40.8%;
  width: 5.5%;
  aspect-ratio: 1;
  object-fit: contain;
  animation: flower-arrive 760ms cubic-bezier(0.2, 0.9, 0.25, 1.25) both;
}

.mascot {
  z-index: 3;
  top: 32.5%;
  left: 20%;
  width: 18%;
  height: 36%;
  object-fit: contain;
  object-position: center bottom;
  filter: drop-shadow(0 7px 4px rgb(44 42 22 / 0.22));
}

.mascot--reaction {
  animation: mascot-reaction 520ms cubic-bezier(0.2, 0.9, 0.25, 1.25) both;
}

.question-card {
  z-index: 2;
  top: 32%;
  left: 34%;
  width: 38%;
  height: 41%;
}

.question-card__board {
  width: 100%;
  height: 100%;
  object-fit: contain;
  filter: drop-shadow(0 6px 3px rgb(65 42 17 / 0.22));
}

.question-card__prompt {
  position: absolute;
  top: 27.5%;
  left: 16%;
  width: 68%;
  margin: 0;
  color: #5a260f;
  font-size: calc(clamp(16px, 1.45vw, 27px) * var(--game-font-scale, 1));
  font-weight: 900;
  line-height: 1.15;
  text-align: center;
}

.shape-display {
  position: absolute;
  top: 56%;
  left: 50%;
  transform: translate(-50%, -50%);
}

.shape-display--square,
.shape-display--rectangle,
.shape-display--circle,
.shape-display--diamond {
  border: clamp(7px, 0.7vw, 13px) solid #10a9aa;
}

.shape-display--square {
  width: 18%;
  aspect-ratio: 1;
  border-radius: 7%;
}

.shape-display--rectangle {
  width: 28%;
  height: 18%;
  border-radius: 6%;
}

.shape-display--circle {
  width: 19%;
  aspect-ratio: 1;
  border-radius: 50%;
}

.shape-display--diamond {
  width: 17%;
  aspect-ratio: 1;
  transform: translate(-50%, -50%) rotate(45deg);
  border-radius: 5%;
}

.shape-display--triangle {
  width: 23%;
  height: 27%;
  background: #10a9aa;
  clip-path: polygon(50% 0, 100% 100%, 0 100%);
}

.shape-display--triangle::after {
  position: absolute;
  inset: 17% 16% 12%;
  background: #f7e4bd;
  clip-path: polygon(50% 0, 100% 100%, 0 100%);
  content: "";
}

.pattern-display {
  position: absolute;
  top: 48%;
  left: 10%;
  display: flex;
  align-items: center;
  justify-content: center;
  gap: 4%;
  width: 80%;
  height: 25%;
}

.pattern-shape {
  position: relative;
  flex: 0 0 auto;
  width: clamp(24px, 3.2vw, 48px);
  height: clamp(24px, 3.2vw, 48px);
  border-width: clamp(3px, 0.38vw, 7px);
}

.pattern-shape.shape-display--rectangle {
  width: clamp(34px, 4.5vw, 68px);
  height: clamp(22px, 2.8vw, 42px);
}

.pattern-shape.shape-display--diamond {
  transform: rotate(45deg) scale(0.82);
}

.pattern-question {
  min-width: 0.8em;
  color: #5a260f;
  font-size: calc(clamp(28px, 3vw, 48px) * var(--game-font-scale, 1));
  font-weight: 900;
  line-height: 1;
}

.answer-row {
  z-index: 4;
  left: 27.5%;
  bottom: 16.5%;
  display: flex;
  gap: 2.8%;
  width: 55%;
  height: 16%;
}

.answer {
  position: relative;
  width: 31.45%;
  height: 100%;
  padding: 0;
  border: 0;
  background: transparent;
  cursor: pointer;
  pointer-events: auto;
  transform: scale(1);
  transition: transform 180ms cubic-bezier(0.2, 0.9, 0.25, 1.35);
  animation: answer-arrive 460ms cubic-bezier(0.2, 0.9, 0.25, 1.3) both;
}

.answer:nth-child(2) { animation-delay: 80ms; }
.answer:nth-child(3) { animation-delay: 160ms; }

.answer img {
  width: 100%;
  height: 100%;
  object-fit: contain;
  filter: drop-shadow(0 4px 2px rgb(59 38 17 / 0.2));
}

.answer span {
  position: absolute;
  top: 43%;
  left: 10%;
  width: 80%;
  color: #5a260f;
  font-family: "Arial Rounded MT Bold", "Trebuchet MS", sans-serif;
  font-size: calc(clamp(14px, 1.12vw, 21px) * var(--game-font-scale, 1));
  font-weight: 900;
  line-height: 1.08;
  text-align: center;
}

.answer:not(:disabled):hover,
.answer:not(:disabled):focus-visible {
  transform: translateY(-3%) scale(1.055);
}

.answer:active {
  transform: translateY(-2%) scale(0.94);
}

.answer:disabled {
  cursor: default;
  pointer-events: none;
}

.answer--selected {
  animation: answer-feedback 780ms cubic-bezier(0.18, 0.9, 0.25, 1.28) both;
}

.answer--correct img,
.answer--reveal img {
  filter: sepia(0.22) saturate(1.35) hue-rotate(62deg) brightness(1.05) drop-shadow(0 0 13px #89c96c);
}

.answer--correct span,
.answer--reveal span {
  color: #356b2d;
}

.answer--wrong img {
  filter: sepia(1) saturate(4.8) hue-rotate(318deg) brightness(0.92) drop-shadow(0 0 15px #e83d35);
}

.answer--wrong span {
  color: #981c19;
}

.answer--reveal {
  animation: correct-hint 340ms ease-in-out 4 alternate;
}

@keyframes flower-arrive {
  from { opacity: 0; transform: translate(260%, 220%) scale(0.08) rotate(-220deg); }
  70% { opacity: 1; transform: translate(0, 0) scale(1.18) rotate(12deg); }
  to { opacity: 1; transform: translate(0, 0) scale(1) rotate(0); }
}

@keyframes answer-arrive {
  from { opacity: 0; }
  to { opacity: 1; }
}

@keyframes correct-hint {
  from { transform: scale(1); }
  to { transform: scale(1.1); }
}

@keyframes answer-feedback {
  0% { transform: scale(0.94); }
  42% { transform: scale(1.18); }
  70% { transform: scale(1.04); }
  100% { transform: scale(1.1); }
}

@keyframes mascot-reaction {
  from { opacity: 0.2; transform: scale(0.88); }
  70% { opacity: 1; transform: scale(1.06); }
  to { opacity: 1; transform: scale(1); }
}

.butterfly {
  position: absolute;
  transform-origin: center;
}

.butterfly__crop {
  width: 100%;
  height: 100%;
  overflow: hidden;
  animation: butterfly-flutter 360ms ease-in-out infinite alternate;
}

.butterfly__image {
  position: absolute;
  max-width: none;
}

.butterfly--left {
  top: 33.47%;
  left: 17.64%;
  width: 7.48%;
  height: 13.28%;
  animation: butterfly-left-flight 11.6s linear infinite;
}

.butterfly__image--left {
  top: -252%;
  left: -236%;
  width: 1337.6%;
  height: 752.8%;
}

.butterfly--right {
  top: 32.41%;
  left: 72.67%;
  width: 8.67%;
  height: 14.35%;
  animation: butterfly-right-flight 13.7s linear infinite;
}

.butterfly__image--right {
  top: -225.93%;
  left: -837.93%;
  width: 1153.1%;
  height: 697%;
}

@keyframes butterfly-flutter {
  from { transform: scaleX(0.9) scaleY(1.02); }
  to { transform: scaleX(1) scaleY(0.98); }
}

@keyframes butterfly-left-flight {
  0%, 100% { transform: translate(15%, 7%) rotate(3deg); }
  25% { transform: translate(60%, -18%) rotate(-7deg); }
  50% { transform: translate(0, 7%) rotate(5deg); }
  75% { transform: translate(-60%, -18%) rotate(-7deg); }
}

@keyframes butterfly-right-flight {
  0%, 100% { transform: translate(32%, 0) rotate(2deg); }
  25% { transform: translate(0, 32%) rotate(8deg); }
  50% { transform: translate(-32%, 0) rotate(-2deg); }
  75% { transform: translate(0, -32%) rotate(-8deg); }
}

@media (max-aspect-ratio: 4 / 3) {
  .ui-stage,
  .butterfly-stage {
    width: 100vw;
    height: 100dvh;
    min-width: 0;
    min-height: 0;
  }

  .game-logo {
    top: 2%;
    left: 10%;
    width: 80%;
    height: 19%;
  }

  .question-progress {
    top: 14%;
    left: 4%;
    width: 92%;
    height: 15%;
  }

  .progress-flower {
    top: 17.6%;
    width: 7%;
  }

  .mascot {
    top: 27%;
    left: 1%;
    width: 31%;
    height: 28%;
  }

  .question-card {
    top: 25%;
    left: 22%;
    width: 76%;
    height: 37%;
  }

  .answer-row {
    left: 4%;
    bottom: 13%;
    gap: 2%;
    width: 92%;
    height: 22%;
  }

  .answer {
    width: 32%;
    min-height: 44px;
  }

  .answer span {
    font-size: calc(clamp(13px, 4vw, 19px) * var(--game-font-scale, 1));
  }
}

</style>
