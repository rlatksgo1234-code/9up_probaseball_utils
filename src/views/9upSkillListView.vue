<script setup lang="ts">
import { onMounted, ref, computed, watch, onBeforeUnmount } from 'vue'
import { ChevronLeft, ChevronRight, ChevronsLeft, ChevronsRight } from 'lucide-vue-next'

type Kind = 'normal' | 'enhanced'
interface NormalSkill {
  skill: string
  description?: string
  image?: string
  effects?: any[]
}
interface EnhancedSkill {
  enhanced_skill: string
  description?: string
  image?: string
  effects_by_level?: any[] | Record<string, any>
  effects_by_year?: Record<string, any[] | any> // 연도별: 배열(레벨별) 또는 기타 구조
}
interface SkillCardVM {
  kind: Kind
  name: string
  imageKey: string
  description: string
  effectsNormal?: any[]
  effectsByLevel?: any[] | Record<string, any>
}

/* =========================
   State
========================= */
const normalSkillData = ref<NormalSkill[]>([])
const enhancedSkillData = ref<EnhancedSkill[]>([])

const loading = ref(true)
const error = ref<string | null>(null)

const q = ref('')
const debouncedQ = ref('')
const type = ref<'normal' | 'enhanced'>('normal')

// 카드별 연도 선택 상태 (압도 전용 사용)
const selectedYearByName = ref<Record<string, string>>({})

// 🌟 전역 레벨(강화 탭) 기본값을 0으로 수정
const level = ref<number>(0)

// 페이지네이션
const page = ref(1)
const pageSize = ref(18)
const pageSizeOptions = [12, 18, 24, 36, 48]

/* =========================
   Debounce Search
========================= */
let qTimer: number | null = null
watch(q, (val) => {
  if (qTimer) window.clearTimeout(qTimer)
  qTimer = window.setTimeout(() => {
    debouncedQ.value = val.trim().toLowerCase()
  }, 150)
})
onBeforeUnmount(() => { if (qTimer) window.clearTimeout(qTimer) })

/* =========================
   Fast lookup maps
========================= */
const normalMap = ref<Record<string, NormalSkill>>({})
const enhancedMap = ref<Record<string, EnhancedSkill>>({})

onMounted(async () => {
  try {
    const [normalSkillRes, enhancedSkillRes] = await Promise.all([
      fetch('/DB/normal_skill.json'),
      fetch('/DB/enhanced_skill.json')
    ])
    normalSkillData.value = await normalSkillRes.json()
    enhancedSkillData.value = await enhancedSkillRes.json()

    // map 캐싱
    const nmp: Record<string, NormalSkill> = {}
    normalSkillData.value.forEach(s => { if (s.skill) nmp[s.skill] = s })
    normalMap.value = nmp

    const emp: Record<string, EnhancedSkill> = {}
    enhancedSkillData.value.forEach(s => { if (s.enhanced_skill) emp[s.enhanced_skill] = s })
    enhancedMap.value = emp

    // 압도(구 존재감) 기본 연도(최신) 초기화
    initDefaultYear('압도')
    // 이전 버전 호환성
    initDefaultYear('존재감')
  } catch (e: any) {
    error.value = e?.message || '스킬 데이터를 불러오지 못했습니다.'
  } finally {
    loading.value = false
  }
})

/* =========================
   Year helpers
========================= */
function yearOptionsFor(name: string): string[] {
  const byYear = enhancedMap.value[name]?.effects_by_year
  if (!byYear || typeof byYear !== 'object') return []
  return Object.keys(byYear).sort().reverse()
}
function initDefaultYear(name: string) {
  const opts = yearOptionsFor(name)
  if (opts.length && !selectedYearByName.value[name]) {
    selectedYearByName.value[name] = opts[0]
  }
}

/* =========================
   Sprite helper
   (압도/존재감 bg-${imageKey}${year})
========================= */
function spriteBgClass(card: SkillCardVM) {
  if (card.kind === 'enhanced' && (card.name === '압도' || card.name === '존재감')) {
    const y = selectedYearByName.value[card.name]
    return y ? `bg-${(card.imageKey || '')}${y}` : `bg-${card.imageKey}`
  }
  return `bg-${card.imageKey}`
}

/* =========================
   Render helpers
========================= */
function renderEffectItem(e: any): string {
  if (e == null) return ''
  if (typeof e === 'string') return e
  if (typeof e === 'number') return String(e)
  if (Array.isArray(e)) return e.map(renderEffectItem).join(', ')
  const parts: string[] = []
  Object.entries(e).forEach(([k, v]) => parts.push(`${k}: ${renderEffectItem(v)}`))
  return parts.join(' / ')
}

/**
 * 🌟 레벨 효과 리졸버 (0~15 레벨 매핑 수정 완료)
 */
function resolveLevelEffects(card: SkillCardVM, lvl: number) {
  if (card.kind !== 'enhanced') return []
  const meta = enhancedMap.value[card.name]
  if (!meta) return []

  const y = selectedYearByName.value[card.name]
  const yearNode = y ? meta.effects_by_year?.[y] : undefined

  // 1) 압도 케이스: 연도별 배열 = 레벨 테이블
  // 🌟 lvl이 0부터 시작하므로 배열 인덱스로 그대로 사용
  if (Array.isArray(yearNode)) {
    const idx = Math.min(Math.max(lvl, 0), Math.max(0, yearNode.length - 1))
    const item = yearNode[idx]
    return Array.isArray(item) ? item : [item]
  }

  // 2) 연도 아래에 레벨 테이블이 중첩된 경우(객체)
  if (yearNode && typeof yearNode === 'object') {
    const byL =
      (yearNode as any).effects_by_level ??
      (yearNode as any).by_level ??
      (yearNode as any).levels
      
    if (Array.isArray(byL)) {
      const idx = Math.min(Math.max(lvl, 0), Math.max(0, byL.length - 1))
      const arr = byL[idx] ?? []
      return Array.isArray(arr) ? arr : [arr]
    }
    if (byL && typeof byL === 'object') {
      // 🌟 JSON의 객체 키가 '1', '2' 형태로 되어 있을 수 있으므로 lvl + 1 로 조회
      const arr = byL[String(lvl + 1)] ?? []
      return Array.isArray(arr) ? arr : [arr]
    }
  }

  // 3) 전역 레벨 테이블
  const byLevel = meta.effects_by_level
  if (Array.isArray(byLevel)) {
    const idx = Math.min(Math.max(lvl, 0), Math.max(0, byLevel.length - 1))
    const arr = byLevel[idx] ?? []
    return Array.isArray(arr) ? arr : [arr]
  }
  if (byLevel && typeof byLevel === 'object') {
    const arr = (byLevel as Record<string, any>)[String(lvl + 1)] ?? []
    return Array.isArray(arr) ? arr : [arr]
  }

  return []
}
const levelEffects = (card: SkillCardVM, lvl: number) => resolveLevelEffects(card, lvl)

function yearEffects(card: SkillCardVM, y?: string) {
  if (!y || card.kind !== 'enhanced') return []
  const node = enhancedMap.value[card.name]?.effects_by_year?.[y]
  if (Array.isArray(node)) return [] 
  if (!node) return []
  return Array.isArray(node) ? node : [node]
}

/* =========================
   Filtered list (name only)
========================= */
const listToRender = computed<SkillCardVM[]>(() => {
  const needle = debouncedQ.value
  const hasNeedle = needle.length > 0

  const passNormal = (s: NormalSkill) =>
    !hasNeedle || (s.skill ?? '').toLowerCase().includes(needle)

  const passEnhanced = (s: EnhancedSkill) =>
    !hasNeedle || (s.enhanced_skill ?? '').toLowerCase().includes(needle)

  if (type.value === 'normal') {
    return normalSkillData.value
      .filter(passNormal)
      .map<SkillCardVM>((s) => ({
        kind: 'normal',
        name: s.skill,
        imageKey: s.image || '',
        description: s.description ?? '',
        effectsNormal: s.effects ?? []
      }))
      .sort((a, b) => a.name.localeCompare(b.name, 'ko'))
  }

  return enhancedSkillData.value
    .filter(passEnhanced)
    .map<SkillCardVM>((s) => ({
      kind: 'enhanced',
      name: s.enhanced_skill,
      imageKey: s.image || '',
      description: s.description ?? '',
      effectsByLevel: s.effects_by_level ?? []
    }))
    .sort((a, b) => {
      const special = ['골든글러브', '압도', '존재감']
      const aIsSpecial = special.includes(a.name)
      const bIsSpecial = special.includes(b.name)
      
      if (aIsSpecial && !bIsSpecial) return -1
      if (!aIsSpecial && bIsSpecial) return 1
      
      return a.name.localeCompare(b.name, 'ko')
    })
})

/* =========================
   Pagination derived
========================= */
const totalCount = computed(() => listToRender.value.length)
const totalPages = computed(() => Math.max(1, Math.ceil(totalCount.value / pageSize.value)))
const pageStartIndex = computed(() => (page.value - 1) * pageSize.value)
const pageEndIndex = computed(() => Math.min(pageStartIndex.value + pageSize.value, totalCount.value))
const pagedList = computed(() => listToRender.value.slice(pageStartIndex.value, pageEndIndex.value))

watch([debouncedQ, type, pageSize], () => { page.value = 1 })
function goToPage(p: number) {
  page.value = Math.min(Math.max(1, p), totalPages.value)
}
</script>

<template>
  <div class="max-w-7xl mx-auto px-4 sm:px-6 lg:px-8 py-6 lg:py-8">
    <!-- Header -->
    <div class="mb-6 lg:mb-8 flex flex-col gap-4 lg:flex-row lg:items-end lg:justify-between">
      <h1 class="text-2xl font-bold tracking-tight text-neutral-900 dark:text-neutral-100">
        스킬 목록
      </h1>

      <div class="w-full lg:w-auto flex flex-col sm:flex-row gap-3 sm:items-center">
        <!-- Tabs -->
        <div role="tablist"
             class="inline-flex w-full sm:w-auto rounded-md border border-neutral-300 dark:border-neutral-600 overflow-hidden shadow-sm">
          <button
            role="tab"
            :aria-selected="type==='normal'"
            class="flex-1 sm:flex-none px-4 py-2 text-sm transition-colors focus:outline-none focus-visible:ring-2 focus-visible:ring-blue-400"
            :class="type==='normal'
              ? 'bg-neutral-200 dark:bg-neutral-700 font-semibold text-neutral-900 dark:text-neutral-100'
              : 'bg-white dark:bg-neutral-900 text-neutral-700 dark:text-neutral-300 hover:bg-neutral-50 dark:hover:bg-neutral-800'"
            @click="type='normal'">
            일반
          </button>
          <button
            role="tab"
            :aria-selected="type==='enhanced'"
            class="flex-1 sm:flex-none px-4 py-2 text-sm transition-colors focus:outline-none focus-visible:ring-2 focus-visible:ring-blue-400 border-l border-neutral-300 dark:border-neutral-600"
            :class="type==='enhanced'
              ? 'bg-neutral-200 dark:bg-neutral-700 font-semibold text-neutral-900 dark:text-neutral-100'
              : 'bg-white dark:bg-neutral-900 text-neutral-700 dark:text-neutral-300 hover:bg-neutral-50 dark:hover:bg-neutral-800'"
            @click="type='enhanced'">
            강화
          </button>
        </div>

        <!-- Search -->
        <div class="relative w-full sm:w-72">
          <input
            v-model="q"
            type="text"
            placeholder="스킬명 검색..."
            class="w-full rounded-md border border-neutral-300 dark:border-neutral-600 bg-white dark:bg-neutral-800 px-3 py-2 pr-10 text-sm text-neutral-900 dark:text-neutral-100
                   placeholder-neutral-400 dark:placeholder-neutral-500 shadow-sm focus:border-blue-500 focus:ring-4 focus:ring-blue-500/10"
            aria-label="스킬 검색"
          />
          <button
            v-if="q"
            @click="q=''"
            class="absolute inset-y-0 right-0 px-3 text-neutral-400 hover:text-neutral-600 dark:hover:text-neutral-300 focus:outline-none"
            aria-label="검색어 지우기">
            ✕
          </button>
        </div>
      </div>
    </div>

    <!-- Top toolbar -->
    <div class="mb-4 flex flex-col gap-3 sm:flex-row sm:items-center sm:justify-between">
      <div class="text-sm text-neutral-600 dark:text-neutral-300">
        총 <span class="font-semibold">{{ totalCount }}</span>개
        <template v-if="totalCount">
          · <span class="font-semibold">{{ pageStartIndex + 1 }}</span>–<span class="font-semibold">{{ pageEndIndex }}</span> 표시
        </template>
      </div>
      <div class="flex items-center gap-2">
        <label class="text-sm text-neutral-600 dark:text-neutral-300">페이지당</label>
        <select
          v-model.number="pageSize"
          class="rounded-md border border-neutral-300 dark:border-neutral-600 bg-white dark:bg-neutral-800 px-2 py-1 text-sm text-neutral-900 dark:text-neutral-100
                 focus:border-blue-500 focus:ring-4 focus:ring-blue-500/10">
          <option v-for="n in pageSizeOptions" :key="n" :value="n">{{ n }}</option>
        </select>
      </div>
    </div>

    <!-- 🌟 Enhanced-only Options (level slider) : 0~15로 수정 완료 -->
    <div v-if="type === 'enhanced'" class="mb-6 flex flex-wrap items-center gap-4">
      <div class="flex items-center gap-3">
        <span class="text-sm text-neutral-700 dark:text-neutral-300">레벨</span>
        <input type="range" min="0" max="15" v-model.number="level" class="slider w-36 accent-blue-600" />
        <span class="inline-flex items-center rounded-full border border-blue-200 dark:border-blue-800 px-2 py-0.5 text-xs font-semibold
                      text-blue-700 dark:text-blue-300 bg-blue-50 dark:bg-blue-900/30">
          {{ level }}
        </span>
      </div>
    </div>

    <!-- Loading -->
    <div v-if="loading" class="grid grid-cols-1 md:grid-cols-2 xl:grid-cols-3 gap-4">
      <div v-for="i in 6" :key="i" class="rounded-md border border-neutral-200 dark:border-neutral-700 bg-white dark:bg-neutral-900 p-4 animate-pulse">
        <div class="h-6 w-40 bg-neutral-200 dark:bg-neutral-700 rounded mb-4"></div>
        <div class="space-y-2">
          <div class="h-3 bg-neutral-200 dark:bg-neutral-700 rounded"></div>
          <div class="h-3 w-11/12 bg-neutral-200 dark:bg-neutral-700 rounded"></div>
          <div class="h-3 w-9/12 bg-neutral-200 dark:bg-neutral-700 rounded"></div>
        </div>
      </div>
    </div>

    <!-- Error -->
    <div v-else-if="error" class="text-center py-10">
      <p class="inline-flex items-center gap-2 rounded-md border border-red-200 bg-red-50 px-4 py-3 text-sm text-red-600
                 dark:border-red-900/40 dark:bg-red-900/20 dark:text-red-300">
        {{ error }}
      </p>
    </div>

    <!-- Cards -->
    <div v-else>
      <div class="grid grid-cols-1 md:grid-cols-2 xl:grid-cols-3 gap-4">
        <div
          v-for="card in pagedList"
          :key="card.kind + ':' + card.name"
          class="group rounded-md border border-neutral-200 dark:border-neutral-700 bg-white dark:bg-neutral-900 p-4 shadow-sm hover:shadow-md transition-shadow"
          :class="['골든글러브', '압도', '존재감'].includes(card.name) ? 'ring-2 ring-yellow-400 dark:ring-yellow-500 bg-yellow-50/10 dark:bg-yellow-900/10' : ''"
        >
          <!-- Header -->
          <div class="flex items-start justify-between mb-3">
            <div class="flex items-center gap-3">
              <div :class="spriteBgClass(card)"
                   class="h-10 w-10 rounded-md ring-1 ring-neutral-200 dark:ring-neutral-700 shrink-0"></div>
              <h3 class="font-semibold text-neutral-900 dark:text-neutral-100 leading-tight flex items-center gap-2">
                {{ card.name }}
                <span v-if="['골든글러브', '압도', '존재감'].includes(card.name)" class="px-1.5 py-0.5 rounded text-[10px] font-bold bg-yellow-100 text-yellow-800 dark:bg-yellow-900/50 dark:text-yellow-200 border border-yellow-200 dark:border-yellow-700/50">특수</span>
              </h3>
            </div>
          </div>

          <!-- Effects -->
          <div v-if="card.kind === 'normal'" class="text-sm text-neutral-800 dark:text-neutral-200">
            <pre class="whitespace-pre-wrap leading-relaxed font-mono text-[13px] bg-neutral-50 dark:bg-neutral-800/60 rounded p-3 border border-neutral-100 dark:border-neutral-800">
{{ card.effectsNormal }}
            </pre>
          </div>

          <div v-else class="space-y-2">
            <div
              v-if="(card.name === '압도' || card.name === '존재감') && yearOptionsFor(card.name).length"
              class="flex items-center gap-3 mb-1"
            >
              <span class="text-xs text-neutral-600 dark:text-neutral-300">년도</span>
              <select
                v-model="selectedYearByName[card.name]"
                class="rounded-md border border-neutral-300 dark:border-neutral-600 bg-white dark:bg-neutral-800 px-2 py-1 text-xs
                       text-neutral-900 dark:text-neutral-100 focus:border-blue-500 focus:ring-4 focus:ring-blue-500/10"
              >
                <option v-for="y in yearOptionsFor(card.name)" :key="y" :value="y">{{ y }}</option>
              </select>
            </div>

            <ul v-if="levelEffects(card, level)?.length" class="list-disc pl-5 text-sm text-neutral-800 dark:text-neutral-200">
              <li v-for="(e, i) in levelEffects(card, level)" :key="'l'+i">
                {{ renderEffectItem(e) }}
              </li>
            </ul>

            <ul
              v-if="yearEffects(card, selectedYearByName[card.name])?.length"
              class="list-disc pl-5 text-sm text-neutral-800 dark:text-neutral-200"
            >
              <li v-for="(e, i) in yearEffects(card, selectedYearByName[card.name])" :key="'y'+i">
                {{ renderEffectItem(e) }}
              </li>
            </ul>
          </div>

          <!-- Description -->
          <p v-if="card.description" class="mt-3 text-xs leading-relaxed text-neutral-600 dark:text-neutral-400">
            {{ card.description }}
          </p>
        </div>
      </div>

      <!-- Pagination -->
      <div v-if="totalPages > 1" class="mt-6 flex flex-col items-center gap-3">
        <nav class="inline-flex -space-x-px rounded-md shadow-sm" aria-label="Pagination">
          <button
            class="px-3 py-2 text-sm border border-neutral-300 dark:border-neutral-700 rounded-l-md
                   bg-white dark:bg-neutral-900 text-neutral-700 dark:text-neutral-300 hover:bg-neutral-50 dark:hover:bg-neutral-800 disabled:opacity-50"
            :disabled="page === 1"
            @click="goToPage(1)"
            aria-label="첫 페이지">
            <ChevronsLeft class="w-4 h-4" />
          </button>
          <button
            class="px-3 py-2 text-sm border border-neutral-300 dark:border-neutral-700
                   bg-white dark:bg-neutral-900 text-neutral-700 dark:text-neutral-300 hover:bg-neutral-50 dark:hover:bg-neutral-800 disabled:opacity-50"
            :disabled="page === 1"
            @click="goToPage(page - 1)"
            aria-label="이전">
            <ChevronLeft class="w-4 h-4" />
          </button>
          <span class="px-4 py-2 text-sm border border-neutral-300 dark:border-neutral-700 bg-neutral-50 dark:bg-neutral-800 text-neutral-700 dark:text-neutral-200">
            {{ page }} / {{ totalPages }}
          </span>
          <button
            class="px-3 py-2 text-sm border border-neutral-300 dark:border-neutral-700
                   bg-white dark:bg-neutral-900 text-neutral-700 dark:text-neutral-300 hover:bg-neutral-50 dark:hover:bg-neutral-800 disabled:opacity-50"
            :disabled="page === totalPages"
            @click="goToPage(page + 1)"
            aria-label="다음">
            <ChevronRight class="w-4 h-4" />
          </button>
          <button
            class="px-3 py-2 text-sm border border-neutral-300 dark:border-neutral-700 rounded-r-md
                   bg-white dark:bg-neutral-900 text-neutral-700 dark:text-neutral-300 hover:bg-neutral-50 dark:hover:bg-neutral-800 disabled:opacity-50"
            :disabled="page === totalPages"
            @click="goToPage(totalPages)"
            aria-label="마지막 페이지">
            <ChevronsRight class="w-4 h-4" />
          </button>
        </nav>

        <div class="text-xs text-neutral-500 dark:text-neutral-400">
          {{ pageStartIndex + 1 }}–{{ pageEndIndex }} / {{ totalCount }}
        </div>
      </div>
    </div>

    <!-- Empty -->
    <div v-if="!loading && !error && !totalCount" class="text-center py-12">
      <div class="inline-flex items-center rounded-md border border-neutral-200 dark:border-neutral-700 bg-white dark:bg-neutral-900 px-4 py-3">
        <span class="text-sm text-neutral-500 dark:text-neutral-400">조건에 맞는 스킬이 없습니다</span>
      </div>
    </div>
  </div>
</template>

<style>
/* range input custom look */
.slider::-webkit-slider-thumb {
  appearance: none;
  height: 18px; width: 18px; border-radius: 50%;
  background: linear-gradient(45deg, #6366f1, #8b5cf6);
  box-shadow: 0 4px 12px rgba(99, 102, 241, 0.3);
  cursor: pointer; transition: all 0.2s ease;
}
.slider::-webkit-slider-thumb:hover { transform: scale(1.08); }
.slider::-webkit-slider-track { height: 8px; border-radius: 4px; background: linear-gradient(to right, #e5e7eb, #d1d5db); }
.slider::-moz-range-thumb {
  height: 18px; width: 18px; border-radius: 50%;
  background: linear-gradient(45deg, #6366f1, #8b5cf6);
  cursor: pointer; border: none; box-shadow: 0 4px 12px rgba(99,102,241,.3);
}
</style>
