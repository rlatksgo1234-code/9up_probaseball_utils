<script setup lang="ts">
import { defineProps, defineEmits, computed, watch, onMounted, onBeforeUnmount, ref } from 'vue'
import { Star, Search, ChevronDown, Check } from 'lucide-vue-next'
import type { PropType } from 'vue'

const decodeHtmlEntities = (text: string): string => {
  if (!text) return text;
  
  return text
    .replace(/&#x27;/g, "'")
    .replace(/&#39;/g, "'")
    .replace(/&apos;/g, "'")
    .replace(/&quot;/g, '"')
    .replace(/&lt;/g, '<')
    .replace(/&gt;/g, '>')
    .replace(/&amp;/g, '&');
}

const decodeJsonData = (data: any): any => {
  if (typeof data === 'string') {
    return decodeHtmlEntities(data);
  } else if (Array.isArray(data)) {
    return data.map(decodeJsonData);
  } else if (data && typeof data === 'object') {
    const decoded: any = {};
    for (const [key, value] of Object.entries(data)) {
      decoded[key] = decodeJsonData(value);
    }
    return decoded;
  }
  return data;
}

const props = defineProps({
  tabKey: { type: String, default: 'Batters' },
  allFields: Array as () => string[],
  selectFields: Array as () => string[],
  rarityField: String,
  filterOptions: Object as PropType<Record<string, string[]>>,
  fieldLabels: Object as PropType<Record<string, string>>,
  filters: Object as PropType<Record<string, any>>,
  synergyOptions: Array as () => string[] | undefined,
  nameOptions: Array as () => string[] | undefined,
})
const emit = defineEmits(['update:filters'])

const normVal = (v: unknown) => String(v ?? '').trim().toLowerCase()

const isSelected = (field: string, value: string) =>
  Array.isArray(props.filters?.[field]) &&
  props.filters![field].some((v: any) => normVal(v) === normVal(value))

const toggleFilter = (field: string, value: string) => {
  const picked: string[] = Array.isArray(props.filters?.[field]) ? [...props.filters![field]] : []
  const idx = picked.findIndex((v) => normVal(v) === normVal(value))
  if (idx !== -1) picked.splice(idx, 1)
  else picked.push(value)
  emit('update:filters', { ...(props.filters ?? {}), [field]: picked })
}

const update = (field: string, value: unknown) =>
  emit('update:filters', { ...(props.filters ?? {}), [field]: value })

const handleImageError = (e: Event) => {
  const target = e.target as HTMLElement
  if (target) {
    target.style.display = 'none'
    const nextElem = target.nextElementSibling as HTMLElement
    if (nextElem) {
      nextElem.style.display = 'flex'
      nextElem.classList.remove('hidden')
    }
  }
}

const currentYear = new Date().getFullYear()
const yearOptions = computed(() => {
  const years: string[] = []
  for (let y = currentYear - 1; y >= 1982; y--) years.push(String(y))
  return years
})

const gradeOrder = ['DGN', 'TOP', 'GG', 'ACE', 'HIT', 'GGY', 'MMVP', 'ROY', 'TEA', 'POS', 'ASG', 'SEA'] as const
const visibleGrades = computed(() =>
  gradeOrder.filter((g) => props.filterOptions?.grade?.includes(g))
)

const areAllGradesSelected = computed(
  () => visibleGrades.value.length > 0 && visibleGrades.value.every((g) => isSelected('grade', g))
)

const allTeamsSelected = computed(() => {
  const teams = props.filterOptions?.team ?? []
  if (!teams.length) return false
  const selArr: string[] = Array.isArray(props.filters?.team) ? props.filters!.team : []
  if (selArr.length === 0) return false
  const selSet = new Set(selArr.map(normVal))
  return teams.every(t => selSet.has(normVal(t)))
})

const toggleAllGrades = () => {
  if (!visibleGrades.value.length) return
  emit('update:filters', {
    ...(props.filters ?? {}),
    grade: areAllGradesSelected.value ? [] : [...visibleGrades.value]
  })
}

const toggleAllTeams = () => {
  const teams = props.filterOptions?.team ?? []
  if (!teams.length) return
  emit('update:filters', {
    ...(props.filters ?? {}),
    team: allTeamsSelected.value ? [] : [...teams]
  })
}

watch(
  () => props.filterOptions?.team,
  (teams) => {
    if (!teams?.length) return
    const selected = props.filters?.team ?? []
    if (selected.length === 0) {
      emit('update:filters', { ...(props.filters ?? {}), team: [...teams] })
    }
  },
  { immediate: true }
)

type CollapsibleKey = 'skill' | 'enhancedSkill' | 'position' | 'year' | 'rgt'
type Collapses = Record<CollapsibleKey, boolean>

const DEFAULT_COLLAPSES: Collapses = {
  skill: false,
  enhancedSkill: false,
  position: false,
  year: false,
  rgt: false,
}

const LS_KEY = '9up.filter.collapses.v1'
const loadAllFromLS = (): Record<string, Collapses> => {
  try { return JSON.parse(localStorage.getItem(LS_KEY) || '{}') } catch { return {} }
}
const saveAllToLS = (obj: Record<string, Collapses>) =>
  localStorage.setItem(LS_KEY, JSON.stringify(obj))

const collapses = ref<Collapses>({ ...DEFAULT_COLLAPSES })
const setByTab = (tab: string) => {
  const all = loadAllFromLS()
  collapses.value = all[tab] ? { ...DEFAULT_COLLAPSES, ...all[tab] } : { ...DEFAULT_COLLAPSES }
}
const persistCurrent = () => {
  const all = loadAllFromLS()
  all[props.tabKey] = { ...collapses.value }
  saveAllToLS(all)
}
watch(() => props.tabKey, (t) => setByTab(t), { immediate: true })
watch(collapses, persistCurrent, { deep: true })
const toggleCollapse = (k: CollapsibleKey) => { collapses.value[k] = !collapses.value[k] }

const selectedCount = (field: 'skill' | 'enhancedSkill') =>
  Array.isArray(props.filters?.[field]) ? props.filters![field].length : 0
const hasSkills = computed(() => (props.filterOptions?.skill?.length ?? 0) > 0)
const hasEnhancedSkills = computed(() => (props.filterOptions?.enhancedSkill?.length ?? 0) > 0)

const teamLogos: Record<string, string> = {
  kia: '/assets/logos/symbol/emblem_s_01_kia.png',
  haitai: '/assets/logos/symbol/emblem_s_01_haitai.png',
  samsung: '/assets/logos/symbol/emblem_s_02_samsung.png',
  lotte: '/assets/logos/symbol/emblem_s_03_lotte.png',
  kiwoom: '/assets/logos/symbol/emblem_s_04_kiwoom.png',
  nexen: '/assets/logos/symbol/emblem_s_04_nexen.png',
  binggrae: '/assets/logos/symbol/emblem_s_05_bingrae.png',
  hanwha: '/assets/logos/symbol/emblem_s_05_hanwha.png',
  lg: '/assets/logos/symbol/emblem_s_06_lg.png',
  mbc: '/assets/logos/symbol/emblem_s_06_mbc.png',
  doosan: '/assets/logos/symbol/emblem_s_07_doosan.png',
  ob: '/assets/logos/symbol/emblem_s_07_ob.png',
  sk: '/assets/logos/symbol/emblem_s_08_sk.png',
  ssg: '/assets/logos/symbol/emblem_s_08_ssg.png',
  nc: '/assets/logos/symbol/emblem_s_09_nc.png',
  kt: '/assets/logos/symbol/emblem_s_10_kt.png',
  sbw: '/assets/logos/symbol/emblem_s_11_sbw.png',
  hyundai: '/assets/logos/symbol/emblem_s_12_hyundai.png',
  pacific: '/assets/logos/symbol/emblem_s_13_pacific.png',
  sammi: '/assets/logos/symbol/emblem_s_14_sammi.png',
  chungbo: '/assets/logos/symbol/emblem_s_15_chungbo.png',
  dream: '/assets/logos/symbol/emblem_s_99_dream.png',
  nanum: '/assets/logos/symbol/emblem_s_99_nanum.png'
}

const normalSkillData = ref<any[]>([])
const enhancedSkillData = ref<any[]>([])

onMounted(async () => {
  try {
    const [n, e] = await Promise.all([
      fetch('/DB/normal_skill.json'),
      fetch('/DB/enhanced_skill.json')
    ])
    
    const normalData = await n.json()
    const enhancedData = await e.json()
    
    normalSkillData.value = decodeJsonData(normalData)
    enhancedSkillData.value = enhancedData
  
  } catch (error) {
    console.error('Failed to load skill data:', error)
  }
})

const matchSkillInfo = (skill: string, type: string) => {
  if (type === 'normal') return normalSkillData.value.find((s) => s.skill === skill)?.image || ''
  if (type === 'enhanced' || type === 'enhanced:GG') return enhancedSkillData.value.find((s) => s.enhanced_skill === skill)?.image || ''
  return ''
}

const getSkillText = (skill: string, type: 'normal' | 'enhanced' = 'enhanced'): string => {
  if (type === 'normal') {
    const found = normalSkillData.value.find((s) => s.skill === skill)
    return found ? decodeHtmlEntities(found.skill || skill) : skill
  }
  
  const found = enhancedSkillData.value.find((s) => s.enhanced_skill === skill)
  return found ? decodeHtmlEntities(found.enhanced_skill || skill) : skill
}

/* =========================
   포함할 이름 (부분 일치)
========================= */
const nameInput = ref('')
const nameWrapperRef = ref<HTMLElement | null>(null)
const showNameSuggestions = ref(false)
const nameActiveIndex = ref(-1)
const norm = (s: string) => (s ?? '').toLowerCase().trim()

watch(() => props.filters?.name, (v) => { nameInput.value = (v ?? '').toString() }, { immediate: true })
watch(nameInput, (v) => emit('update:filters', { ...(props.filters ?? {}), name: (v ?? '').toString() }))

const filteredNameSuggestions = computed(() => {
  const q = norm(nameInput.value)
  const src = Array.from(new Set((props.nameOptions ?? []).filter(Boolean)))
  const list = q ? src.filter(n => n.toLowerCase().includes(q)) : src
  return list.slice(0, 100)
})
const pickName = (val: string) => { nameInput.value = val; showNameSuggestions.value = false }
const moveName = (delta: number) => {
  if (!showNameSuggestions.value || filteredNameSuggestions.value.length === 0) return
  const n = filteredNameSuggestions.value.length
  nameActiveIndex.value = ((nameActiveIndex.value + delta + n) % n)
}
const onNameEnter = () => {
  if (showNameSuggestions.value && nameActiveIndex.value >= 0) pickName(filteredNameSuggestions.value[nameActiveIndex.value])
}

/* =========================
   🌟 제외할 이름 (NOT) 자동완성 & 태그 로직으로 완전 개조
========================= */
const excludedNameInput = ref('')
const excludedNameWrapperRef = ref<HTMLElement | null>(null)
const showExcludedNameSuggestions = ref(false)
const isExcludedNameComposing = ref(false)
const selectedExcludedNameTags = ref<string[]>([])
const excludedNameActiveIndex = ref(-1)

const searchInput = ref('')
const showSuggestions = ref(false)
const isComposing = ref(false)
const wrapperRef = ref<HTMLElement | null>(null)
const selectedTags = ref<string[]>([])
const synergyActiveIndex = ref(-1)

const excludedInput = ref('')
const showExcludedSuggestions = ref(false)
const isExcludedComposing = ref(false)
const excludedWrapperRef = ref<HTMLElement | null>(null)
const selectedExcludedTags = ref<string[]>([])
const excludedActiveIndex = ref(-1)

// 🌟 제외할 이름: 이미 고른 태그는 빼고 추천
const filteredExcludedNameSuggestions = computed(() => {
  const q = norm(excludedNameInput.value)
  const picked = new Set(selectedExcludedNameTags.value.map(norm))
  return Array.from(new Set((props.nameOptions ?? []).filter(Boolean)))
    .filter((s) => !picked.has(norm(s)))
    .filter((s) => (q ? norm(s).includes(q) : true))
    .slice(0, 100)
})

const filteredSuggestions = computed(() => {
  const q = norm(searchInput.value)
  const picked = new Set(selectedTags.value.map(norm))
  return (props.synergyOptions ?? [])
    .filter((s) => !picked.has(norm(s)))
    .filter((s) => (q ? norm(s).includes(q) : true))
    .slice(0, 100)
})

const filteredExcludedSuggestions = computed(() => {
  const q = norm(excludedInput.value)
  const picked = new Set(selectedExcludedTags.value.map(norm))
  return (props.synergyOptions ?? [])
    .filter((s) => !picked.has(norm(s)))
    .filter((s) => (q ? norm(s).includes(q) : true))
    .slice(0, 100)
})

// 🌟 본사 데이터와 싱크 맞추기 (제외할 이름 배열 추가)
const syncFromParent = () => {
  const src = props.filters?.synergy
  selectedTags.value = Array.isArray(src) ? [...new Set(src.filter(Boolean).map(String))] : []
  
  const srcExc = props.filters?.excludedSynergy
  selectedExcludedTags.value = Array.isArray(srcExc) ? [...new Set(srcExc.filter(Boolean).map(String))] : []

  const srcExcName = props.filters?.excludedName
  selectedExcludedNameTags.value = Array.isArray(srcExcName) ? [...new Set(srcExcName.filter(Boolean).map(String))] : []
}
watch(() => props.filters, syncFromParent, { immediate: true, deep: true })

const pushToParent = () => {
  const next = { ...(props.filters ?? {}) }
  next.synergy = [...selectedTags.value]
  next.excludedSynergy = [...selectedExcludedTags.value]
  next.excludedName = [...selectedExcludedNameTags.value] // 🌟 부모 컴포넌트로 태그 배열 발송
  emit('update:filters', next)
}

// 🌟 제외할 이름 칩(태그) 추가/제거 함수
const addExcludedNameTag = (v: string) => {
  const val = (v ?? '').trim()
  if (!val) return
  if (!selectedExcludedNameTags.value.map(norm).includes(norm(val))) {
    selectedExcludedNameTags.value.push(val)
    pushToParent()
  }
  excludedNameInput.value = ''
  showExcludedNameSuggestions.value = true
  excludedNameActiveIndex.value = -1
}
const removeExcludedNameTag = (v: string) => {
  selectedExcludedNameTags.value = selectedExcludedNameTags.value.filter((t) => norm(t) !== norm(v))
  pushToParent()
}
const moveExcludedName = (delta: number) => {
  if (!showExcludedNameSuggestions.value || filteredExcludedNameSuggestions.value.length === 0) return
  const n = filteredExcludedNameSuggestions.value.length
  excludedNameActiveIndex.value = ((excludedNameActiveIndex.value + delta + n) % n)
}
const commitExcludedNameByEnter = () => {
  if (isExcludedNameComposing.value) return
  if (showExcludedNameSuggestions.value && excludedNameActiveIndex.value >= 0) return addExcludedNameTag(filteredExcludedNameSuggestions.value[excludedNameActiveIndex.value])
  if (excludedNameInput.value.trim()) addExcludedNameTag(excludedNameInput.value)
  else if (filteredExcludedNameSuggestions.value.length) addExcludedNameTag(filteredExcludedNameSuggestions.value[0])
}
const onExcludedNameCompositionStart = () => (isExcludedNameComposing.value = true)
const onExcludedNameCompositionEnd = () => (isExcludedNameComposing.value = false)
const onExcludedNameInputDelimit = (e: Event) => {
  const v = (e.target as HTMLInputElement).value
  const parts = v.split(/[,;]+/).map((s) => s.trim()).filter(Boolean)
  if (parts.length > 1) {
    parts.forEach(addExcludedNameTag)
    excludedNameInput.value = ''
  }
}
const onExcludedNameBackspace = (e: KeyboardEvent) => {
  if ((e.target as HTMLInputElement).value === '' && selectedExcludedNameTags.value.length > 0) {
    removeExcludedNameTag(selectedExcludedNameTags.value[selectedExcludedNameTags.value.length - 1])
  }
}

// 시너지 칩 함수들
const addTag = (v: string) => {
  const val = (v ?? '').trim()
  if (!val) return
  if (!selectedTags.value.map(norm).includes(norm(val))) {
    selectedTags.value.push(val)
    pushToParent()
  }
  searchInput.value = ''
  showSuggestions.value = true
  synergyActiveIndex.value = -1
}
const removeTag = (v: string) => {
  selectedTags.value = selectedTags.value.filter((t) => norm(t) !== norm(v))
  pushToParent()
}
const moveSynergy = (delta: number) => {
  if (!showSuggestions.value || filteredSuggestions.value.length === 0) return
  const n = filteredSuggestions.value.length
  synergyActiveIndex.value = ((synergyActiveIndex.value + delta + n) % n)
}
const commitByEnter = () => {
  if (isComposing.value) return
  if (showSuggestions.value && synergyActiveIndex.value >= 0) return addTag(filteredSuggestions.value[synergyActiveIndex.value])
  if (searchInput.value.trim()) addTag(searchInput.value)
  else if (filteredSuggestions.value.length) addTag(filteredSuggestions.value[0])
}
const onCompositionStart = () => (isComposing.value = true)
const onCompositionEnd = () => (isComposing.value = false)
const onInputDelimit = (e: Event) => {
  const v = (e.target as HTMLInputElement).value
  const parts = v.split(/[,;]+|\s{2,}/).map((s) => s.trim()).filter(Boolean)
  if (parts.length > 1) {
    parts.forEach(addTag)
    searchInput.value = ''
  }
}
const onSynergyBackspace = (e: KeyboardEvent) => {
  if ((e.target as HTMLInputElement).value === '' && selectedTags.value.length > 0) {
    removeTag(selectedTags.value[selectedTags.value.length - 1])
  }
}

// 제외할 시너지 칩 함수들
const addExcludedTag = (v: string) => {
  const val = (v ?? '').trim()
  if (!val) return
  if (!selectedExcludedTags.value.map(norm).includes(norm(val))) {
    selectedExcludedTags.value.push(val)
    pushToParent()
  }
  excludedInput.value = ''
  showExcludedSuggestions.value = true
  excludedActiveIndex.value = -1
}
const removeExcludedTag = (v: string) => {
  selectedExcludedTags.value = selectedExcludedTags.value.filter((t) => norm(t) !== norm(v))
  pushToParent()
}
const moveExcluded = (delta: number) => {
  if (!showExcludedSuggestions.value || filteredExcludedSuggestions.value.length === 0) return
  const n = filteredExcludedSuggestions.value.length
  excludedActiveIndex.value = ((excludedActiveIndex.value + delta + n) % n)
}
const commitExcludedByEnter = () => {
  if (isExcludedComposing.value) return
  if (showExcludedSuggestions.value && excludedActiveIndex.value >= 0) return addExcludedTag(filteredExcludedSuggestions.value[excludedActiveIndex.value])
  if (excludedInput.value.trim()) addExcludedTag(excludedInput.value)
  else if (filteredExcludedSuggestions.value.length) addExcludedTag(filteredExcludedSuggestions.value[0])
}
const onExcludedCompositionStart = () => (isExcludedComposing.value = true)
const onExcludedCompositionEnd = () => (isExcludedComposing.value = false)
const onExcludedInputDelimit = (e: Event) => {
  const v = (e.target as HTMLInputElement).value
  const parts = v.split(/[,;]+|\s{2,}/).map((s) => s.trim()).filter(Boolean)
  if (parts.length > 1) {
    parts.forEach(addExcludedTag)
    excludedInput.value = ''
  }
}
const onExcludedBackspace = (e: KeyboardEvent) => {
  if ((e.target as HTMLInputElement).value === '' && selectedExcludedTags.value.length > 0) {
    removeExcludedTag(selectedExcludedTags.value[selectedExcludedTags.value.length - 1])
  }
}

const onDocClick = (ev: MouseEvent) => {
  const t = ev.target as Node
  if (nameWrapperRef.value && !nameWrapperRef.value.contains(t)) showNameSuggestions.value = false
  if (excludedNameWrapperRef.value && !excludedNameWrapperRef.value.contains(t)) showExcludedNameSuggestions.value = false
  if (wrapperRef.value && !wrapperRef.value.contains(t)) showSuggestions.value = false
  if (excludedWrapperRef.value && !excludedWrapperRef.value.contains(t)) showExcludedSuggestions.value = false
}
onMounted(() => document.addEventListener('click', onDocClick))
onBeforeUnmount(() => document.removeEventListener('click', onDocClick))

defineExpose({
  decodeHtmlEntities,
  getSkillText
})
</script>

<template>
  <div class="mx-auto max-w-[1280px] p-2 space-y-4 md:space-y-6">
    <div class="grid grid-cols-1 md:grid-cols-2 gap-4 md:gap-6">
      <div class="space-y-4">
        <section class="rounded-lg border border-neutral-200 dark:border-neutral-700 bg-white/95 dark:bg-neutral-900/95 shadow-sm">
          <button type="button" class="w-full px-4 py-3 flex items-center justify-between hover:bg-neutral-50 dark:hover:bg-neutral-800 rounded-t-lg transition-colors"
                  @click="toggleCollapse('skill')" :aria-expanded="collapses.skill">
            <h3 class="text-sm font-semibold text-neutral-700 dark:text-neutral-200">{{ fieldLabels?.skill || '스킬' }}</h3>
            <div class="flex items-center gap-2 text-xs text-neutral-500 dark:text-neutral-400">
              <span>선택 {{ selectedCount('skill') }}개</span>
              <ChevronDown class="w-4 h-4 transition-transform" :class="collapses.skill ? 'rotate-180' : ''"/>
            </div>
          </button>
          <div v-if="collapses.skill" class="p-4 pt-0">
            <div v-if="hasSkills"
                 class="overflow-y-auto overscroll-contain rounded-lg border border-neutral-200 dark:border-neutral-700 p-2 bg-neutral-50/80 dark:bg-neutral-800/80 max-h-56">
              <div class="grid grid-cols-4 sm:grid-cols-5 md:grid-cols-6 gap-1.5 sm:gap-2">
                <button
                  v-for="skill in props.filterOptions?.skill ?? []" :key="skill" :title="skill"
                  @click="toggleFilter('skill', skill)"
                  class="group relative inline-flex flex-col items-center justify-center gap-2 rounded-xl border py-2 text-xs font-medium select-none
                         focus:outline-none focus-visible:ring-2 focus-visible:ring-blue-500 transition-all duration-200
                         w-[70px] md:w-[74px] lg:w-[78px]"
                  :class="isSelected('skill', skill)
                    ? 'bg-blue-500 text-white border-blue-500 shadow-md hover:bg-blue-600 dark:bg-blue-700 dark:border-blue-700 dark:hover:bg-blue-600'
                    : 'bg-white dark:bg-neutral-800 text-neutral-700 dark:text-neutral-200 border-neutral-200 dark:border-neutral-600 hover:bg-blue-50 dark:hover:bg-neutral-700 hover:border-blue-300 dark:hover:border-neutral-500'">
                  <div class="w-10 h-10 sm:w-11 sm:h-11 rounded-lg"
                       :class="['bg-neutral-100 dark:bg-neutral-700', isSelected('skill', skill) ? 'ring-2 ring-blue-300 bg-white/20' : '', `bg-${matchSkillInfo(skill,'normal')}`]"/>
                  <span class="block w-full text-center font-semibold truncate">{{ skill }}</span>
                </button>
              </div>
            </div>
            <div v-else class="px-4 py-8 text-sm text-center text-neutral-500 dark:text-neutral-400">스킬 데이터가 없습니다.</div>
          </div>
        </section>

        <section class="rounded-lg border border-neutral-200 dark:border-neutral-700 bg-white/95 dark:bg-neutral-900/95 shadow-sm">
          <button type="button" class="w-full px-4 py-3 flex items-center justify-between hover:bg-neutral-50 dark:hover:bg-neutral-800 rounded-t-lg transition-colors"
                  @click="toggleCollapse('enhancedSkill')" :aria-expanded="collapses.enhancedSkill">
            <h3 class="text-sm font-semibold text-neutral-700 dark:text-neutral-200">{{ fieldLabels?.enhancedSkill || '강화 스킬' }}</h3>
            <div class="flex items-center gap-2 text-xs text-neutral-500 dark:text-neutral-400">
              <span>선택 {{ selectedCount('enhancedSkill') }}개</span>
              <ChevronDown class="w-4 h-4 transition-transform" :class="collapses.enhancedSkill ? 'rotate-180' : ''"/>
            </div>
          </button>
          <div v-if="collapses.enhancedSkill" class="p-4 pt-0">
            <div v-if="hasEnhancedSkills"
                 class="overflow-y-auto overscroll-contain rounded-lg border border-neutral-200 dark:border-neutral-700 p-2 bg-neutral-50/80 dark:bg-neutral-800/80 max-h-56">
              <div class="grid grid-cols-4 sm:grid-cols-5 md:grid-cols-6 gap-1.5 sm:gap-2">
                <button
                  v-for="skill in props.filterOptions?.enhancedSkill ?? []" :key="skill" :title="skill"
                  @click="toggleFilter('enhancedSkill', skill)"
                  class="group relative inline-flex flex-col items-center justify-center gap-2 rounded-xl border py-2 text-xs font-medium select-none
                         focus:outline-none focus-visible:ring-2 focus-visible:ring-blue-500 transition-all duration-200
                         w-[70px] md:w-[74px] lg:w-[78px]"
                  :class="isSelected('enhancedSkill', skill)
                    ? 'bg-blue-500 text-white border-blue-500 shadow-md hover:bg-blue-600 dark:bg-blue-700 dark:border-blue-700 dark:hover:bg-blue-600'
                    : 'bg-white dark:bg-neutral-800 text-neutral-700 dark:text-neutral-200 border-neutral-200 dark:border-neutral-600 hover:bg-blue-50 dark:hover:bg-neutral-700 hover:border-blue-300 dark:hover:border-neutral-500'">
                  <div class="w-10 h-10 sm:w-11 sm:h-11 rounded-lg"
                       :class="['bg-neutral-100 dark:bg-neutral-700', isSelected('enhancedSkill', skill) ? 'ring-2 ring-blue-300 bg-white/20' : '', `bg-${matchSkillInfo(skill,'enhanced')}`]"/>
                  <span class="block w-full text-center font-semibold">{{ skill }}</span>
                </button>
              </div>
            </div>
            <div v-else class="px-4 py-8 text-sm text-center text-neutral-500 dark:text-neutral-400">강화 스킬 데이터가 없습니다.</div>
          </div>
        </section>
      </div>

      <div class="space-y-4">
        <section class="rounded-lg border border-neutral-200 dark:border-neutral-700 bg-white/95 dark:bg-neutral-900/95 shadow-sm">
          <button type="button" class="w-full px-4 py-3 flex items-center justify-between hover:bg-neutral-50 dark:hover:bg-neutral-800 rounded-t-lg transition-colors"
                  @click="toggleCollapse('position')" :aria-expanded="collapses.position">
            <h3 class="text-sm font-semibold text-neutral-700 dark:text-neutral-200">{{ fieldLabels?.position || '포지션' }}</h3>
            <ChevronDown class="w-4 h-4 text-neutral-500 dark:text-neutral-400 transition-transform" :class="collapses.position ? 'rotate-180' : ''"/>
          </button>
          
          <div v-if="collapses.position" class="px-4 pb-4 space-y-4 sm:space-y-5">
            <div>
              <h4 class="text-xs font-semibold text-neutral-600 dark:text-neutral-400 mb-2">외야수 / 지명타자</h4>
              <div class="grid grid-cols-4 gap-2 sm:gap-4">
                <button v-for="pos in ['LF','CF','RF','DH']" :key="'of-'+pos"
                        @click="toggleFilter('position', pos)"
                        :class="['px-2 py-1.5 rounded-md text-sm font-bold border transition-all duration-200 focus:outline-none focus-visible:ring-2 focus-visible:ring-blue-500',
                                 isSelected('position', pos) ? 'bg-blue-500 text-white border-blue-500 shadow-md hover:bg-blue-600 dark:bg-blue-700 dark:border-blue-700 dark:hover:bg-blue-600'
                                                             : 'bg-white dark:bg-neutral-800 text-neutral-700 dark:text-neutral-200 border-neutral-200 dark:border-neutral-600 hover:bg-blue-50 dark:hover:bg-neutral-700 hover:border-blue-300 dark:hover:border-neutral-500']">
                  {{ pos }}
                </button>
              </div>
            </div>
            <div>
              <h4 class="text-xs font-semibold text-neutral-600 dark:text-neutral-400 mb-2">내야</h4>
              <div class="grid grid-cols-5 gap-2 sm:gap-3">
                <button v-for="pos in ['1B','2B','SS','3B','C']" :key="'if-'+pos"
                        @click="toggleFilter('position', pos === '1B' ? 'B1' : pos === '2B' ? 'B2' : pos === '3B' ? 'B3' : pos)"
                        :class="['px-2 py-1.5 rounded-md text-sm font-bold border transition-all duration-200 focus:outline-none focus-visible:ring-2 focus-visible:ring-blue-500',
                                 isSelected('position', (pos === '1B' ? 'B1' : pos === '2B' ? 'B2' : pos === '3B' ? 'B3' : pos)) ? 'bg-blue-500 text-white border-blue-500 shadow-md hover:bg-blue-600 dark:bg-blue-700 dark:border-blue-700 dark:hover:bg-blue-600'
                                                                                                                                   : 'bg-white dark:bg-neutral-800 text-neutral-700 dark:text-neutral-200 border-neutral-200 dark:border-neutral-600 hover:bg-blue-50 dark:hover:bg-neutral-700 hover:border-blue-300 dark:hover:border-neutral-500']">
                  {{ pos }}
                </button>
              </div>
            </div>
            <div>
              <h4 class="text-xs font-semibold text-neutral-600 dark:text-neutral-400 mb-2">선발투수 / 중간계투</h4>
              <div class="grid grid-cols-3 gap-2 sm:gap-3">
                <button v-for="pos in ['SP','RP']" :key="'pd-'+pos"
                        @click="toggleFilter('position', pos)"
                        :class="['px-2 py-1.5 rounded-md text-sm font-bold border transition-all duration-200 focus:outline-none focus-visible:ring-2 focus-visible:ring-blue-500',
                                 isSelected('position', pos) ? 'bg-blue-500 text-white border-blue-500 shadow-md hover:bg-blue-600 dark:bg-blue-700 dark:border-blue-700 dark:hover:bg-blue-600'
                                                             : 'bg-white dark:bg-neutral-800 text-neutral-700 dark:text-neutral-200 border-neutral-200 dark:border-neutral-600 hover:bg-blue-50 dark:hover:bg-neutral-700 hover:border-blue-300 dark:hover:border-neutral-500']">
                  {{ pos }}
                </button>
              </div>
            </div>
            
            <div class="mt-4 pt-4 border-t border-neutral-200 dark:border-neutral-700">
              <h4 class="text-xs font-semibold text-neutral-600 dark:text-neutral-400 mb-2">{{ fieldLabels?.pitchingFormGroup || '투구 폼 (좌우놀이)' }}</h4>
              <div class="grid grid-cols-3 gap-2 sm:gap-3">
                <button v-for="form in ['좌완', '우완', '언더·사이드']" :key="'form-'+form"
                        @click="toggleFilter('pitchingFormGroup', form)"
                        :class="['px-2 py-1.5 rounded-md text-sm font-bold border transition-all duration-200 focus:outline-none focus-visible:ring-2 focus-visible:ring-blue-500',
                                 isSelected('pitchingFormGroup', form) ? 'bg-blue-500 text-white border-blue-500 shadow-md hover:bg-blue-600 dark:bg-blue-700 dark:border-blue-700 dark:hover:bg-blue-600'
                                                                       : 'bg-white dark:bg-neutral-800 text-neutral-700 dark:text-neutral-200 border-neutral-200 dark:border-neutral-600 hover:bg-blue-50 dark:hover:bg-neutral-700 hover:border-blue-300 dark:hover:border-neutral-500']">
                  {{ form }}
                </button>
              </div>
            </div>

          </div>
        </section>

        <section class="rounded-lg border border-neutral-200 dark:border-neutral-700 bg-white/95 dark:bg-neutral-900/95 shadow-sm">
          <button type="button" class="w-full px-4 py-3 flex items-center justify-between hover:bg-neutral-50 dark:hover:bg-neutral-800 rounded-t-lg transition-colors"
                  @click="toggleCollapse('year')" :aria-expanded="collapses.year">
            <h3 class="text-sm font-semibold text-neutral-700 dark:text-neutral-200">{{ fieldLabels?.year || '연도' }}</h3>
            <ChevronDown class="w-4 h-4 text-neutral-500 dark:text-neutral-400 transition-transform" :class="collapses.year ? 'rotate-180' : ''"/>
          </button>
          <div v-if="collapses.year" class="flex-1 overflow-y-auto m-4 mt-2 rounded-lg border border-neutral-200 dark:border-neutral-700 p-2 bg-neutral-50/80 dark:bg-neutral-800/80">
            <div class="grid grid-cols-4 sm:grid-cols-6 md:grid-cols-8 lg:grid-cols-10 xl:grid-cols-12 gap-2">
              <button
                v-for="year in yearOptions" :key="year"
                @click="toggleFilter('year', year)"
                :class="[
                  'inline-flex items-center justify-center px-2.5 py-1.5 rounded-md text-xs font-medium text-center border transition-all duration-200 focus:outline-none focus-visible:ring-2 focus-visible:ring-blue-500',
                  isSelected('year', year)
                    ? 'bg-blue-500 text-white border-blue-500 shadow-md hover:bg-blue-600 dark:bg-blue-700 dark:border-blue-700 dark:hover:bg-blue-600'
                    : 'bg-white dark:bg-neutral-800 text-neutral-700 dark:text-neutral-200 border-neutral-200 dark:border-neutral-600 hover:bg-blue-50 dark:hover:bg-neutral-700 hover:border-blue-300 dark:hover:border-neutral-500'
                ]"
              >
                {{ year }}
              </button>
            </div>
          </div>
        </section>
      </div>

      <section class="col-span-full md:col-span-2 rounded-lg border border-neutral-200 dark:border-neutral-700 bg-white/95 dark:bg-neutral-900/95 shadow-sm">
        <button
          type="button"
          class="w-full px-3 md:px-4 py-3 flex items-center justify-between hover:bg-neutral-50 dark:hover:bg-neutral-800 rounded-t-lg transition-colors"
          @click="toggleCollapse('rgt')"
          :aria-expanded="collapses.rgt"
        >
          <h3 class="text-sm font-semibold text-neutral-700 dark:text-neutral-200">
            {{ (fieldLabels?.rarity || '레어도') }} · {{ (fieldLabels?.grade || '등급') }} · {{ (fieldLabels?.team || '팀') }}
          </h3>
          <ChevronDown
            class="w-4 h-4 text-neutral-500 dark:text-neutral-400 transition-transform"
            :class="collapses.rgt ? 'rotate-180' : ''"
          />
        </button>

        <div v-if="collapses.rgt" class="p-2 sm:p-3 md:p-4 pt-0">
          <section class="rounded-lg bg-white/95 dark:bg-neutral-900/95">
            <div class="divide-y divide-neutral-200 dark:divide-neutral-700">
              <div class="grid grid-cols-1 sm:grid-cols-[auto_1fr_auto] md:grid-cols-[auto_1fr_auto] items-center gap-2 sm:gap-2.5 md:gap-3 px-3 sm:px-3.5 md:px-4 py-3 sm:py-2.5 md:py-3">
                <span class="text-xs sm:text-[11px] md:text-xs font-semibold text-neutral-600 dark:text-neutral-300 sm:w-12 md:w-14 leading-tight">
                  {{ fieldLabels?.rarity || '레어도' }}
                </span>
                <div class="min-w-0 overflow-x-auto whitespace-nowrap no-scrollbar scroll-fade-x snap-x snap-mandatory touch-pan-x overscroll-x-contain -mx-1.5 sm:-mx-1 px-1.5 sm:px-1">
                  <div class="inline-flex items-center gap-2 sm:gap-1.5 md:gap-2">
                    <div
                      v-for="i in 6" :key="'star-'+i"
                      class="p-1 rounded-lg sm:rounded-md snap-start select-none pointer-events-none"
                      :title="`${i} Star`"
                    >
                      <Star
                        class="w-5 h-5 sm:w-4 sm:h-4 md:w-5 md:h-5 cursor-pointer pointer-events-auto focus:outline-none focus-visible:ring-2 focus-visible:ring-yellow-400 transition-colors duration-200"
                        :class="i <= (props.filters?.[props.rarityField!] ?? 0) ? 'text-yellow-400 hover:text-yellow-500' : 'text-neutral-300 dark:text-neutral-600 hover:text-yellow-300'"
                        fill="currentColor"
                        role="button"
                        tabindex="0"
                        :aria-pressed="i <= (props.filters?.[props.rarityField!] ?? 0)"
                        @click="update(props.rarityField!, i === (props.filters?.[props.rarityField!] ?? 0) ? '' : i)"
                        @keydown.enter.space="update(props.rarityField!, i === (props.filters?.[props.rarityField!] ?? 0) ? '' : i)"
                      />
                    </div>
                  </div>
                </div>
                <button
                  v-if="props.filters?.[props.rarityField!]"
                  @click="update(props.rarityField!, '')"
                  class="order-3 sm:order-none justify-self-end mt-2 sm:mt-1 md:mt-0 ml-0 md:ml-1 px-3 sm:px-2 py-1.5 sm:py-1 text-xs rounded-lg sm:rounded-md border border-neutral-200 dark:border-neutral-600 text-neutral-700 dark:text-neutral-200 bg-white dark:bg-neutral-800 hover:bg-neutral-50 dark:hover:bg-neutral-700 hover:border-neutral-300 dark:hover:border-neutral-500 transition-all duration-200 touch-manipulation"
                >
                  초기화
                </button>
              </div>

              <div class="px-3 sm:px-3.5 md:px-4 py-3 sm:py-2.5 md:py-3">
                <div class="md:hidden space-y-3">
                  <div class="flex items-center justify-between">
                    <span class="text-xs font-semibold text-neutral-600 dark:text-neutral-300">
                      {{ fieldLabels?.grade || '등급' }}
                    </span>
                    <button
                      @click="toggleAllGrades"
                      class="px-3 py-1.5 text-xs rounded-lg border transition-all duration-200 touch-manipulation"
                      :class="areAllGradesSelected
                        ? 'border-blue-300 text-blue-700 bg-blue-50 hover:bg-blue-100 dark:border-blue-600 dark:text-blue-300 dark:bg-blue-900/30 hover:dark:bg-blue-900/50'
                        : 'border-neutral-200 text-neutral-700 bg-white hover:bg-neutral-50 dark:border-neutral-600 dark:text-neutral-200 dark:bg-neutral-800 hover:dark:bg-neutral-700'">
                      {{ areAllGradesSelected ? '해제' : '전체' }}
                    </button>
                  </div>
                  <div class="grid grid-cols-8 gap-2">
                    <button
                      v-for="grade in visibleGrades" :key="grade"
                      @click="toggleFilter('grade', grade)"
                      :title="grade"
                      class="relative aspect-square p-1 rounded-lg border transition-all duration-200 select-none overflow-hidden flex items-center justify-center focus:outline-none"
                      :class="isSelected('grade', grade) ? 'border-blue-500 bg-blue-50 dark:border-blue-500 dark:bg-blue-900/30' : 'border-neutral-200 dark:border-neutral-600 bg-white dark:bg-neutral-800 hover:bg-neutral-50 dark:hover:bg-neutral-700'">
                      <img
                        :src="`/assets/logos/grade/${grade}.png`"
                        :alt="grade"
                        class="w-full h-full object-contain drop-shadow-sm transition-all duration-300"
                        :class="isSelected('grade', grade) ? 'grayscale-0 brightness-100 scale-105' : 'grayscale brightness-75 hover:grayscale-0 hover:brightness-100'"
                        loading="lazy"
                        @error="handleImageError"
                      />
                      <span class="hidden w-full h-full items-center justify-center text-[11px] font-bold"
                            :class="isSelected('grade', grade) ? 'text-blue-600 dark:text-blue-400' : 'text-neutral-500 dark:text-neutral-400'">{{ grade }}</span>
                    </button>
                  </div>
                </div>

                <div class="hidden md:grid md:grid-cols-[auto_1fr_auto] items-center gap-3">
                  <span class="text-xs font-semibold text-neutral-600 dark:text-neutral-300 w-14">
                    {{ fieldLabels?.grade || '등급' }}
                  </span>
                  <div class="min-w-0 overflow-x-auto whitespace-nowrap no-scrollbar scroll-fade-x snap-x snap-mandatory touch-pan-x overscroll-x-contain -mx-1 px-1">
                    <div class="inline-flex items-center gap-2">
                      <button
                        v-for="grade in visibleGrades" :key="grade"
                        @click="toggleFilter('grade', grade)"
                        :title="grade"
                        class="relative inline-flex w-16 h-16 p-1 rounded-md items-center justify-center border transition-all duration-200 select-none snap-start overflow-hidden focus:outline-none"
                        :class="isSelected('grade', grade) ? 'border-blue-500 bg-blue-50 dark:border-blue-500 dark:bg-blue-900/30' : 'border-neutral-200 dark:border-neutral-600 bg-white dark:bg-neutral-800 hover:bg-neutral-50 dark:hover:bg-neutral-700'">
                        <img
                          :src="`/assets/logos/grade/${grade}.png`"
                          :alt="grade"
                          class="w-full h-full object-contain drop-shadow-sm transition-all duration-300"
                          :class="isSelected('grade', grade) ? 'grayscale-0 brightness-100 scale-105' : 'grayscale brightness-75 hover:grayscale-0 hover:brightness-100'"
                          loading="lazy"
                          @error="handleImageError"
                        />
                        <span class="hidden w-full h-full items-center justify-center text-[13px] font-bold break-all whitespace-normal leading-tight"
                              :class="isSelected('grade', grade) ? 'text-blue-600 dark:text-blue-400' : 'text-neutral-500 dark:text-neutral-400'">{{ grade }}</span>
                      </button>
                    </div>
                  </div>
                  <button
                    @click="toggleAllGrades"
                    class="px-2 py-1 text-xs rounded-md border transition-all duration-200"
                    :class="areAllGradesSelected
                      ? 'border-blue-300 text-blue-700 bg-blue-50 hover:bg-blue-100 dark:border-blue-600 dark:text-blue-300 dark:bg-blue-900/30 hover:dark:bg-blue-900/50'
                      : 'border-neutral-200 text-neutral-700 bg-white hover:bg-neutral-50 dark:border-neutral-600 dark:text-neutral-200 dark:bg-neutral-800 hover:dark:bg-neutral-700'">
                    {{ areAllGradesSelected ? '해제' : '전체' }}
                  </button>
                </div>
              </div>

              <div class="px-3 sm:px-3.5 md:px-4 py-3 sm:py-2.5 md:py-3">
                <div class="md:hidden space-y-3">
                  <div class="flex items-center justify-between">
                    <span class="text-xs font-semibold text-neutral-600 dark:text-neutral-300">
                      {{ fieldLabels?.team || '팀' }}
                    </span>
                    <button
                      @click="toggleAllTeams"
                      class="px-3 py-1.5 text-xs rounded-lg border transition-all duration-200 touch-manipulation"
                      :class="allTeamsSelected
                        ? 'border-blue-300 text-blue-700 bg-blue-50 hover:bg-blue-100 dark:border-blue-600 dark:text-blue-300 dark:bg-blue-900/30 hover:dark:bg-blue-900/50'
                        : 'border-neutral-200 text-neutral-700 bg-white hover:bg-neutral-50 dark:border-neutral-600 dark:text-neutral-200 dark:bg-neutral-800 hover:dark:bg-neutral-700'">
                      {{ allTeamsSelected ? '해제' : '전체' }}
                    </button>
                  </div>
                  <div class="grid grid-cols-8 gap-2">
                    <button
                      v-for="team in props.filterOptions?.team ?? []" :key="team"
                      @click="toggleFilter('team', team)"
                      :title="team"
                      class="relative aspect-square p-1.5 rounded-lg border transition-all duration-200 select-none overflow-hidden flex items-center justify-center focus:outline-none"
                      :class="isSelected('team', team) ? 'border-blue-500 bg-blue-50 dark:border-blue-500 dark:bg-blue-900/30' : 'border-neutral-200 dark:border-neutral-600 bg-white dark:bg-neutral-800 hover:bg-neutral-50 dark:hover:bg-neutral-700'">
                      <img
                        :src="teamLogos[team]" :alt="team"
                        class="w-full h-full object-contain drop-shadow-sm transition-all duration-300"
                        :class="isSelected('team', team) ? 'grayscale-0 brightness-100 scale-105' : 'grayscale brightness-75 hover:grayscale-0 hover:brightness-100'"
                        loading="lazy"
                        @error="handleImageError"
                      />
                      <span class="hidden w-full h-full items-center justify-center text-[10px] font-bold text-center break-all whitespace-normal leading-tight"
                            :class="isSelected('team', team) ? 'text-blue-600 dark:text-blue-400' : 'text-neutral-500 dark:text-neutral-400'">{{ team }}</span>
                    </button>
                  </div>
                </div>

                <div class="hidden md:grid md:grid-cols-[auto_1fr_auto] items-center gap-3">
                  <span class="text-xs font-semibold text-neutral-600 dark:text-neutral-300 w-14">
                    {{ fieldLabels?.team || '팀' }}
                  </span>
                  <div class="min-w-0 overflow-x-auto whitespace-nowrap no-scrollbar scroll-fade-x snap-x snap-mandatory touch-pan-x overscroll-x-contain -mx-1 px-1">
                    <div class="inline-flex items-center gap-2">
                      <button
                        v-for="team in props.filterOptions?.team ?? []" :key="team"
                        @click="toggleFilter('team', team)"
                        :title="team"
                        class="relative inline-flex w-10 h-10 p-1 rounded-md items-center justify-center border transition-all duration-200 select-none snap-start overflow-hidden focus:outline-none"
                        :class="isSelected('team', team) ? 'border-blue-500 bg-blue-50 dark:border-blue-500 dark:bg-blue-900/30' : 'border-neutral-200 dark:border-neutral-600 bg-white dark:bg-neutral-800 hover:bg-neutral-50 dark:hover:bg-neutral-700'">
                        <img
                          :src="teamLogos[team]" :alt="team"
                          class="w-8 h-8 object-contain drop-shadow-sm transition-all duration-300"
                          :class="isSelected('team', team) ? 'grayscale-0 brightness-100 scale-105' : 'grayscale brightness-75 hover:grayscale-0 hover:brightness-100'"
                          loading="lazy"
                          @error="handleImageError"
                        />
                        <span class="hidden w-full h-full items-center justify-center text-[9px] font-bold text-center break-all whitespace-normal leading-tight"
                              :class="isSelected('team', team) ? 'text-blue-600 dark:text-blue-400' : 'text-neutral-500 dark:text-neutral-400'">{{ team }}</span>
                      </button>
                    </div>
                  </div>
                  <button
                    @click="toggleAllTeams"
                    class="px-2 py-1 text-xs rounded-md border transition-all duration-200"
                    :class="allTeamsSelected
                      ? 'border-blue-300 text-blue-700 bg-blue-50 hover:bg-blue-100 dark:border-blue-600 dark:text-blue-300 dark:bg-blue-900/30 hover:dark:bg-blue-900/50'
                      : 'border-neutral-200 text-neutral-700 bg-white hover:bg-neutral-50 dark:border-neutral-600 dark:text-neutral-200 dark:bg-neutral-800 hover:dark:bg-neutral-700'">
                    {{ allTeamsSelected ? '해제' : '전체' }}
                  </button>
                </div>
              </div>
            </div>
          </section>
        </div>
      </section>
    </div>

    <section class="rounded-lg border border-neutral-200 dark:border-neutral-700 bg-white/95 dark:bg-neutral-900/95 shadow-sm p-3 md:p-4">
      <div class="grid grid-cols-1 md:grid-cols-[1fr_1fr_1fr_1fr_auto] gap-3 md:gap-4 items-start">
        
        <div>
          <label class="block text-xs font-semibold text-neutral-600 dark:text-neutral-300 mb-1">
            {{ fieldLabels?.name || '이름 (부분 일치)' }}
          </label>
          <div ref="nameWrapperRef" class="relative">
            <input
              v-model="nameInput"
              @focus="showNameSuggestions = true"
              @keydown.down.prevent="moveName(1)"
              @keydown.up.prevent="moveName(-1)"
              @keydown.enter.prevent="onNameEnter"
              @keydown.esc="showNameSuggestions = false"
              placeholder="선수명을 입력해 주세요."
              class="w-full bg-white dark:bg-neutral-800 border border-neutral-200 dark:border-neutral-600 rounded-md px-3 py-2 pr-10 text-sm text-neutral-900 dark:text-neutral-100 placeholder-neutral-400 dark:placeholder-neutral-500 shadow-sm focus:outline-none focus:ring-2 focus:ring-blue-500 focus:border-blue-500 transition-colors"
              role="combobox" aria-expanded="showNameSuggestions" aria-controls="name-suggest"
            />
            <Search class="absolute right-3 top-1/2 -translate-y-1/2 w-4 h-4 text-neutral-400 dark:text-neutral-500 pointer-events-none" />
            <ul
              v-if="showNameSuggestions && filteredNameSuggestions.length"
              id="name-suggest"
              class="absolute z-10 w-full mt-1 bg-white dark:bg-neutral-800 border border-neutral-200 dark:border-neutral-600 rounded-md shadow-lg max-h-56 md:max-h-64 overflow-y-auto"
              role="listbox"
            >
              <li
                v-for="(suggestion, idx) in filteredNameSuggestions"
                :key="suggestion"
                @mousedown.prevent="pickName(suggestion)"
                @mouseenter="nameActiveIndex = idx"
                class="px-3 py-2 text-sm cursor-pointer hover:bg-blue-50 dark:hover:bg-blue-900/20 transition-colors"
                :class="idx === nameActiveIndex ? 'bg-blue-50 dark:bg-blue-900/20' : ''"
                role="option"
                :aria-selected="idx === nameActiveIndex"
              >
                {{ suggestion }}
              </li>
            </ul>
          </div>
        </div>

        <!-- 🌟 새롭게 추가된 '제외할 이름 (NOT)' 태그 방식 -->
        <div>
          <label class="block text-xs font-semibold text-rose-600 dark:text-rose-400 mb-1">
            {{ fieldLabels?.excludedName || '제외할 이름 (NOT)' }}
          </label>
          <div ref="excludedNameWrapperRef" class="relative">
            <input
              v-model="excludedNameInput"
              @input="onExcludedNameInputDelimit"
              @focus="showExcludedNameSuggestions = true"
              @keydown.down.prevent="moveExcludedName(1)"
              @keydown.up.prevent="moveExcludedName(-1)"
              @keydown.enter.prevent="commitExcludedNameByEnter"
              @keydown.esc="showExcludedNameSuggestions = false"
              @keydown.backspace="onExcludedNameBackspace"
              @compositionstart="onExcludedNameCompositionStart"
              @compositionend="onExcludedNameCompositionEnd"
              placeholder="제외할 선수명을 입력해 주세요."
              class="w-full bg-white dark:bg-neutral-800 border border-neutral-200 dark:border-neutral-600 rounded-md px-3 py-2 pr-10 text-sm text-neutral-900 dark:text-neutral-100 placeholder-neutral-400 dark:placeholder-neutral-500 shadow-sm focus:outline-none focus:ring-2 focus:ring-rose-500 focus:border-rose-500 transition-colors"
              role="combobox" aria-expanded="showExcludedNameSuggestions" aria-controls="excluded-name-suggest"
            />
            <Search class="absolute right-3 top-1/2 -translate-y-1/2 w-4 h-4 text-rose-400 dark:text-rose-500 pointer-events-none" />
            <ul
              v-if="showExcludedNameSuggestions && filteredExcludedNameSuggestions.length"
              id="excluded-name-suggest"
              class="absolute z-10 w-full mt-1 bg-white dark:bg-neutral-800 border border-rose-200 dark:border-rose-800 rounded-md shadow-lg max-h-56 md:max-h-64 overflow-y-auto"
              role="listbox"
            >
              <li
                v-for="(suggestion, idx) in filteredExcludedNameSuggestions"
                :key="suggestion"
                @mousedown.prevent="addExcludedNameTag(suggestion)"
                @mouseenter="excludedNameActiveIndex = idx"
                class="px-3 py-2 text-sm cursor-pointer hover:bg-rose-50 dark:hover:bg-rose-900/20 transition-colors"
                :class="idx === excludedNameActiveIndex ? 'bg-rose-50 dark:bg-rose-900/20' : ''"
                role="option"
                :aria-selected="idx === excludedNameActiveIndex"
              >
                {{ suggestion }}
              </li>
            </ul>
          </div>
          <!-- 제외할 이름 태그 뱃지 -->
          <div class="flex flex-wrap gap-2 mt-2">
            <span
              v-for="tag in selectedExcludedNameTags" :key="tag"
              class="flex items-center gap-1 px-2 py-1 text-xs bg-rose-100 text-rose-800 dark:bg-rose-900/30 dark:text-rose-200 rounded-full transition-colors border border-rose-200 dark:border-rose-800"
            >
              {{ tag }}
              <button @click="removeExcludedNameTag(tag)" class="text-rose-600 hover:text-rose-800 dark:text-rose-300 dark:hover:text-rose-100 transition-colors font-black" aria-label="태그 제거">&times;</button>
            </span>
          </div>
        </div>

        <div>
          <label class="block text-xs font-semibold text-neutral-600 dark:text-neutral-300 mb-1">
            {{ fieldLabels?.synergy || '시너지 (정확 일치 AND)' }}
          </label>
          <div ref="wrapperRef" class="relative">
            <input
              v-model="searchInput"
              @input="onInputDelimit"
              @focus="showSuggestions = true"
              @keydown.down.prevent="moveSynergy(1)"
              @keydown.up.prevent="moveSynergy(-1)"
              @keydown.enter.prevent="commitByEnter"
              @keydown.esc="showSuggestions = false"
              @keydown.backspace="onSynergyBackspace"
              @compositionstart="onCompositionStart"
              @compositionend="onCompositionEnd"
              placeholder="포함할 시너지를 입력해 주세요."
              class="w-full bg-white dark:bg-neutral-800 border border-neutral-200 dark:border-neutral-600 rounded-md px-3 py-2 pr-10 text-sm text-neutral-900 dark:text-neutral-100 placeholder-neutral-400 dark:placeholder-neutral-500 shadow-sm focus:outline-none focus:ring-2 focus:ring-blue-500 focus:border-blue-500 transition-colors"
              role="combobox" aria-expanded="showSuggestions" aria-controls="syn-suggest"
            />
            <Search class="absolute right-3 top-1/2 -translate-y-1/2 w-4 h-4 text-neutral-400 dark:text-neutral-500 pointer-events-none" />
            <ul
              v-if="showSuggestions && filteredSuggestions.length"
              id="syn-suggest"
              class="absolute z-10 w-full mt-1 bg-white dark:bg-neutral-800 border border-neutral-200 dark:border-neutral-600 rounded-md shadow-lg max-h-56 md:max-h-64 overflow-y-auto"
              role="listbox"
            >
              <li
                v-for="(suggestion, idx) in filteredSuggestions"
                :key="suggestion"
                @mousedown.prevent="addTag(suggestion)"
                @mouseenter="synergyActiveIndex = idx"
                class="px-3 py-2 text-sm cursor-pointer hover:bg-blue-50 dark:hover:bg-blue-900/20 transition-colors"
                :class="idx === synergyActiveIndex ? 'bg-blue-50 dark:bg-blue-900/20' : ''"
                role="option"
                :aria-selected="idx === synergyActiveIndex"
              >
                {{ suggestion }}
              </li>
            </ul>
          </div>
          <div class="flex flex-wrap gap-2 mt-2">
            <span
              v-for="tag in selectedTags" :key="tag"
              class="flex items-center gap-1 px-2 py-1 text-xs bg-blue-100 text-blue-800 dark:bg-blue-900/30 dark:text-blue-200 rounded-full transition-colors"
            >
              {{ tag }}
              <button @click="removeTag(tag)" class="text-blue-600 hover:text-blue-800 dark:text-blue-300 dark:hover:text-blue-100 transition-colors" aria-label="태그 제거">&times;</button>
            </span>
          </div>
        </div>

        <div>
          <label class="block text-xs font-semibold text-rose-600 dark:text-rose-400 mb-1">
            {{ fieldLabels?.excludedSynergy || '제외할 시너지 (NOT)' }}
          </label>
          <div ref="excludedWrapperRef" class="relative">
            <input
              v-model="excludedInput"
              @input="onExcludedInputDelimit"
              @focus="showExcludedSuggestions = true"
              @keydown.down.prevent="moveExcluded(1)"
              @keydown.up.prevent="moveExcluded(-1)"
              @keydown.enter.prevent="commitExcludedByEnter"
              @keydown.esc="showExcludedSuggestions = false"
              @keydown.backspace="onExcludedBackspace"
              @compositionstart="onExcludedCompositionStart"
              @compositionend="onExcludedCompositionEnd"
              placeholder="제외할 시너지를 입력해 주세요."
              class="w-full bg-white dark:bg-neutral-800 border border-neutral-200 dark:border-neutral-600 rounded-md px-3 py-2 pr-10 text-sm text-neutral-900 dark:text-neutral-100 placeholder-neutral-400 dark:placeholder-neutral-500 shadow-sm focus:outline-none focus:ring-2 focus:ring-rose-500 focus:border-rose-500 transition-colors"
              role="combobox" aria-expanded="showExcludedSuggestions" aria-controls="exc-suggest"
            />
            <Search class="absolute right-3 top-1/2 -translate-y-1/2 w-4 h-4 text-rose-400 dark:text-rose-500 pointer-events-none" />
            <ul
              v-if="showExcludedSuggestions && filteredExcludedSuggestions.length"
              id="exc-suggest"
              class="absolute z-10 w-full mt-1 bg-white dark:bg-neutral-800 border border-rose-200 dark:border-rose-800 rounded-md shadow-lg max-h-56 md:max-h-64 overflow-y-auto"
              role="listbox"
            >
              <li
                v-for="(suggestion, idx) in filteredExcludedSuggestions"
                :key="suggestion"
                @mousedown.prevent="addExcludedTag(suggestion)"
                @mouseenter="excludedActiveIndex = idx"
                class="px-3 py-2 text-sm cursor-pointer hover:bg-rose-50 dark:hover:bg-rose-900/20 transition-colors"
                :class="idx === excludedActiveIndex ? 'bg-rose-50 dark:bg-rose-900/20' : ''"
                role="option"
                :aria-selected="idx === excludedActiveIndex"
              >
                {{ suggestion }}
              </li>
            </ul>
          </div>
          <div class="flex flex-wrap gap-2 mt-2">
            <span
              v-for="tag in selectedExcludedTags" :key="tag"
              class="flex items-center gap-1 px-2 py-1 text-xs bg-rose-100 text-rose-800 dark:bg-rose-900/30 dark:text-rose-200 rounded-full transition-colors border border-rose-200 dark:border-rose-800"
            >
              {{ tag }}
              <button @click="removeExcludedTag(tag)" class="text-rose-600 hover:text-rose-800 dark:text-rose-300 dark:hover:text-rose-100 transition-colors font-black" aria-label="태그 제거">&times;</button>
            </span>
          </div>
        </div>

        <div class="flex flex-col h-full md:mt-0 mt-2">
          <label class="block text-[11px] font-bold text-transparent select-none mb-1 hidden md:block">필터</label>
          <button 
            type="button"
            @click="update('hsUniSynergyOnly', !props.filters?.hsUniSynergyOnly)"
            class="flex items-center justify-center gap-1.5 h-[38px] px-3 md:px-4 rounded-md border transition-colors shadow-sm whitespace-nowrap focus:outline-none focus:ring-2 focus:ring-indigo-500"
            :class="props.filters?.hsUniSynergyOnly ? 'ring-2 ring-indigo-500 bg-indigo-50 dark:bg-indigo-900/30 border-indigo-300 dark:border-indigo-600' : 'border-neutral-200 dark:border-neutral-600 bg-neutral-50 dark:bg-neutral-800 hover:bg-neutral-100 dark:hover:bg-neutral-700'"
          >
            <div class="w-3.5 h-3.5 flex items-center justify-center rounded border"
                 :class="props.filters?.hsUniSynergyOnly ? 'bg-indigo-600 border-indigo-600 text-white' : 'border-neutral-300 bg-white dark:bg-neutral-900 dark:border-neutral-500'">
              <Check v-if="props.filters?.hsUniSynergyOnly" class="w-3 h-3" stroke-width="3" />
            </div>
            <span class="text-xs font-bold" :class="props.filters?.hsUniSynergyOnly ? 'text-indigo-700 dark:text-indigo-300' : 'text-neutral-600 dark:text-neutral-300'">
              고교+대학 보유
            </span>
          </button>
        </div>

      </div>
    </section>

  </div>
</template>

<style scoped>
.acc-enter-active, .acc-leave-active {
  transition: transform .15s ease, opacity .15s ease;
  transform-origin: top;
}
.acc-enter-from, .acc-leave-to {
  opacity: 0;
  transform: scaleY(.98);
}
.drop-glow {
  filter: drop-shadow(0 0 6px rgba(250, 204, 21, 0.45));
}
.theme-switching *, .theme-switching *::before, .theme-switching *::after {
  transition: none !important;
  animation: none !important;
}
select::-webkit-scrollbar,
.overflow-y-auto::-webkit-scrollbar { width: 8px; height: 8px; }
select::-webkit-scrollbar-track,
.overflow-y-auto::-webkit-scrollbar-track { background: rgba(148,163,184,0.15); border-radius: 4px; }
select::-webkit-scrollbar-thumb,
.overflow-y-auto::-webkit-scrollbar-thumb { background: rgba(100,116,139,0.35); border-radius: 4px; }
select::-webkit-scrollbar-thumb:hover,
.overflow-y-auto::-webkit-scrollbar-thumb:hover { background: rgba(100,116,139,0.55); }
button, select { cursor: pointer; }
input[type="text"] { cursor: text; }
:focus { outline: none; }
input:hover, select:hover { border-color: #9ca3af; }
.space-y-1 > label { transition: color .2s ease; }
@media (prefers-reduced-motion: reduce) {
  * { transition: none !important; transform: none !important; }
}
</style>
