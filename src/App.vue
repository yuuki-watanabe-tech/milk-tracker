<script setup lang="ts">
import { computed, onMounted, reactive, ref, watch } from 'vue';

// ─────────────────────────────────────────────
// LocalStorage ユーティリティ
// ─────────────────────────────────────────────
const STORAGE_KEY = 'milk-tracker:v1'

type FeedEntry = { id: string; time: string; amount: number }

type DayData = { entries: FeedEntry[]; manualTotal?: number }

type Store = {
  dailyLimit: number
  days: Record<string, DayData>
}

function getTodayKey() {
  const d = new Date()
  const y = d.getFullYear()
  const m = String(d.getMonth() + 1).padStart(2, '0')
  const day = String(d.getDate()).padStart(2, '0')
  return `${y}-${m}-${day}`
}

const defaultStore: Store = {
  dailyLimit: 800, // ml: 初期値。必要に応じて変更可
  days: {}
}

const store = reactive<Store>({ ...defaultStore })

function load() {
  try {
    const raw = localStorage.getItem(STORAGE_KEY)
    if (!raw) return
    const parsed = JSON.parse(raw)
    Object.assign(store, defaultStore, parsed)
  } catch (e) {
    console.error('Failed to parse storage', e)
  }
}

function save() {
  localStorage.setItem(STORAGE_KEY, JSON.stringify(store))
}

onMounted(load)
watch(store, save, { deep: true })

// ─────────────────────────────────────────────
// アプリ状態
// ─────────────────────────────────────────────
const todayKey = ref(getTodayKey())
const now = () => new Date()

const currentDay = computed<DayData>(() => {
  if (!store.days[todayKey.value]) store.days[todayKey.value] = { entries: [] }
  return store.days[todayKey.value]
})

const amountNow = ref<number | null>(null)

// 手入力の当日合計（要件に合わせて「ユーザが入力できる」よう保持）
const manualTotal = computed({
  get: () => currentDay.value.manualTotal ?? NaN,
  set: (v: number) => (currentDay.value.manualTotal = isNaN(Number(v)) ? undefined : Number(v))
})

const calcTotal = computed(() =>
  currentDay.value.entries.reduce((sum, e) => sum + (Number(e.amount) || 0), 0)
)

// 表示に使う「当日の摂取量合計」: 手入力があれば優先、それ以外は計算値
const totalForToday = computed(() => {
  return Number.isFinite(manualTotal.value) ? Number(manualTotal.value) : calcTotal.value
})

const limit = computed({
  get: () => store.dailyLimit,
  set: (v: number) => (store.dailyLimit = Math.max(0, Math.floor(Number(v) || 0)))
})

const percent = computed(() => {
  if (!limit.value) return 0
  return Math.min(100, Math.round((totalForToday.value / limit.value) * 100))
})

function addEntry() {
  if (amountNow.value == null || isNaN(Number(amountNow.value)) || Number(amountNow.value) <= 0) return
  const d = now()
  const hh = String(d.getHours()).padStart(2, '0')
  const mm = String(d.getMinutes()).padStart(2, '0')
  currentDay.value.entries.unshift({
    id: `${d.getTime()}-${Math.random().toString(36).slice(2, 6)}`,
    time: `${hh}:${mm}`,
    amount: Math.floor(Number(amountNow.value))
  })
  // 手入力の当日合計を使っている場合は、それも同期的に増やす
  if (Number.isFinite(manualTotal.value)) {
    currentDay.value.manualTotal = Number(manualTotal.value) + Math.floor(Number(amountNow.value))
  }
  amountNow.value = null
}

function deleteEntry(id: string) {
  currentDay.value.entries = currentDay.value.entries.filter(e => e.id !== id)
}

function resetToday() {
  if (!confirm('本日の記録をリセットしますか？')) return
  currentDay.value.entries = []
  currentDay.value.manualTotal = undefined
}

function carryOverToManual() {
  currentDay.value.manualTotal = calcTotal.value
}

// 日付が変わったら todayKey を更新（シンプルチェック）
setInterval(() => {
  const k = getTodayKey()
  if (k !== todayKey.value) todayKey.value = k
}, 60_000)
</script>

<template>
  <main class="min-h-screen px-4 pb-4">
    <header class="max-w-xl mx-auto pt-10 pb-6 text-center">
      <h1 class="text-2xl font-bold text-rose-700 tracking-tight">ミルク記録</h1>
      <p class="mt-1 text-sm text-slate-600">一日の摂取量をかんたん管理（オフライン・ログイン不要）</p>
    </header>

    <section class="max-w-xl mx-auto card p-5">
      <div class="flex items-center gap-3">
        <div class="shrink-0 w-14 h-14 rounded-2xl bg-rose-100 grid place-items-center text-2xl">🍼</div>
        <div class="flex-1">
          <div class="flex items-end justify-between gap-3">
            <div>
              <p class="text-xs text-slate-500">本日 ({{ todayKey }})</p>
              <p class="text-3xl font-semibold">{{ totalForToday }} <span class="text-base text-slate-500">ml</span></p>
            </div>
            <div class="text-right">
              <p class="text-xs text-slate-500">上限</p>
              <p class="text-lg font-medium">{{ limit }} ml</p>
            </div>
          </div>
          <div class="mt-3 w-full h-3 bg-rose-100 rounded-full overflow-hidden" aria-label="progress">
            <div class="h-full bg-rose-400" :style="{ width: percent + '%' }"></div>
          </div>
          <p class="mt-1 text-xs text-slate-600">達成率: {{ percent }}%</p>
        </div>
      </div>

      <div class="mt-6 grid grid-cols-1 md:grid-cols-2 gap-4">
        <div>
          <label class="label">今回の摂取量 (ml)</label>
          <div class="flex gap-2">
            <input class="input" type="number" inputmode="numeric" min="0" step="10" v-model.number="amountNow"
              placeholder="例: 120" />
            <button class="btn-primary bg-rose-500 text-white" @click="addEntry">追加する</button>
          </div>
        </div>
        <div>
          <label class="label">当日の摂取量合計 (ml)</label>
          <div class="flex gap-2">
            <input class="input" type="number" inputmode="numeric" min="0" step="10" v-model.number="manualTotal"
              placeholder="自動計算 or 手入力" />
            <button class="btn-primary bg-white text-rose-700" @click="carryOverToManual">計算値を反映</button>
          </div>
          <label class="label mt-4 block">一日の摂取量の上限 (ml)</label>
          <input class="input" type="number" inputmode="numeric" min="0" step="10" v-model.number="limit"
            placeholder="例: 800" />
        </div>
      </div>

      <div class="mt-6 flex flex-wrap gap-2">
        <button class="btn-primary bg-white text-rose-700" @click="resetToday">本日データをリセット</button>
      </div>
    </section>

    <section class="max-w-xl mx-auto mt-6 card p-5">
      <h2 class="text-base font-semibold text-rose-700">本日の記録</h2>
      <p v-if="!currentDay.entries.length" class="text-sm text-slate-600 mt-2">まだ記録がありません。上のフォームから「今回の摂取量」を追加してください。</p>
      <ul v-else class="mt-3 divide-y divide-rose-100">
        <li v-for="e in currentDay.entries" :key="e.id" class="py-3 flex items-center justify-between">
          <div>
            <p class="font-medium">{{ e.amount }} ml</p>
            <p class="text-xs text-slate-500">{{ e.time }}</p>
          </div>
          <button class="text-rose-600 hover:underline" @click="deleteEntry(e.id)">削除</button>
        </li>
      </ul>
    </section>

    <footer class="max-w-xl mx-auto text-center text-xs text-slate-500 mt-6">
      <p>データはお使いの端末の <span class="font-medium">LocalStorage</span> に保存されます。ブラウザを変えるとデータは引き継がれません。</p>
      <footer class="mt-2 py-6 border-t text-center text-sm text-gray-500">
        <a href="/about.html" class="mx-3 hover:text-orange-500">About</a>
        <a href="/privacy.html" class="mx-3 hover:text-orange-500">Privacy Policy</a>
      </footer>
    </footer>
  </main>
</template>

<style scoped>
/**** ほんのり女性向けの柔らかい雰囲気を意識した配色・丸み ****/
</style>
