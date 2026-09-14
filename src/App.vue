<script setup>
import { computed, provide, reactive, ref } from 'vue'
import SekilBahcesi from './SekilBahcesi.vue'

const queryLevel = new URLSearchParams(window.location.search).get('level')
const level = ['kolay', 'orta', 'zor'].includes(queryLevel) ? queryLevel : 'kolay'

const round = ref(1)
const total = ref(5)
const correct = ref(0)
const points = ref(0)
const finished = computed(() => round.value > total.value)
const score = computed(() => points.value)

const engine = {
  round,
  total,
  correct,
  points,
  score,
  finished,
  answer(isCorrect, meta = {}, pts = 1) {
    if (finished.value) return
    if (isCorrect) {
      correct.value += 1
      points.value += Number(pts) || 0
    }
    console.info('Şekil Bahçesi tur cevabı:', { isCorrect, meta, pts })
    round.value += 1
  },
  complete(meta = {}) {
    if (finished.value) return
    console.info('Şekil Bahçesi tamamlandı:', meta)
    round.value = total.value + 1
  },
}

const params = reactive({
  level,
  rounds: total.value,
  feedbackDurationMs: 1900,
})

const liveSettings = reactive({ sizeScale: 1, elementCount: null, speed: 100 })
provide('liveSettings', liveSettings)
</script>

<template>
  <SekilBahcesi :engine="engine" :params="params" :pool="null" />
</template>
