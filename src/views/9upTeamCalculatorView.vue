<script setup lang="ts">
import { ref, computed, onMounted, onBeforeUnmount, reactive, watch } from 'vue'
import Papa from 'papaparse'
// 상단 lucide-vue-next 임포트 목록에 RefreshCw 추가
import { Search, Calculator, Star, Shield, Zap, TrendingUp, X, Users, ArrowUpCircle, Sparkles, UserCheck, Filter, ChevronRight as ChevronRightIcon, Check, Save, FolderOpen, Download, Upload, Camera, RefreshCw } from 'lucide-vue-next'
import { createWorker } from 'tesseract.js'
  
type Raw = Record<string, any>
type CountOp = '==' | '>=' | '<=' | '>' | '<' | 'between'

interface JsonBonus { unit: 'percent' | 'fixed'; value: number }
interface JsonCond  { count: any; stat: string; bonus: JsonBonus }
interface JsonSynergy { id: number | string; synergy: string; conditions: JsonCond[] }
interface PlayerBuff {
  enhancementLevel: number; breakthroughLevel: number; careerTeamCount: number;
  hitAceBuff: number; imprintStarterPower: number;
  careerAllStatFlat: number; 
  imprintCoreStat: number; // 🌟 추가: 각인 전체 능력치 (5대 스탯)
  careerCoreStat: number;  // 🌟 추가: 커리어 전체 능력치 (5대 스탯)
  selectedSkills: string[];
  battingOrder: number | null;
  playerLevel: number; collectionBuff: number; careerLevelBuff: number;
  binderBuff: number; ultimateImprintPercent: number;
  imprintStats: Record<string, number>;
  careerStats: Record<string, number>;
}
// ========================================================
// 🌟 9up 인게임 고증: 커리어 장착 시스템 엔진 🌟
// ========================================================
type CareerGrade = '루키' | '엘리트' | '프로' | '마스터';
interface CareerSlot { grade: CareerGrade; statType: string; value: number; }

const showCareerManager = ref(false);
// 🌟 타순 일괄 변경 엔진 및 상태
const showBattingOrderManager = ref(false);
const batterLineupPositions = ['LF', 'CF', 'RF', '3B', 'SS', '2B', '1B', 'C', 'DH'];

const sortedBattersForOrder = computed(() => {
   const filled = batterLineupPositions.filter(pos => lineup.value[pos] && playerBuffs.value[pos]);
   return filled.sort((a, b) => {
      const orderA = playerBuffs.value[a].battingOrder || 99;
      const orderB = playerBuffs.value[b].battingOrder || 99;
      return orderA - orderB;
   });
});

const handleBattingOrderChange = (pos: string, newVal: number | null) => {
  if (!playerBuffs.value[pos]) return;
  if (newVal === null) {
    playerBuffs.value[pos].battingOrder = null;
    return;
  }
  const existingPos = batterLineupPositions.find(k => k !== pos && playerBuffs.value[k]?.battingOrder === newVal);
  const oldOrder = playerBuffs.value[pos].battingOrder;
  
  if (existingPos && playerBuffs.value[existingPos]) {
     playerBuffs.value[existingPos].battingOrder = oldOrder;
  }
  playerBuffs.value[pos].battingOrder = newVal;
};
const KOR_STAT_MAP: Record<string, string> = {
  '컨택': 'contact', '갭파워': 'gapPower', '홈런파워': 'homeRunPower', '선구': 'plateDiscipline', '삼진회피': 'strikeoutAvoidance',
  '무브먼트': 'movement', '장타억제': 'longHitSuppression', '홈런억제': 'homeRunSuppression', '컨트롤': 'control', '스터프': 'stuff'
};
// 🌟 타순 변경 모달 전용 드래그 앤 드롭 엔진
const onBattingOrderDragStart = (e: DragEvent, pos: string) => {
  e.dataTransfer?.setData('text/plain', pos);
  if (e.dataTransfer) e.dataTransfer.effectAllowed = 'move';
};

const onBattingOrderDrop = (e: DragEvent, targetPos: string) => {
  const sourcePos = e.dataTransfer?.getData('text/plain');
  if (!sourcePos || sourcePos === targetPos) return;

  // 두 선수의 타순 번호를 쏙 빼서 서로 맞교환(Swap)
  const sourceOrder = playerBuffs.value[sourcePos]?.battingOrder;
  const targetOrder = playerBuffs.value[targetPos]?.battingOrder;

  if (playerBuffs.value[sourcePos]) playerBuffs.value[sourcePos].battingOrder = targetOrder || null;
  if (playerBuffs.value[targetPos]) playerBuffs.value[targetPos].battingOrder = sourceOrder || null;
};

const getAvailableCareerStatTypes = (p: Raw | null, grade: string) => {
  if (!p) return [];
  const base = isPitcher(p) ? ['무브먼트', '장타억제', '홈런억제', '컨트롤', '스터프'] : ['컨택', '갭파워', '홈런파워', '선구', '삼진회피'];
  base.push('전체 능력치');
  if (grade === '마스터') base.push('동일팀 파워');
  return base;
}

const getAvailableCareerValues = (grade: string, statType: string) => {
  if (!statType || statType === '동일팀 파워') return [];
  const isCore = statType !== '전체 능력치';
  if (grade === '마스터') return isCore ? [60, 55, 46] : [14, 13, 12];
  if (grade === '프로') return isCore ? [41, 37, 32] : [10, 9, 8];
  if (grade === '엘리트') return isCore ? [28, 23, 19] : [7, 6, 5];
  if (grade === '루키') return isCore ? [14, 10, 5] : [4, 3, 2];
  return [];
}

const onCareerChange = (c: any) => {
  if (c.statType === '동일팀 파워' && c.grade !== '마스터') c.statType = '';
  const validValues = getAvailableCareerValues(c.grade, c.statType);
  if (validValues.length > 0 && !validValues.includes(c.value)) c.value = validValues[0];
  else if (validValues.length === 0) c.value = 0;
}

const getCareerGradeColor = (g: string) => {
  if (g === '마스터') return 'text-purple-600';
  if (g === '프로') return 'text-blue-600';
  if (g === '엘리트') return 'text-emerald-600';
  return 'text-neutral-500';
}

const setAllCareersToMaster = () => {
  if (!selectedSlot.value || !playerBuffs.value[selectedSlot.value]) return;
  if (!playerBuffs.value[selectedSlot.value].careers) return;
  playerBuffs.value[selectedSlot.value].careers.forEach(c => {
      c.grade = '마스터';
      onCareerChange(c);
  });
}

const getCareerSetEffectText = (careers: CareerSlot[] | undefined) => {
  if (!careers) return '';
  const counts: Record<string, number> = {};
  careers.forEach(c => { if(c.statType) counts[c.statType] = (counts[c.statType]||0) + 1; });
  const effects: string[] = [];
  Object.entries(counts).forEach(([type, count]) => {
      if (count >= 3) {
          if (type === '전체 능력치') effects.push(`전능 +${count === 6 ? 30 : count * 6}`);
          else if (type === '동일팀 파워') effects.push(`동일팀파워 +${count}배`);
          else effects.push(`${type} +${count * 25}`);
      }
  });
  return effects.join(', ');
}
// ========================================================
// 🌟 커스텀 토스트 알림 시스템 (alert 대체) 🌟
interface Toast { id: number; msg: string; type: 'success' | 'error' | 'info'; }
const toasts = ref<Toast[]>([]);
let toastCounter = 0;
const showToast = (msg: string, type: 'success' | 'error' | 'info' = 'info') => {
  const id = toastCounter++;
  toasts.value.push({ id, msg, type });
  setTimeout(() => {
    toasts.value = toasts.value.filter(t => t.id !== id);
  }, 3000); // 3초 후 자동 삭제
};

// 🌟 1. 등급 맵핑 함수 (grade 필터 버그 및 이미지 출력 공통 사용)
const getMappedGrade = (grade: unknown) => {
  if (!grade) return '';
  const g = String(grade).toUpperCase();
  const map: Record<string, string> = {
    'DIGNITY':'DGN', '디그니티':'DGN', 
    'TOP CLASS':'TOP', '탑클래스':'TOP', 
    'GOLDEN GLOVE':'GG', '골든글러브':'GG', '골글':'GG', 
    'ACE PITCHER':'ACE', '에이스':'ACE', 
    'HIT BATTER':'HIT', '히트':'HIT', 
    'TEAM PLAYER':'TEA', '팀플':'TEA',
    'MONTHLY MVP':'MMVP', '월간MVP':'MMVP', '월간':'MMVP', 
    'ROOKIE OF THE YEAR':'ROY', '신인왕':'ROY', 
    'GG OF THE YEAR':'GGY', '연도골글':'GGY', '연글':'GGY',
    'NATIONAL TEAM':'NT', '국가대표':'NT', 
    'ALLSTAR':'ASG', '올스타':'ASG', 
    'SEASON':'SEA', '시즌':'SEA', 
    'POST SEASON':'POS', '포스트시즌':'POS'
  };
  return map[g] || g;
}

// 엑스박스 방지용 이미지 경로 로더
const getGradeImage = (grade: unknown) => {
  const mappedGrade = getMappedGrade(grade);
  return mappedGrade ? `/assets/logos/grade/${mappedGrade}.png` : '';
}
  
const POSITION_ALIASES: Record<string, string> = {
  'b1': '1B', '1b': '1B', '1': '1B', '1루': '1B',
  'b2': '2B', '2b': '2B', '2': '2B', '2루': '2B',
  'b3': '3B', '3b': '3B', '3': '3B', '3루': '3B',
  'c': 'C', '포': 'C', 'ss': 'SS', '유격': 'SS',
  'lf': 'LF', '좌익': 'LF', 'cf': 'CF', '중견': 'CF', 'rf': 'RF', '우익': 'RF',
  'sp': 'SP', '선발': 'SP', 'rp': 'RP', '불펜': 'RP', 'dh': 'DH', '지타': 'DH',
}

const STAT_LABELS: Record<string, string> = {
  contact: '컨택트', gapPower: '갭파워', homeRunPower: '홈런파워', plateDiscipline: '선구', strikeoutAvoidance: '삼진회피',
  stealing: '도루', baseRunning: '주루', defense: '수비',
  movement: '무브먼트', longHitSuppression: '장타억제', homeRunSuppression: '홈런억제', control: '컨트롤', stuff: '스터프',
  runnerControl: '주자견제', pitchLimit: '한계투구'
}

const batterStats = ['contact', 'gapPower', 'homeRunPower', 'plateDiscipline', 'strikeoutAvoidance', 'stealing', 'baseRunning', 'defense'];
const pitcherStats = ['movement', 'longHitSuppression', 'homeRunSuppression', 'control', 'stuff', 'defense', 'pitchLimit', 'runnerControl'];

// 🌟 레이더 차트 및 스탯 박스 표시용 (인게임 순서 완벽 고증)
const radarBatterStats = ['contact', 'homeRunPower', 'strikeoutAvoidance', 'plateDiscipline', 'gapPower'];
const radarPitcherStats = ['movement', 'homeRunSuppression', 'stuff', 'control', 'longHitSuppression'];

// 🌟 레이더 차트 (오각형) 그리기 도우미 함수들 (인게임 2000 고정 스케일)
const RADAR_CENTER = 100;
const RADAR_RADIUS = 60; // 2000 스탯일 때의 기준 반지름
const RADAR_ANGLES = [0, 72, 144, 216, 288];

const getRadarWebPoints = (level: number) => {
  const r = (RADAR_RADIUS / 4) * level; 
  return RADAR_ANGLES.map(angle => {
    const rad = (angle - 90) * (Math.PI / 180);
    return `${RADAR_CENTER + r * Math.cos(rad)},${RADAR_CENTER + r * Math.sin(rad)}`;
  }).join(' ');
};

const getRadarStatPoints = (slot: string) => {
  if (!slot || !computedPlayerStats.value[slot]) return '';
  const p = lineups.value[activeDeck.value][slot];
  if (!p) return '';
  const isPit = isPitcher(p);
  const coreStats = isPit ? radarPitcherStats : radarBatterStats;
  const stats = computedPlayerStats.value[slot].stats;
  const values = coreStats.map(s => stats[s] || 0);

  return values.map((val, i) => {
    const r = (val / 2000) * RADAR_RADIUS; 
    const rad = (RADAR_ANGLES[i] - 90) * (Math.PI / 180);
    return `${RADAR_CENTER + r * Math.cos(rad)},${RADAR_CENTER + r * Math.sin(rad)}`;
  }).join(' ');
};

const getRadarStatDots = (slot: string) => {
  if (!slot || !computedPlayerStats.value[slot]) return [];
  const p = lineups.value[activeDeck.value][slot];
  if (!p) return [];
  const isPit = isPitcher(p);
  const coreStats = isPit ? radarPitcherStats : radarBatterStats;
  const stats = computedPlayerStats.value[slot].stats;
  const values = coreStats.map(s => stats[s] || 0);

  return values.map((val, i) => {
    const r = (val / 2000) * RADAR_RADIUS;
    const rad = (RADAR_ANGLES[i] - 90) * (Math.PI / 180);
    return { x: RADAR_CENTER + r * Math.cos(rad), y: RADAR_CENTER + r * Math.sin(rad) };
  });
};

const getRadarLabels = (slot: string) => {
  if (!slot || !computedPlayerStats.value[slot]) return [];
  const p = lineups.value[activeDeck.value][slot];
  if (!p) return [];
  const isPit = isPitcher(p);
  const coreStats = isPit ? radarPitcherStats : radarBatterStats;
  const stats = computedPlayerStats.value[slot].stats;

  return coreStats.map((s, i) => {
    const val = stats[s] || 0;
    const r = (val / 2000) * RADAR_RADIUS;
    const labelRadius = Math.max(RADAR_RADIUS, r) + 18;

    const rad = (RADAR_ANGLES[i] - 90) * (Math.PI / 180);
    let x = RADAR_CENTER + labelRadius * Math.cos(rad);
    let y = RADAR_CENTER + labelRadius * Math.sin(rad);
    
    if (i === 0) y -= 2;
    if (i === 2 || i === 3) y += 6;
    
    return { 
      text: `${STAT_LABELS[s] || s} ${stats[s] || 0}`, 
      x, 
      y 
    };
  });
};


const isPitcher = (p: Raw | null) => {
  if (!p) return false;
  const pos = String(p.position || '').toUpperCase();
  return pos.includes('SP') || pos.includes('RP') || !!p.movement;
}

const isLoading = ref(true)
const players = ref<Raw[]>([])
const synergys = ref<JsonSynergy[]>([])
const teamData = ref<any[]>([])

const advancedFilterOpen = ref(false)
const isSynergyDropdownOpen = ref(false)
const synergyWrapperRef = ref<HTMLElement | null>(null)

const onDocClick = (ev: MouseEvent) => {
  const t = ev.target as Node
  if (synergyWrapperRef.value && !synergyWrapperRef.value.contains(t)) {
    isSynergyDropdownOpen.value = false
  }
}

onMounted(() => {
  document.addEventListener('click', onDocClick)
})

onBeforeUnmount(() => {
  document.removeEventListener('click', onDocClick)
})

const currentPage = ref(1)
const pageSize = 50
const synergySearchText = ref('')
const synergyOptions = ref<string[]>([])

const expandedSynergy = ref<string | null>(null)
const expandedPendingSynergy = ref<string | null>(null)

const searchQuery = reactive({
  search: '', position: [] as string[], team: [] as string[],
  synergy: [] as string[], skill: [] as string[], rarity: null as number | null, grade: [] as string[]
})

const lineupViewMode = ref('batter')
const selectedSlot = ref<string | null>(null)
const isManualSelection = ref(false)
const activeDeck = ref<1|2>(1);
const lineups = ref({ 1: {
  C: null, '1B': null, '2B': null, '3B': null, SS: null,
  LF: null, CF: null, RF: null, DH: null,
  SP1: null, SP2: null, SP3: null, SP4: null, SP5: null,
  RP1: null, RP2: null, RP3: null, RP4: null, RP5: null, RP6: null,
  BENCH1: null, BENCH2: null, BENCH3: null, BENCH4: null,
  BENCH5: null, BENCH6: null, BENCH7: null, BENCH8: null
} as Record<string, Raw | null>,
  2: {
    C: null, '1B': null, '2B': null, '3B': null, SS: null,
    LF: null, CF: null, RF: null, DH: null,
    SP1: null, SP2: null, SP3: null, SP4: null, SP5: null,
    RP1: null, RP2: null, RP3: null, RP4: null, RP5: null, RP6: null,
    BENCH1: null, BENCH2: null, BENCH3: null, BENCH4: null,
    BENCH5: null, BENCH6: null, BENCH7: null, BENCH8: null
  } as Record<string, Raw | null>
})

const globalBuffsAll = reactive({ 1: {
  teamLevel: 100, preferredTeam: [] as string[], clanBuff: 15, managerType: '', managerEnhance: 0,
  synergyMasteries: ['', '', '', '', ''],
  amplifiedMasteryIndex: -1,
  // 🌟 바인더 100레벨 기본값 + 5x5 빙고판 데이터 배열
  binderLevel: 100, 
  binderMatrix: Array(5).fill(0).map(() => ({ team: '', position: '', player: '', year: '', grade: '' })),
  // 🌟 감독 전술 지시 데이터
  managerBreakthrough: 0,
  tacticLevels: Array(15).fill(0),
  tacticBaseRates: { scoring: 50, cleanup: 40 },
  tacticCondRates: [5, 24.5, 20, 24.5, 21, 7.5, 30, 25.5, 25, 50, 0, 0, 50, 60, 32.5]
  },
  2: {
    teamLevel: 100, preferredTeam: [] as string[], clanBuff: 15, managerType: '', managerEnhance: 0,
    synergyMasteries: ['', '', '', '', ''],
    amplifiedMasteryIndex: -1,
    binderLevel: 100, 
    binderMatrix: Array(5).fill(0).map(() => ({ team: '', position: '', player: '', year: '', grade: '' })),
    managerBreakthrough: 0,
    tacticLevels: Array(15).fill(0),
    tacticBaseRates: { scoring: 50, cleanup: 40 },
    tacticCondRates: [5, 24.5, 20, 24.5, 21, 7.5, 30, 25.5, 25, 50, 0, 0, 50, 60, 32.5]
  }
})


const lineup = computed({
  get: () => lineups.value[activeDeck.value],
  set: (val) => lineups.value[activeDeck.value] = val
})
const playerBuffs = computed({
  get: () => allPlayerBuffs.value[activeDeck.value],
  set: (val) => allPlayerBuffs.value[activeDeck.value] = val
})
const globalBuffs = computed({
  get: () => globalBuffsAll[activeDeck.value],
  set: (val) => globalBuffsAll[activeDeck.value] = val
})

const TACTICS_INFO = [
  { id:0, name: '마음껏 휘둘러라', type: '타자', req: [0,2,5,8,12,15], pt: [0,1,3,6,10,15], baseVals: [0,25,40,70,120,200], condVals: [0,25,40,70,120,200], descBase: (v) => `클린업(3~5번) 홈런 +${v}`, descCond: (v) => `클린업(3~5번) 홈런 3회시 하위(6~9번) 컨택트 +${v}` },
  { id:1, name: '연결고리', type: '타자', req: [0,2,5,8,12,15], pt: [0,1,3,6,10,15], baseVals: [0,20,30,50,90,150], condVals: [0,10,20,35,60,100], descBase: (v) => `9, 1, 2번 타순 컨택트 +${v}`, descCond: (v) => `9, 1, 2번 타순 4출루시 클린업(3~5번) 갭 파워 +${v}` },
  { id:2, name: '하위 타선의 반란', type: '타자', req: [0,2,5,8,12,15], pt: [0,1,3,6,10,15], baseVals: [0,10,25,35,60,100], condVals: [0,25,40,70,120,200], descBase: (v) => `하위 타선(6~9번) 컨택트 +${v}`, descCond: (v) => `하위 타선 4안타시 상위 타선(1,2번) 선구 +${v}` },
  { id:3, name: '끈질긴 승부', type: '타자', req: [0,2,5,8,12,15], pt: [0,1,3,6,10,15], baseVals: [0,10,15,25,40,70], condVals: [0,5,10,18,30,50], descBase: (v) => `전체 타자 선구 +${v}`, descCond: (v) => `팀 볼넷 5회 달성 시 전체 타자 컨택트 +${v}` },
  { id:4, name: '배럴 타구 생산', type: '타자', req: [0,2,5,8,12,15], pt: [0,1,3,6,10,15], baseVals: [0,10,15,25,40,70], condVals: [0,3,6,10,18,30], descBase: (v) => `전체 타자 갭 파워 +${v}`, descCond: (v) => `팀 2루타 3회 달성 시 전체 타자 컨택트 +${v}` },
  { id:5, name: '존에 욱여넣어라', type: '투수', req: [0,2,5,8,12,15], pt: [0,1,3,6,10,15], baseVals: [0,15,25,45,70,120], condVals: [0,40,65,100,180,300], descBase: (v) => `선발투수 컨트롤 +${v}`, descCond: (v) => `선발 볼넷 0개인 경우 전체 야수(투타전체) 수비 +${v}` },
  { id:6, name: '벌떼 야구', type: '투수', req: [0,2,5,8,12,15], pt: [0,1,3,6,10,15], baseVals: [0,20,30,50,90,150], condVals: [0,25,40,70,120,200], descBase: (v) => `계투진 파워 +${v}`, descCond: (v) => `6회 이전 계투 등판 시 계투진 무브먼트 +${v}` },
  { id:7, name: '이닝이터', type: '투수', req: [0,2,5,8,12,15], pt: [0,1,3,6,10,15], baseVals: [0,2,3,5,9,15], condVals: [0,20,30,50,90,150], descBase: (v) => `선발투수 한계 투구 +${v}`, descCond: (v) => `선발 90구 이상 투구 시 선발 파워 +${v}` },
  { id:8, name: '감독 마운드 방문', type: '투수', req: [0,2,5,8,12,15], pt: [0,1,3,6,10,15], baseVals: [0,20,30,50,90,150], condVals: [0,20,30,50,90,150], descBase: (v) => `득점권 상황 투수 컨트롤 +${v}`, descCond: (v) => `득점권 무실점 이닝 2회 달성 시 투수진 홈런 억제 +${v}` },
  { id:9, name: '좌우놀이', type: '투수', req: [0,2,5,8,12,15], pt: [0,1,3,6,10,15], baseVals: [0,25,40,70,120,200], condVals: [0,15,25,45,70,120], descBase: (v) => `계투진에 좌/우/언더 각각 있을 시 계투 파워 +${v}`, descCond: (v) => `계투진의 동일 핸드타입 타자 상대 시 무브먼트 +${v}` },
  { id:10, name: '작전 야구', type: '운영', req: [0,2,5,8,12,15], pt: [0,1,3,6,10,15], baseVals: [0,15,25,45,70,120], condVals: [0,60,100,180,300,500], descBase: (v) => `상위(1,2번)/하위타선(6~9번) 선구 +${v}`, descCond: (v) => `희생번트 성공 직후 다음 타자 파워 +${v}` },
  { id:11, name: '발야구', type: '운영', req: [0,2,5,8,12,15], pt: [0,1,3,6,10,15], baseVals: [0,5,8,15,25,40], condVals: [0,25,40,70,120,200], descBase: (v) => `상위(1,2번)/하위타선(6~9번) 도루, 주루 +${v}`, descCond: (v) => `도루 성공 직후 다음 타자 갭 파워, 홈런 +${v}` },
  { id:12, name: '라인 수비', type: '운영', req: [0,2,5,8,12,15], pt: [0,1,3,6,10,15], baseVals: [0,15,25,45,70,120], condVals: [0,15,25,45,70,120], descBase: (v) => `클린업(3~5번) 상대 장타 억제 +${v}`, descCond: (v) => `6회까지 클린업 장타 3개 미만 시 계투진 무브먼트 +${v}` },
  { id:13, name: '기본기 중시', type: '운영', req: [0,2,5,8,12,15], pt: [0,1,3,6,10,15], baseVals: [0,40,65,100,180,300], condVals: [0,40,65,100,180,300], descBase: (v) => `전체 야수(투타전체) 수비 +${v}`, descCond: (v) => `6회까지 실책 0개인 경우 셋업, 마무리 파워 +${v}` },
  { id:14, name: '내야 시프트', type: '운영', req: [0,2,5,8,12,15], pt: [0,1,3,6,10,15], baseVals: [0,40,65,100,180,300], condVals: [0,20,30,50,90,150], descBase: (v) => `내야수(1/2/3루,유격) 수비 +${v}`, descCond: (v) => `3회까지 안타 4개 미만인 경우 선발 컨트롤, 스터프 +${v}` }
];

const totalTacticPt = computed(() => 2 + (globalBuffsAll[activeDeck.value].managerEnhance * 2) + (globalBuffsAll[activeDeck.value].managerBreakthrough * 4));
const usedTacticPt = computed(() => {
  let sum = 0;
  if(globalBuffsAll[activeDeck.value].tacticLevels) {
    globalBuffsAll[activeDeck.value].tacticLevels.forEach((lv, i) => {
      if(lv > 0 && TACTICS_INFO[i]) sum += TACTICS_INFO[i].pt[lv];
    });
  }
  return sum;
});
const remainingTacticPt = computed(() => totalTacticPt.value - usedTacticPt.value);


const allPlayerBuffs = ref<Record<1|2, Record<string, PlayerBuff>>>({ 1: {}, 2: {} })

// 🌟 9up 인게임 고증: 각인(Imprint) 시스템 상태 및 로직 🌟
type ImprintRole = '타자' | '투수'; 
type ImprintGrade = '노말' | '고급' | '특별' | '레전드' | '얼티밋';
type SubOptType = '컨택' | '갭파워' | '홈런파워' | '선구' | '삼진회피' | '무브먼트' | '장타억제' | '홈런억제' | '컨트롤' | '스터프' | '수비' | '한계투구 증가' | '1~2선발시 파워증가' | '전체 능력치' | '조건부 파워' | '수익 증가';

interface ImprintSubOption {
  type: SubOptType;
  value: number;
}

interface Imprint {
  id: string;
  name: string;
  role: ImprintRole;
  grade: ImprintGrade;
  mainStat: string; // 🌟 핵심: 5대 스탯 중 어떤 스탯을 올려주는지 저장!
  mainPower: number; 
  subOptions: ImprintSubOption[]; 
  ultimateBonus?: { targetGrade: string, power: number }; 
}

const imprintInventory = ref<Imprint[]>([]);
const showImprintManager = ref(false);

const newImprint = ref({ 
  name: '', role: '타자' as ImprintRole, grade: '레전드' as ImprintGrade, 
  mainStat: '컨택', // 생성 기본값
  mainPower: 270,
  subOptions: [
    { type: '컨택' as SubOptType, value: 0 },
    { type: '갭파워' as SubOptType, value: 0 },
    { type: '전체 능력치' as SubOptType, value: 0 }
  ],
  ultimateBonus: { targetGrade: 'DGN', power: 0 } // 내부 코드로 변경
});

// 🌟 역할(타/투) 변경 시 주옵/부옵 스탯 꼬임 방지 🌟
const handleRoleChange = () => {
  newImprint.value.mainStat = newImprint.value.role === '타자' ? '컨택' : '무브먼트';
  newImprint.value.subOptions.forEach(opt => {
    opt.type = newImprint.value.role === '타자' ? '컨택' : '무브먼트';
    opt.value = 0;
  });
};

const updateSubOptionsCount = () => {
  let count = 3; 
  if (newImprint.value.grade === '노말') count = 0;
  if (newImprint.value.grade === '고급') count = 1;
  if (newImprint.value.grade === '특별') count = 2;
  
  while (newImprint.value.subOptions.length < count) {
    newImprint.value.subOptions.push({ type: newImprint.value.role === '타자' ? '컨택' : '무브먼트', value: 0 });
  }
  if (newImprint.value.subOptions.length > count) {
    newImprint.value.subOptions.splice(count);
  }
};

const createImprint = () => {
  if (!newImprint.value.name.trim()) return showToast('각인 이름을 입력해주세요!', 'error');
  const imp: Imprint = {
    id: Date.now().toString(),
    name: newImprint.value.name.trim(),
    role: newImprint.value.role,
    grade: newImprint.value.grade,
    mainStat: newImprint.value.mainStat, // 주옵션 저장
    mainPower: Number(newImprint.value.mainPower) || 0,
    subOptions: newImprint.value.subOptions.map(o => ({ type: o.type, value: Number(o.value) || 0 }))
  };
  
  if (imp.grade === '얼티밋') {
    imp.ultimateBonus = {
      targetGrade: newImprint.value.ultimateBonus.targetGrade,
      power: Number(newImprint.value.ultimateBonus.power) || 0
    };
  }
  
  imprintInventory.value.push(imp);
  showToast(`[${imp.role}용 - ${imp.mainStat}] 각인이 생성되었습니다!`, 'success');
};

const deleteImprint = (id: string) => {
  if(!confirm('이 각인을 보관함에서 완전히 삭제하시겠습니까?')) return;
  imprintInventory.value = imprintInventory.value.filter(i => i.id !== id);
  Object.keys(playerBuffs.value).forEach(pos => {
    if (playerBuffs.value[pos]?.imprint1?.id === id) playerBuffs.value[pos].imprint1 = null;
    if (playerBuffs.value[pos]?.imprint2?.id === id) playerBuffs.value[pos].imprint2 = null;
  });
};

const showImprintEquipper = ref(false);
const equipTarget = ref<{ pos: string, slot: 1 | 2 } | null>(null);

const openEquipModal = (pos: string, slot: 1 | 2) => {
  equipTarget.value = { pos, slot };
  showImprintEquipper.value = true;
};

const equipImprint = (imprint: Imprint) => {
  if (!equipTarget.value) return;
  const { pos, slot } = equipTarget.value;
  const p = lineup.value[pos];
  if (!p) return;
  
  const isPitcherSlot = pos.startsWith('SP') || pos.startsWith('RP');
  const targetRole = isPitcherSlot ? '투수' : '타자';
  if (imprint.role !== targetRole) return showToast(`장착 실패! ${targetRole}용 각인만 낄 수 있습니다!`, 'error');

  if (!playerBuffs.value[pos]) playerBuffs.value[pos] = {};
  const currentBuffs = playerBuffs.value[pos];
  
  const otherImprint = slot === 1 ? currentBuffs.imprint2 : currentBuffs.imprint1;

  if (otherImprint?.id === imprint.id) return showToast('이미 반대쪽 슬롯에 장착된 각인입니다.', 'error');

  // 🌟 핵심 방어막: 주옵션 중복 장착 절대 불가!
  if (otherImprint?.mainStat === imprint.mainStat) {
    return showToast(`장착 실패! 동일 주옵션 중복 장착 불가`, 'error');
  }

  if (imprint.grade === '얼티밋' && p.grade !== 'Dignity') { 
    if (otherImprint?.grade === '얼티밋') return showToast('디그니티 등급이 아닌 선수는 얼티밋 각인을 1개만 장착할 수 있습니다!', 'error');
  }

  if (slot === 1) playerBuffs.value[pos].imprint1 = imprint;
  else playerBuffs.value[pos].imprint2 = imprint;
  
  showImprintEquipper.value = false;
};

const unequipImprint = (pos: string, slot: 1 | 2) => {
  if (playerBuffs.value[pos]) {
    if (slot === 1) playerBuffs.value[pos].imprint1 = null;
    else playerBuffs.value[pos].imprint2 = null;
  }
};

const getGradeColor = (grade: ImprintGrade) => {
  switch(grade) {
    case '얼티밋': return 'text-red-600 bg-red-100 border-red-300';
    case '레전드': return 'text-purple-600 bg-purple-100 border-purple-300';
    case '특별': return 'text-blue-600 bg-blue-100 border-blue-300';
    case '고급': return 'text-green-600 bg-green-100 border-green-300';
    default: return 'text-neutral-600 bg-neutral-200 border-neutral-300';
  }
};
  
const initPlayerBuff = (slot: string, p: Raw) => {
  const grade = String(p.grade || '').toUpperCase()
  let colBuff = 1200
  
  if (['SEA', 'ASG'].includes(grade)) colBuff = 800
  else if (['POS', 'TEA', 'MMVP', 'HIT', 'ACE', 'GGY'].includes(grade)) colBuff = 900
  else if (['GG', 'ROY'].includes(grade)) colBuff = 1000
  else if (grade === 'TOP') colBuff = 1200
  else if (grade === 'DGN') colBuff = 0

  const existing = playerBuffs.value[slot]
  const savedImprintStats = existing ? { ...existing.imprintStats } : {}
  const savedImprintCoreStat = existing ? existing.imprintCoreStat : 0
  const savedUltimateImprintPercent = existing ? existing.ultimateImprintPercent : 0
  const savedImprintStarterPower = existing ? existing.imprintStarterPower : 0
  const savedImprint1 = existing ? existing.imprint1 : null;
  const savedImprint2 = existing ? existing.imprint2 : null;
  
  // 🌟 타순 자동 배정 AI (신규)
  let savedBattingOrder = existing ? existing.battingOrder : null;
  if (!isPitcher(p) && !slot.startsWith('BENCH') && savedBattingOrder === null) {
    const usedOrders = new Set();
    ['LF', 'CF', 'RF', '3B', 'SS', '2B', '1B', 'C', 'DH'].forEach(s => {
      if (playerBuffs.value[s]?.battingOrder) usedOrders.add(playerBuffs.value[s].battingOrder);
    });
    // 1번부터 9번까지 스캔해서 남는 번호가 있으면 즉시 부여!
    for (let i = 1; i <= 9; i++) {
      if (!usedOrders.has(i)) {
        savedBattingOrder = i;
        break;
      }
    }
  }

  playerBuffs.value[slot] = {
    enhancementLevel: grade === 'DGN' ? 10 : 15, breakthroughLevel: 0,
    careerTeamCount: 0, hitAceBuff: 0, 
    imprintStarterPower: savedImprintStarterPower, 
    careerAllStatFlat: 0, 
    imprintCoreStat: savedImprintCoreStat, 
    careerCoreStat: 0, 
    selectedSkills: [], 
    battingOrder: savedBattingOrder, // 🌟 타순 유지 및 자동 부여
    playerLevel: 100, collectionBuff: colBuff, careerLevelBuff: 149,
    binderBuff: 537, 
    ultimateImprintPercent: savedUltimateImprintPercent, 
    imprintStats: savedImprintStats, 
    careerStats: {},
    imprint1: savedImprint1, 
    imprint2: savedImprint2  
  }
}
  
const rightPanelTab = ref<'global' | 'player'>('global')
const playerTab = ref<'stats' | 'synergy'>('stats')

// 🌟 1단계: 시너지 카테고리 자동 분류 및 탭 필터링 로직 추가
const activeSynergyCategory = ref('전체');
const playerSynergyCategory = ref('전체');

const getSynergyCategory = (synName: string) => {
  const name = String(synName || '').replace(/\s+/g, '').trim();
  
  // 🌟 예외 처리 VIP: '배터리' 키워드 추가! (기록 탭에 뺏기지 않고 무조건 인물 탭으로 직행!)
  if (name.match(/대통령배MVP|봉황대기MVP|청룡기MVP|고춧가루|왕조주역|돌격대|황금세대|\d{4}국가대표팀|실업야구|라이벌|원투펀치|배터리/)) {
    return '인물';
  }

  // 1. 기본 (연도, 구단, 시즌카 종류)
  if (/^\d{4}$|^\d{4}년/.test(name) || name.match(/SSG|SK|키움|히어로즈|넥센|KIA|해태|삼성|두산|OB|롯데|LG|MBC|한화|빙그레|NC|KT|현대|태평양|청보|삼미|쌍방울|디그니티|탑클|에이스|히트|골든글러브|골글|MVP|신인왕|포스트시즌|올스타|국가대표|타이틀|프랜차이즈/)) return '기본';
  
  // 2. 출신 (학교, 외국인 등)
  if (name.match(/출신|외국인|용병|해외파|고등학교|대학교|중학교|초등학교/) || name.endsWith('고') || name.endsWith('대') || name.endsWith('상고') || name.endsWith('공고')) return '출신';
  
  // 3. 기록 (통산, 한시즌, 경기 등)
  if (name.match(/경기|안타|홈런|도루|타점|득점|승|세이브|홀드|탈삼진|이닝|클럽|철인|기록|-|1위/)) return '기록';
  
  // 4. 인물 (나머지 특수 시너지들)
  return '인물';
}

const filteredActiveTeamSynergies = computed(() => {
  return activeTeamSynergies.value.filter(s => activeSynergyCategory.value === '전체' || getSynergyCategory(s.name) === activeSynergyCategory.value);
});
const filteredPendingTeamSynergies = computed(() => {
  return pendingTeamSynergies.value.filter(s => activeSynergyCategory.value === '전체' || getSynergyCategory(s.name) === activeSynergyCategory.value);
});
const filteredPlayerActiveSynergies = computed(() => {
  if (!selectedSlot.value || !lineup.value[selectedSlot.value]) return [];
  return getExpandedPlayerSynergies(lineup.value[selectedSlot.value]!).filter(s => {
    const matchCategory = playerSynergyCategory.value === '전체' || getSynergyCategory(s) === playerSynergyCategory.value;
    return matchCategory && isSynergyActiveForPlayer(lineup.value[selectedSlot.value]!, s);
  });
});
const filteredPlayerInactiveSynergies = computed(() => {
  if (!selectedSlot.value || !lineup.value[selectedSlot.value]) return [];
  return getExpandedPlayerSynergies(lineup.value[selectedSlot.value]!).filter(s => {
    const matchCategory = playerSynergyCategory.value === '전체' || getSynergyCategory(s) === playerSynergyCategory.value;
    return matchCategory && !isSynergyActiveForPlayer(lineup.value[selectedSlot.value]!, s);
  });
});
  
const synergyHierarchy: Record<string, string[]> = {
  '190안타 클럽': ['180안타 클럽', '170안타 클럽'], '180안타 클럽': ['170안타 클럽'],
  '40홈런 클럽': ['30홈런 클럽'], '40도루 클럽': ['30도루 클럽'],
  '20승 클럽': ['15승 클럽'], '180탈삼진 클럽': ['150탈삼진 클럽'],
  '200이닝 클럽': ['180이닝 클럽'], '30세이브 클럽': ['20세이브 클럽'],
  '30홀드 클럽': ['20홀드 클럽'], '계투 80이닝 클럽': ['계투 70이닝 클럽'],
  '3-30-100-100 클럽': ['3-30-100 클럽', '100득점-100타점 클럽', '100타점 클럽', '30홈런 클럽'],
  '3-30-100 클럽': ['100타점 클럽', '30홈런 클럽'], '100득점-100타점 클럽': ['100타점 클럽'],
  '통산 2000경기 클럽': ['통산 1500경기 클럽'], '통산 1500경기 클럽': [],
  '통산 700경기 클럽': ['통산 500경기 클럽'], '통산 500경기 클럽': [],
  '통산 2000안타 클럽': ['통산 1500안타 클럽'], '통산 300도루 클럽': ['통산 200도루 클럽'],
  '통산 300홈런 클럽': ['통산 200홈런 클럽']
}

const SKILL_EFFECTS: Record<string, any> = {
  "1번": {"powerPercent": 10.0, "stats": {}}, "2번": {"powerPercent": 10.0, "stats": {}}, 
  "3번": {"powerPercent": 10.0, "stats": {}}, "4번": {"powerPercent": 10.0, "stats": {}}, 
  "5번": {"powerPercent": 10.0, "stats": {}}, "6번": {"powerPercent": 10.0, "stats": {}}, 
  "7번": {"powerPercent": 10.0, "stats": {}}, "8번": {"powerPercent": 10.0, "stats": {}}, 
  "9번": {"powerPercent": 10.0, "stats": {}}, "OPS형 타자": {"powerPercent": 0, "stats": {"gapPower": 10.0, "homeRunPower": 10.0}}, 
  "갭 히터": {"powerPercent": 0, "stats": {"gapPower": 15.0}}, "게스히팅": {"powerPercent": 0, "stats": {"gapPower": 8.0, "homeRunPower": 10.0, "strikeoutAvoidance": -5.0}}, 
  "공갈포": {"powerPercent": 0, "stats": {"homeRunPower": 20.0, "contact": -7.0, "strikeoutAvoidance": -7.0}}, 
  "그라운드볼러": {"powerPercent": 0, "stats": {"movement": -5.0, "homeRunSuppression": 10.0, "longHitSuppression": 10.0}}, 
  "그린라이트": {"powerPercent": 0, "stats": {}}, 
  "너클볼": {"powerPercent": 0, "stats": {"stuff": 20.0, "homeRunSuppression": -5.0, "longHitSuppression": -5.0, "movement": 20.0, "control": 20.0}}, 
  "더티 무브먼트": {"powerPercent": 0, "stats": {"movement": 25.0}}, "라이징 무브먼트": {"powerPercent": 0, "stats": {"stuff": 20.0}}, 
  "로우볼 히터": {"powerPercent": 0, "stats": {"gapPower": 5.0, "homeRunPower": 10.0, "plateDiscipline": -5.0}}, 
  "롱맨": {"powerPercent": 10.0, "stats": {}}, 
  "맞춰잡기": {"powerPercent": 0, "stats": {"control": 15.0, "pitchLimit": 10.0}}, 
  "묵직함": {"powerPercent": 0, "stats": {"longHitSuppression": 10.0, "homeRunSuppression": 10.0}}, 
  "믿을맨": {"powerPercent": 10.0, "stats": {}}, "배드볼히터": {"powerPercent": 0, "stats": {"contact": 15.0, "gapPower": 20.0, "plateDiscipline": -3.0}}, 
  "배럴 히터": {"powerPercent": 0, "stats": {"contact": 10.0, "gapPower": 10.0, "strikeoutAvoidance": 10.0}}, "변칙타순": {"powerPercent": 4.0, "stats": {}}, 
  "변칙투구": {"powerPercent": 0, "stats": {}}, "선구안": {"powerPercent": 0, "stats": {"strikeoutAvoidance": 15.0, "plateDiscipline": 15.0}}, 
  "셋업": {"powerPercent": 10.0, "stats": {}}, "셋업맨": {"powerPercent": 10.0, "stats": {}},
  "스토퍼": {"powerPercent": 10.0, "stats": {}}, "마무리": {"powerPercent": 10.0, "stats": {}},
  "스플리터": {"powerPercent": 0, "stats": {"movement": 15.0, "stuff": 25.0, "control": -5.0}}, 
  "승리계투": {"powerPercent": 10.0, "stats": {}}, "숏릴리프": {"powerPercent": 10.0, "stats": {}},
  "스피드스터": {"powerPercent": 0, "stats": {}}, "슬랩 히터": {"powerPercent": 0, "stats": {"contact": 20.0, "baseRunning": 10.0}}, 
  "싱커(투심)": {"powerPercent": 0, "stats": {"homeRunSuppression": 20.0, "stuff": -5.0}}, 
  "에이스": {"powerPercent": 9.0, "stats": {}}, "와일드씽": {"powerPercent": 0, "stats": {"control": -3.0, "stuff": 10.0}}, "원투펀치": {"powerPercent": 8.0, "stats": {}}, 
  "원포인터": {"powerPercent": 10.0, "stats": {}}, "이닝이팅": {"powerPercent": 0, "stats": {"pitchLimit": 5.0}}, "적극성": {"powerPercent": 0, "stats": {"contact": 15.0}}, 
  "지명타자": {"powerPercent": 8.5, "stats": {}}, "체인지업": {"powerPercent": 0, "stats": {"longHitSuppression": 15.0}}, 
  "커브": {"powerPercent": 0, "stats": {"movement": 15.0, "longHitSuppression": 10.0}}, 
  "컨택터": {"powerPercent": 0, "stats": {"contact": 20.0}}, "클로저": {"powerPercent": 10.0, "stats": {}}, "클린업": {"powerPercent": 8.0, "stats": {}}, 
  "타격 전략": {"powerPercent": 0, "stats": {"contact": 20.0}}, "테이블세터": {"powerPercent": 7.0, "stats": {}}, "파워": {"powerPercent": 0, "stats": {"gapPower": 15.0, "homeRunPower": 15.0}}, 
  "파이어볼러": {"powerPercent": 0, "stats": {"stuff": 15.0}}, "펀치력": {"powerPercent": 0, "stats": {"gapPower": 10.0, "homeRunPower": 5.0}}, 
  "플라이볼피쳐": {"powerPercent": 0, "stats": {"movement": 20.0, "homeRunSuppression": -5.0}}, "하위타선": {"powerPercent": 8.0, "stats": {"defense": 10.0}}, 
  "하이볼 히터": {"powerPercent": 0, "stats": {"contact": 10.0, "strikeoutAvoidance": 5.0, "homeRunPower": -5.0}}
}


const normalSkillData = ref<any[]>([])
const enhancedSkillData = ref<any[]>([])

const matchSkillInfo = (skill: string) => {
  return normalSkillData.value.find((s) => s.skill === skill)?.image || ''
}

const getNormalSkillDescription = (skillName: string) => {
  const data = normalSkillData.value.find(s => s.skill === skillName);
  
  const effectText = data?.effects || data?.effect;
  if (effectText) {
    if (Array.isArray(effectText)) {
       return effectText.map(e => e.startsWith('-') ? e : `- ${e}`).join('\n');
    }
    return String(effectText).replace(/\\n/g, '\n');
  }
  
  const eff = SKILL_EFFECTS[skillName];
  if (eff) {
    const parts = [];
    if (eff.powerPercent) parts.push(`- 파워 +${eff.powerPercent}%`);
    for (const [k, v] of Object.entries(eff.stats || {})) {
      parts.push(`- ${STAT_LABELS[k] || k} +${v}`);
    }
    if (parts.length > 0) return parts.join('\n');
  }

  return '- 특수 조건 발동 스킬';
}

const matchEnhancedSkillImage = (skill: string) => {
  const img = enhancedSkillData.value.find((s) => s.enhanced_skill === skill)?.image || '';
  if (!img) return '';
  return img.startsWith('bg-') ? img : `bg-${img}`;
}

const getEnhancedSkillEffect = (skillName: string, level: number) => {
  const data = enhancedSkillData.value.find((s) => s.enhanced_skill === skillName);
  if (!data) return '효과 정보를 불러올 수 없습니다.';

  const list = data.effects_by_level || data.effects || data.levels;
  if (Array.isArray(list) && list.length > 0) {
    const idx = Math.min(Math.max(0, level), list.length - 1);
    return list[idx] || list[list.length - 1] || '';
  }
  if (list && typeof list === 'object') {
    if (list[level]) return list[level];
    if (list[String(level)]) return list[String(level)];
  }
  const exactKeys = [`lv${level}`, `lv_${level}`, `Lv${level}`, `Lv_${level}`, `level${level}`, `level_${level}`, `effect${level}`, `effect_${level}`, `eff${level}`, `eff_${level}`];
  for (const k of exactKeys) {
    if (data[k]) return data[k];
  }
  if (level === 0) {
    if (data['기본']) return data['기본'];
    if (data.base) return data.base;
  }
  return data.description || '해당 레벨의 효과 데이터가 없습니다.';
}

const tooltipState = reactive({
  show: false,
  skill: '',
  x: 0,
  y: 0,
  transform: 'translate(-50%, -100%)',
  arrowLeft: '50%'
});

const showSkillTooltip = (e: MouseEvent, sk: string) => {
  const rect = (e.currentTarget as HTMLElement).getBoundingClientRect();
  let x = rect.left + rect.width / 2;
  let y = rect.top;
  let transform = 'translate(-50%, -100%)';
  let arrowLeft = '50%';

  if (x < 130) {
    transform = 'translate(-20%, -100%)';
    arrowLeft = '20%';
  } else if (window.innerWidth - x < 130) {
    transform = 'translate(-80%, -100%)';
    arrowLeft = '80%';
  }

  tooltipState.skill = sk;
  tooltipState.x = x;
  tooltipState.y = y;
  tooltipState.transform = transform;
  tooltipState.arrowLeft = arrowLeft;
  tooltipState.show = true;
};

const hideSkillTooltip = () => {
  tooltipState.show = false;
};

const isSkillActive = (skillName: string, slot: string, battingOrder: number | null) => {
  const s = slot.toUpperCase()
  if (skillName === '에이스' || skillName === '원투펀치') return s === 'SP1' || s === 'SP2'
  if (skillName === '승리계투') return s === 'RP1' || s === 'RP2'
  if (skillName === '숏릴리프') return s === 'RP1' || s === 'RP2' || s === 'RP3'
  if (skillName === '셋업' || skillName === '셋업맨') return s === 'RP3'
  if (skillName === '클로저') return s === 'RP4'
  if (skillName === '롱맨') return s === 'RP5'
  if (skillName === '스토퍼' || skillName === '마무리') return s === 'RP6'
  if (skillName === '지명타자') return s === 'DH'
  if (skillName.endsWith('번') && skillName.length === 2) return battingOrder === parseInt(skillName[0])
  if (skillName === '테이블세터') return battingOrder === 1 || battingOrder === 2
  if (skillName === '클린업') return battingOrder === 3 || battingOrder === 4 || battingOrder === 5
  if (skillName === '하위타선') return battingOrder !== null && battingOrder >= 6 && battingOrder <= 9
  return true
}

const toLowerCase = (s: unknown): string => String(s ?? '').toLowerCase().trim()
const normalizeText = (text: unknown): string => String(text ?? '').normalize('NFKC').replace(/[​-‍﻿]/g, '').replace(/\s+/g, ' ').trim().toLowerCase()
const getCleanArray = (value: any): string[] => {
  if (!value) return []
  let str = Array.isArray(value) ? value.join(',') : String(value)
  str = str.replace(/[\[\]"'`]/g, '')
  return str.split(/[,;]+/).map(x => x.trim()).filter(Boolean)
}
const getArray = getCleanArray
const toArray = getCleanArray

const normalizePosition = (position: any): string => {
  const str = String(position ?? '').replace(/[\[\]"'`\s]/g, '').trim()
  if (!str) return ''
  const lower = str.toLowerCase()
  return POSITION_ALIASES[lower] ?? str.toUpperCase()
}

const searchOptions = computed(() => {
  const o: Record<string, Set<string>> = { team: new Set(), position: new Set(), grade: new Set() }
  for (const p of players.value) {
    getArray(p.team).forEach(v => o.team.add(v))
    getArray(p.position).forEach(v => o.position.add(v))
    if (p.grade) o.grade.add(String(p.grade))
  }
  return {
    team: [...o.team].sort(),
    position: [...o.position].sort(),
    grade: [...o.grade].sort((a, b) => {
      const gradeOrder = ['SS', 'S', 'A', 'B', 'C', 'D']
      return gradeOrder.indexOf(a) - gradeOrder.indexOf(b)
    })
  }
})

interface PreparedPlayer { raw: Raw; nameNormalized: string; teamLowerCase: string[]; positionLowerCase: string[]; yearsNumeric: number[]; synergyNormalizedSet: Set<string>; }

const preparedPlayers = computed<PreparedPlayer[]>(() =>
    players.value.map(player => ({
      raw: player, nameNormalized: normalizeText(player.name),
      teamLowerCase: toArray(player.team).map(toLowerCase), positionLowerCase: toArray(player.position).map(normalizePosition).map(toLowerCase),
      yearsNumeric: toArray(player.year).map((y:any)=>Number(y)).filter((y:any)=>!Number.isNaN(y)),
      synergyNormalizedSet: new Set(toArray(player.synergy).map(normalizeText))
    }))
)
const groupedTeams = [
  { id: ['ssg', 'sk'], name: 'SSG/SK' }, { id: ['kiwoom', 'nexen'], name: '키움/히어로즈' },
  { id: ['kia', 'haitai'], name: 'KIA/해태' }, { id: ['samsung'], name: '삼성' },
  { id: ['doosan', 'ob'], name: '두산/OB' }, { id: ['lotte'], name: '롯데' },
  { id: ['lg', 'mbc'], name: 'LG/MBC' }, { id: ['hanwha', 'binggrae'], name: '한화/빙그레' },
  { id: ['nc'], name: 'NC' }, { id: ['kt'], name: 'KT' },
  { id: ['hyundai', 'pacific', 'chungbo', 'sammi'], name: '현대/태평양/청보/삼미' }, { id: ['sbw'], name: '쌍방울' }
]

const toggleTeamGroup = (group: { id: string[], name: string }) => {
  if (isTeamGroupSelected(group)) searchQuery.team = searchQuery.team.filter(t => !group.id.includes(t))
  else {
    const newTeams = [...searchQuery.team]
    group.id.forEach(t => { if (!newTeams.includes(t)) newTeams.push(t) })
    searchQuery.team = newTeams
  }
}
const isTeamGroupSelected = (group: { id: string[], name: string }) => group.id.every(t => searchQuery.team.includes(t))
const findTeamLogo = (teamKey: string): string | null => {
  for (const team of teamData.value) {
    if (!team.history) continue
    for (const history of team.history) { if (history.key === teamKey) return history.logo }
  }
  return null
}
const findTeamName = (teamKeyOrName: string): string => {
  const key = String(teamKeyOrName ?? '')
  for (const team of teamData.value) {
    if (team.key === key) return team.name
    if (!team.history) continue
    for (const h of team.history) { if (h.key === key || h.name === key) return h.name }
  }
  return key
}
const getTeamLogoUrl = (teamKey: string): string => findTeamLogo(teamKey) ?? '/assets/logos/teams/unknown.png'

const filteredSynergyOptions = computed(() => {
  const query = normalizeText(synergySearchText.value)
  if (!query) return synergyOptions.value
  return synergyOptions.value.filter(s => normalizeText(s).includes(query))
})
const toggleSynergyFilter = (s: string) => {
  if (searchQuery.synergy.includes(s)) searchQuery.synergy = searchQuery.synergy.filter(x => x !== s)
  else searchQuery.synergy.push(s)
}


// 🌟 스킬 검색용 드롭다운 및 이미지 매칭 엔진
const isSkillDropdownOpen = ref(false)
const skillSearchText = ref('')
const excludedFilterSkills = ['마무리', '셋업맨', '숏릴리프', '승리계투'];
const skillOptions = computed(() => {
  const dbSkills = normalSkillData.value.map(s => s.skill);
  const hardcodedSkills = Object.keys(SKILL_EFFECTS);
  const allSkills = Array.from(new Set([...dbSkills, ...hardcodedSkills]));
  return allSkills.filter(sk => !excludedFilterSkills.includes(sk)).sort();
});

const filteredSkillOptions = computed(() => {
  const query = normalizeText(skillSearchText.value)
  if (!query) return skillOptions.value
  return skillOptions.value.filter(s => normalizeText(s).includes(query))
})

const toggleSkillFilter = (s: string) => {
  if (searchQuery.skill.includes(s)) searchQuery.skill = searchQuery.skill.filter(x => x !== s)
  else searchQuery.skill.push(s)
}

// 🌟 2. 필터 검색 엔진 완벽 개조 (단어 띄어쓰기 AND 검색, 등급/별 필터 완벽 호환)

const filteredPlayers = computed(() => {
  // 🌟 수정됨: 쉼표(,)는 OR 검색으로, 띄어쓰기( )는 AND 검색으로 동작하도록 스마트 파싱
  const searchGroups = searchQuery.search 
    ? searchQuery.search.split(',').map(g => g.trim()).filter(Boolean).map(g => g.split(/\s+/).map(t => normalizeText(t)).filter(Boolean))
    : [];
  
  return preparedPlayers.value.filter(({ raw: p, nameNormalized, teamLowerCase, positionLowerCase, yearsNumeric, synergyNormalizedSet }) => {
    
    // ① 팀 필터
    if (searchQuery.team.length && !searchQuery.team.some(t => teamLowerCase.includes(toLowerCase(t)))) return false;
    
    // ② 별(희귀도) 필터 고장 수정: p.rarity와 p.stars 모두 호환 적용
    if (searchQuery.rarity != null && Number(p.rarity || p.stars || 1) !== Number(searchQuery.rarity)) return false;
    
    // ③ 등급 필터 고장 수정: 풀네임('DIGNITY')과 약자('DGN')를 모두 매핑해서 완벽 비교
    if (searchQuery.grade.length) {
       const playerGradeMapped = getMappedGrade(p.grade);
       const queryGradesMapped = searchQuery.grade.map(g => getMappedGrade(g));
       if (!queryGradesMapped.includes(playerGradeMapped)) return false;
    }

    
    // ④ 포지션 및 시너지 필터
    if (searchQuery.position.length && !searchQuery.position.some(v => positionLowerCase.includes(toLowerCase(v)))) return false;
    if (searchQuery.synergy.length && !searchQuery.synergy.map(normalizeText).every(t => synergyNormalizedSet.has(t))) return false;
    
    // 🌟 핵심: 스킬 필터 (선택한 모든 스킬을 선수가 가지고 있어야 함)
    if (searchQuery.skill.length > 0) {
       const playerSkills = [...getArray(p.skill), ...getArray(p.enhancedSkill)];
       const hasAllSkills = searchQuery.skill.every(sk => playerSkills.includes(sk));
       if (!hasAllSkills) return false;
    }

    
    // ⑤ 🌟 핵심: 쉼표(OR) & 띄어쓰기(AND) 복합 검색 엔진!
    if (searchGroups.length > 0) {
      const hay = [nameNormalized, ...teamLowerCase, ...positionLowerCase, ...Array.from(synergyNormalizedSet), ...yearsNumeric.map(String)].join(' ');
      
      // 여러 그룹(쉼표로 구분) 중 단 하나라도, 그 그룹 안의 모든 키워드(띄어쓰기로 구분)를 가지고 있다면 통과!
      const isMatch = searchGroups.some(tokens => tokens.every(t => hay.includes(t)));
      if (!isMatch) return false;
    }
    
    return true;
  }).map(pp => pp.raw);
})
  
const totalPlayers = computed(() => filteredPlayers.value.length)
const totalPages = computed(() => Math.max(1, Math.ceil(totalPlayers.value / pageSize)))
const paginatedPlayers = computed(() => filteredPlayers.value.slice((currentPage.value-1)*pageSize, (currentPage.value)*pageSize))
const goToPage = (page:number) => { if (page>=1 && page<=totalPages.value) currentPage.value = page }
watch(searchQuery, () => { currentPage.value = 1 }, { deep: true })
const resetFilters = () => { searchQuery.search=''; searchQuery.team=[]; searchQuery.position=[]; searchQuery.synergy=[]; searchQuery.skill=[]; searchQuery.rarity=null; searchQuery.grade=[] }

const checkSynergyInclusion = (target: string, playerSynergies: string[]) => {
  const clean = (x:string)=>String(x??'').normalize('NFKC').replace(/[​-‍﻿]/g,'').replace(/[,\s클럽]/g,'').trim()
  const keyClean = clean(target)
  
  if (keyClean === '신') {
     return playerSynergies.some(s => clean(s) === '신');
  }

  // 🌟 핵심: '시즌'과 '포스트시즌' 글자 겹침으로 인한 교차 발동 완벽 차단 함수!
  const isIncludesSafe = (sClean: string, kClean: string) => {
    if (kClean.includes('시즌') && !kClean.includes('포스트') && sClean.includes('포스트시즌')) return false;
    return sClean.includes(kClean);
  }

  if (playerSynergies.some(s => clean(s) === keyClean)) return true
  const tm = keyClean.match(/^(\D*)(\d+)(\D*)$/)
  
  if (!tm) return playerSynergies.some(s => isIncludesSafe(clean(s), keyClean))
  
  const [,tp,tn,ts] = tm
  const isYearTarget = tn.length === 4 && (ts === '' || ts === '년' || ts === '년도');
  if (isYearTarget || tp.includes('동명이인') || ts.includes('동명이인')) return false
  
  const tnum = parseInt(tn,10)

  const careerThresholds: Record<string, number> = {
    '경기': 200, '안타': 250, '홈런': 100, '도루': 100, '타점': 200, '득점': 200,
    '승': 40, '세이브': 80, '홀드': 80, '탈삼진': 350, '이닝': 350
  };
  const isCareer = (val: number, type: string) => val >= (careerThresholds[type] || 9999);

  return playerSynergies.some(s => {
    const sClean = clean(s)
    if (isIncludesSafe(sClean, keyClean)) return true;
    
    const parts = sClean.split('-')
    for (const part of parts) {
      const sm = part.match(/^(\D*)(\d+)(\D*)$/)
      if (!sm) continue
      const [,pp,pn,ps] = sm
      
      const isYearPlayer = pn.length === 4 && (ps === '' || ps === '년' || ps === '년도');
      if (isYearPlayer || pp.includes('동명이인') || ps.includes('동명이인')) continue
      
      const cleanPrefix = (str: string) => str.replace(/통산|투수|타자/g, '').trim();
      
      if (cleanPrefix(pp) === cleanPrefix(tp) && ps === ts && parseInt(pn,10) >= tnum) {
        if (!isCareer(tnum, ts) && isCareer(parseInt(pn,10), ps)) continue; 
        
        // 🌟 혹시 연도(숫자)가 포함된 시즌/포스트시즌일 경우의 방어막
        if (ts.includes('시즌') && !ts.includes('포스트') && ps.includes('포스트시즌')) continue;
        
        return true; 
      }
    }
    return false
  })
}

const compareCondition = (op: CountOp, lhs: number, rhs?: number, max?: number): boolean => {
  if (op==='==') return lhs === (rhs ?? 0)
  if (op=== '>=') return lhs >= (rhs ?? 0)
  if (op=== '<=') return lhs <= (rhs ?? 0)
  if (op===  '>') return lhs >  (rhs ?? 0)
  if (op===  '<') return lhs <  (rhs ?? 0)
  if (op==='between') return lhs >= (rhs ?? 0) && lhs <= (max ?? Number.POSITIVE_INFINITY)
  return false
}

// 🌟 시너지 마스터리 증폭 및 인원수 차감 계산 적용 🌟
const activeSynergiesDeck1 = computed(() => getActiveTeamSynergies(1));
const activeSynergiesDeck2 = computed(() => getActiveTeamSynergies(2));
const activeTeamSynergiesMap = { 1: activeSynergiesDeck1, 2: activeSynergiesDeck2 };
const activeTeamSynergies = computed(() => activeTeamSynergiesMap[activeDeck.value].value);

const getActiveTeamSynergies = (deckId: 1 | 2) => {
  const lineupPlayers = Object.values(lineups.value[deckId]).filter(Boolean) as Raw[]
  const result: { name: string, bonuses: { stat: string, bonus: JsonBonus }[], matchedPlayers: string[] }[] = []
  for (const s of synergys.value) {
    const name = String(s.synergy).trim()
    
    // 🌟 타자/투수 교차 발동 완벽 차단!
    const synType = getSynergyType(name, s.conditions)
    const matchedPlayers = lineupPlayers.filter(p => {
      if (!checkSynergyInclusion(name, getArray(p.synergy))) return false;
      const pIsPit = isPitcher(p);
      if (pIsPit && synType === 'batter') return false;
      if (!pIsPit && synType === 'pitcher') return false;
      return true;
    })
    const count = matchedPlayers.length

    // 마스터리 인원 보정 및 증폭 확인
    let masteryCount = 0;
    let isAmplified = false;
    if (!globalBuffsAll[deckId].synergyMasteries) globalBuffsAll[deckId].synergyMasteries = ['', '', '', '', ''];
    globalBuffsAll[deckId].synergyMasteries.forEach((m, idx) => {
      if (m === name) {
        masteryCount++;
        if (globalBuffsAll[deckId].amplifiedMasteryIndex === idx) isAmplified = true;
      }
    });

    const effectiveCount = count + masteryCount; // 마스터리로 인해 가상의 인원이 추가된 효과

    if (effectiveCount > 0) {
       const matched = (s.conditions||[]).filter(c => {
          const op = c.count?.op as CountOp
          return op === 'between' 
            ? compareCondition('between', effectiveCount, c.count?.min, c.count?.max) 
            : compareCondition(op, effectiveCount, c.count?.value)
       })
       
       if (matched.length > 0) {
         const getThreshold = (c: any) => c.count?.op === 'between' ? (c.count?.max || 0) : (c.count?.value || 0)
         const maxThreshold = Math.max(...matched.map(getThreshold))
         const highestTierConditions = matched.filter(c => getThreshold(c) === maxThreshold)
         
         result.push({ 
           name, 
           bonuses: highestTierConditions.map(c => {
             let bVal = c.bonus.value;
             if (isAmplified && c.stat === 'power') {
               if (c.bonus.unit === 'percent') bVal += 0.5; // 퍼센트면 0.5 추가
               else if (c.bonus.unit === 'fixed') bVal += 50; // 상수면 50 추가
             }
             return { stat: c.stat, bonus: { unit: c.bonus.unit, value: bVal } }
           }),
           matchedPlayers: matchedPlayers.map(p => p.name)
         })
       }
    }
  }
  return result
}

const pendingSynergiesDeck1 = computed(() => getPendingTeamSynergies(1));
const pendingSynergiesDeck2 = computed(() => getPendingTeamSynergies(2));
const pendingTeamSynergiesMap = { 1: pendingSynergiesDeck1, 2: pendingSynergiesDeck2 };
const pendingTeamSynergies = computed(() => pendingTeamSynergiesMap[activeDeck.value].value);

const getPendingTeamSynergies = (deckId: 1 | 2) => {
  const lineupPlayers = Object.values(lineups.value[deckId]).filter(Boolean) as Raw[]
  const result: { name: string, current: number, required: number, matchedPlayers: string[] }[] = []
  
  for (const s of synergys.value) {
    const name = String(s.synergy).trim()
    
    // 🌟 대기 시너지에도 교차 발동 차단 로직 똑같이 적용!
    const synType = getSynergyType(name, s.conditions)
    const matchedPlayers = lineupPlayers.filter(p => {
      if (!checkSynergyInclusion(name, getArray(p.synergy))) return false;
      const pIsPit = isPitcher(p);
      if (pIsPit && synType === 'batter') return false;
      if (!pIsPit && synType === 'pitcher') return false;
      return true;
    })
    const count = matchedPlayers.length
    
    let masteryCount = 0;
    if (!globalBuffsAll[deckId].synergyMasteries) globalBuffsAll[deckId].synergyMasteries = ['', '', '', '', ''];
    globalBuffsAll[deckId].synergyMasteries.forEach(m => {
      if (m === name) masteryCount++;
    });
    const effectiveCount = count + masteryCount;

    if (effectiveCount > 0) {
       const matched = (s.conditions||[]).filter(c => {
          const op = c.count?.op as CountOp
          return op === 'between' 
            ? compareCondition('between', effectiveCount, c.count?.min, c.count?.max) 
            : compareCondition(op, effectiveCount, c.count?.value)
       })
       
       if (matched.length === 0) {
         let minRequired = Infinity;
         (s.conditions||[]).forEach(c => {
           let req = 0;
           if (c.count?.op === 'between') req = c.count?.min;
           else if (['>=', '==', '>'].includes(c.count?.op)) req = c.count?.value;
           
           if (req > effectiveCount && req < minRequired) {
              minRequired = req;
           }
         });
         
         if (minRequired !== Infinity) {
            result.push({ 
              name, 
              current: effectiveCount,
              required: minRequired,
              matchedPlayers: matchedPlayers.map(p => p.name)
            })
         }
       }
    }
  }
  return result
}

const getTeamLevelPower = (level: number, isPref: boolean) => {
  const l = Math.min(100, Math.max(0, level || 0));
  let prefPwr = 0;
  let otherPwr = 0;
  if (l > 0) prefPwr += Math.min(l, 25) * 10;
  if (l > 25) otherPwr += Math.min(l - 25, 25) * 10;
  if (l > 50) prefPwr += Math.min(l - 50, 25) * 10;
  if (l > 75) {
    prefPwr += (l - 75) * 10;
    otherPwr += (l - 75) * 10;
  }
  return isPref ? prefPwr : otherPwr;
};

const getSameTeamCount = (p: Raw | null, deckId: 1 | 2) => {
  if (!p) return 0;
  const myTeams = toArray(p.team).map(toLowerCase);
  let validTeamIds = new Set<string>(myTeams);
  groupedTeams.filter(g => g.id.some(id => myTeams.includes(id))).forEach(g => g.id.forEach(id => validTeamIds.add(id)));

  let count = 0;
  Object.values(lineups.value[deckId]).forEach(other => {
     if (other) {
        const otherTeams = toArray(other.team).map(toLowerCase);
        if (otherTeams.some(t => validTeamIds.has(t))) count++;
     }
  });
  return count;
}

const getCareerTeamMultiplier = (slots: number) => {
  const s = Number(slots) || 0;
  if (s === 1) return 1;
  if (s === 2) return 2;
  if (s === 3) return 6;
  if (s === 4) return 8;
  if (s === 5) return 10;
  if (s >= 6) return 12;
  return 0;
}

// 🌟 1. 이름 + 스탯 이중 검사로 타자/투수를 완벽히 판별하는 엔진
const getSynergyType = (synName: string, conditions: any[]) => {
  // 쉼표와 공백을 모두 제거하여 정확도 100% 보장 (예: "1,500 경기" -> "1500경기")
  const name = String(synName || '').replace(/,/g, '').replace(/\s+/g, '').trim();
  
  // 🌟 핵심 방어막: '배터리'나 '투타' 단어가 들어가면 무조건 타자/투수 공동 적용! (포수 강퇴 방지)
  if (name.includes('배터리') || name.includes('투타')) return 'both';

  // 🌟 타자 전용을 무조건 먼저 검사!!! ("1500" 안에 "500"이 포함되어 오작동하는 억울한 버그 완벽 차단)
  if (name.includes('1000경기') || name.includes('1500경기') || name.includes('2000경기') || name.includes('2500경기') || name.includes('3000경기') || name.includes('안타') || name.includes('홈런') || name.includes('도루') || name.includes('타점') || name.includes('득점')) return 'batter';

  // 그 다음 투수 전용 검사
  if (name.includes('500경기') || name.includes('700경기') || name.includes('승') || name.includes('세이브') || name.includes('홀드') || name.includes('탈삼진') || name.includes('이닝')) return 'pitcher';

  // 기존 스탯 기반 판별 로직
  const pitStats = ['movement', 'longHitSuppression', 'homeRunSuppression', 'control', 'stuff', 'pitchLimit', 'runnerControl'];
  const batStats = ['contact', 'gapPower', 'homeRunPower', 'plateDiscipline', 'strikeoutAvoidance', 'stealing', 'baseRunning'];
  const isPit = conditions?.some(c => pitStats.includes(c.stat));
  const isBat = conditions?.some(c => batStats.includes(c.stat));
  if (isPit && !isBat) return 'pitcher';
  if (isBat && !isPit) return 'batter';
  return 'both';
}

// 각 선수가 특정 시너지를 받고 있는지 확인 (개인설정 탭 용도)
const isPlayerReceivingSynergy = (p: Raw, synName: string, deckId: 1|2) => {
  if (!p) return false;
  const hasSynergy = checkSynergyInclusion(synName, getArray(p.synergy));
  if (!hasSynergy) return false;
  const synData = synergys.value.find(s => String(s.synergy).trim() === synName);
  if (!synData) return false;
  const synType = getSynergyType(synName, synData.conditions); // 🌟 변경
  const playerIsPit = isPitcher(p);
  if (playerIsPit && synType === 'batter') return false;
  if (!playerIsPit && synType === 'pitcher') return false;
  return true;
}

const isSynergyActiveForPlayer = (p: Raw, rawSyn: string) => {
  if (!p) return false;
  return activeTeamSynergies.value.some(activeSyn => {
    if (!checkSynergyInclusion(activeSyn.name, [rawSyn])) return false;
    const synData = synergys.value.find(s => String(s.synergy).trim() === activeSyn.name);
    if (!synData) return false;
    const synType = getSynergyType(activeSyn.name, synData.conditions); // 🌟 변경
    const isPit = isPitcher(p);
    if (isPit && synType === 'batter') return false;
    if (!isPit && synType === 'pitcher') return false;
    return true;
  });
}

// 🌟 개인 시너지 화면에 하위 호환 시너지를 모두 찾아서 표시해 주는 함수
const getExpandedPlayerSynergies = (p: Raw) => {
  if (!p) return [];
  const rawSyns = getArray(p.synergy);
  const expanded = new Set<string>(rawSyns);
  const isPit = isPitcher(p);
  
  // 게임에 존재하는 모든 시너지를 한 바퀴 돌면서, 이 선수가 조건을 만족하는지 검사
  synergys.value.forEach(s => {
    const name = String(s.synergy).trim();
    const synType = getSynergyType(name, s.conditions); // 🌟 name 파라미터 추가
    
    // 🌟 여기서도 타자/투수가 서로의 시너지를 빼앗아 입는 것을 원천 봉쇄!
    if (isPit && synType === 'batter') return;
    if (!isPit && synType === 'pitcher') return;

    if (checkSynergyInclusion(name, rawSyns)) {
      expanded.add(name);
    }
  });
  
  return Array.from(expanded);
}
// 🌟 개인 시너지 '조건 미달' 시 필요 인원수 텍스트 반환 도우미 함수
const getPendingSynergyText = (synName: string) => {
  const found = pendingTeamSynergies.value.find(s => s.name === synName);
  return found ? `${found.current} / ${found.required}명` : '조건미달';
}
  
// 🌟 기존 코드: 절대 지우지 말고 그대로 두세요! (파워 계산기 엔진)
const getPlayerSynergySum = (p: Raw | null, unit: 'fixed' | 'percent', deckId: 1|2) => {
  if (!p) return 0;
  let total = 0;
  activeTeamSynergiesMap[deckId].value.forEach(syn => {
    if (isPlayerReceivingSynergy(p, syn.name, deckId)) {
      syn.bonuses.forEach(b => {
        if (b.stat === 'power' && b.bonus.unit === unit) total += b.bonus.value;
      });
    }
  });
  return total;
}

// 🌟 1단계 코드: 기존 코드 바로 밑에 "새로 추가" 해주세요! (화면 표시용)
const getPlayerSynergies = (p: Raw) => {
  if (!p || !p.synergy) return []
  return p.synergy.split(',').map(s => s.trim()).filter(Boolean)
}

const calculateTeamPlayerDignityBuff = (p: Raw, deckId: 1|2) => {
  if (!p) return 0;
  const pTeams = toArray(p.team).map(toLowerCase);
  let validTeamIds = new Set<string>(pTeams);
  groupedTeams.filter(g => g.id.some(id => pTeams.includes(id))).forEach(g => g.id.forEach(id => validTeamIds.add(id)));

  let maxTeamPlayerPower = 0;
  let totalDignityPower = 0;

  Object.entries(lineups.value[deckId]).forEach(([slotKey, other]) => {
     if (other) {
        const otherTeams = toArray(other.team).map(toLowerCase);
        if (otherTeams.some(t => validTeamIds.has(t))) {
           const oGrade = String(other.grade).toUpperCase();
           const oBuffs = allPlayerBuffs.value[deckId][slotKey];
           const enhanceLvl = oBuffs?.enhancementLevel || 0;
           if (!oBuffs) return; // 만약 oBuffs가 없다면 (매우 오래된 세이브) 안전하게 넘기기

           if (oGrade === 'TEA') {
             const power = 8 + Math.min(15, Math.max(0, enhanceLvl));
             if (power > maxTeamPlayerPower) maxTeamPlayerPower = power;
           } else if (oGrade === 'DGN') {
             const safeEnhance = Math.min(10, Math.max(0, enhanceLvl));
             const power = safeEnhance === 0 ? 5 : (safeEnhance * 10);
             totalDignityPower += power;
           }
        }
     }
  });

  return maxTeamPlayerPower + totalDignityPower;
}

const getMaxEnhance = (p: Raw) => {
  if (!p) return 15;
  const grade = String(p.grade).toUpperCase();
  return grade === 'DGN' ? 10 : 15;
}

const getMaxBreakthrough = (p: Raw | null) => {
  if (!p) return 0;
  const grade = String(p.grade || '').toUpperCase();
  if (grade === 'DGN') return 0;
  const r = parseInt(String(p.rarity || 1), 10) || 1;
  return r + 1;
}

const getEnhanceMultiplier = (p: Raw) => {
  const grade = String(p.grade).toUpperCase()
  const map: Record<string, number> = { 'SEA':30, 'ASG':30, 'POS':40, 'TEA':40, 'MMVP':40, 'ROY':50, 'HIT':50, 'ACE':50, 'GG':50, 'TOP':50, 'GGY':50, 'DGN':300 }
  return map[grade] || 0
}

const getBreakthroughFixed = (p: Raw, level: number) => {
  if (level === 0 || !p) return 0
  const grade = String(p.grade).toUpperCase()
  if (['SEA','ASG','POS'].includes(grade)) {
    const mults = [0, 1, 3, 6, 10, 15, 21, 28, 36]
    return 30 * (mults[level] || 0)
  } else if (['TEA','ROY','MMVP'].includes(grade)) {
    const mults = [0, 1, 3, 6, 10, 15, 21, 28, 36]
    return 50 * (mults[level] || 0)
  } else if (['HIT','ACE','GG','TOP','GGY'].includes(grade)) {
    const mults = [0, 1, 2.5, 4.5, 7, 10, 15, 21, 28] 
    return 100 * (mults[level] || 0)
  }
  return 0
}
const MANAGER_STATS_MY = [
  { main: 2, sub: 1 }, { main: 5, sub: 2 }, { main: 10, sub: 5 }, { main: 15, sub: 7 }, { main: 20, sub: 10 },
  { main: 50, sub: 25 }, { main: 55, sub: 27 }, { main: 60, sub: 30 }, { main: 65, sub: 32 }, { main: 70, sub: 35 },
  { main: 100, sub: 50 }, { main: 105, sub: 52 }, { main: 110, sub: 55 }, { main: 115, sub: 57 }, { main: 120, sub: 60 },
  { main: 150, sub: 75 }
];

const MANAGER_STATS_COM = [
  { main: 2, sub: 1 }, { main: 5, sub: 2 }, { main: 10, sub: 5 }, { main: 15, sub: 7 }, { main: 20, sub: 10 },
  { main: 45, sub: 20 }, { main: 50, sub: 22 }, { main: 55, sub: 25 }, { main: 60, sub: 27 }, { main: 65, sub: 30 },
  { main: 90, sub: 40 }, { main: 95, sub: 42 }, { main: 100, sub: 45 }, { main: 105, sub: 47 }, { main: 110, sub: 50 },
  { main: 135, sub: 60 }
];

const MANAGER_TYPES: Record<string, { main: string, sub: string }> = {
  '1st': { main: 'contact', sub: 'strikeoutAvoidance' },
  '2nd': { main: 'homeRunPower', sub: 'gapPower' },
  '3rd': { main: 'movement', sub: 'stuff' },
  '4th': { main: 'homeRunSuppression', sub: 'longHitSuppression' }
};

const getManagerBonusText = (typeKey: string, enhance: number) => {
  if (!typeKey) return '';
  const isMy = typeKey.startsWith('my_');
  const t = typeKey.split('_')[1];
  const table = isMy ? MANAGER_STATS_MY : MANAGER_STATS_COM;
  const level = Math.min(15, Math.max(0, enhance || 0));
  const stats = table[level];
  const mainName = STAT_LABELS[MANAGER_TYPES[t].main] || MANAGER_TYPES[t].main;
  const subName = STAT_LABELS[MANAGER_TYPES[t].sub] || MANAGER_TYPES[t].sub;
  return `${mainName} +${stats.main}, ${subName} +${stats.sub}`;
};

// 🌟 9up 인게임 고증: 바인더 파워 자동 계산 도우미 🌟
const calcBinderBonus = (count: number) => {
  if (count === 1) return 10;
  if (count === 2) return 17;
  if (count === 3) return 22;
  if (count === 4) return 25;
  if (count === 5) return 27;
  return 0;
}

// 🌟 이제 매칭 엔진이 선수의 이름, 연도뿐만 아니라 '등급'까지 한 번에 스캔합니다!
const getBinderMatchCount = (val: string, p: Raw, type: string) => {
  if (!val || !p) return false;
  const v = String(val).trim().toLowerCase();
  
  if (type === 'team') {
    const searchTeams = v.split('/').map(s => s.trim());
    return getArray(p.team).some(t => {
      const tName = findTeamName(t).toLowerCase();
      const tRaw = String(t).toLowerCase();
      return searchTeams.some(st => tName.includes(st) || tRaw.includes(st));
    });
  }
  if (type === 'position') {
    const posArr = getArray(p.position).map(normalizePosition); 
    if (v === '선발') return posArr.includes('SP');
    if (v === '계투') return posArr.includes('RP');
    if (v === '내야') return posArr.some(x => ['1B','2B','3B','SS','C'].includes(x));
    if (v === '외야 & 지명' || v === '외야&지명') return posArr.some(x => ['LF','CF','RF','DH'].includes(x));
    return false;
  }
  if (type === 'player') return String(p.name).trim().toLowerCase().includes(v);
  if (type === 'year') {
    // 🌟 핵심 방어막: 탑클래스(TOP) 카드는 연도 매칭에서 무조건 탈락!
    if (String(p.grade || '').toUpperCase() === 'TOP') return false;
    
    return getArray(p.year).some(y => String(y).trim() === v);
  }
  if (type === 'grade') {
    const map: Record<string, string> = {
      '디그니티':'DGN', '탑클래스':'TOP', '에이스':'ACE', '히트':'HIT', '팀플':'TEA',
      '월간mvp':'MMVP', '월간':'MMVP', '신인왕':'ROY', '연도골글':'GGY', '연글':'GGY', 
      '골든글러브':'GG', '골글':'GG', '국가대표':'NT', '올스타':'ASG', '시즌':'SEA', '포스트시즌':'POS'
    }
    const targetG = map[v] || v;
    return String(p.grade).trim().toUpperCase() === targetG.toUpperCase();
  }
  return false;
}

const getPlayerBinderPower = (p: Raw | null, deckId: 1|2) => {
  if (!p) return 0;
  const binderBase = (globalBuffsAll[deckId].binderLevel || 0) * 5; 
  let binderMatrixSum = 0;
  
  if (globalBuffsAll[deckId].binderMatrix && globalBuffsAll[deckId].binderMatrix.length === 5) {
    const matchCounts = { team: 0, position: 0, player: 0, year: 0, grade: 0 };
    globalBuffsAll[deckId].binderMatrix.forEach(row => {
      // 🌟 이제 스탯 쪼가리(p.year)가 아니라 선수 본체(p)를 엔진에 집어넣어 등급까지 깐깐하게 검사합니다!
      if (row.team && getBinderMatchCount(row.team, p, 'team')) matchCounts.team++;
      if (row.position && getBinderMatchCount(row.position, p, 'position')) matchCounts.position++;
      if (row.player && getBinderMatchCount(row.player, p, 'player')) matchCounts.player++;
      if (row.year && getBinderMatchCount(row.year, p, 'year')) matchCounts.year++;
      if (row.grade && getBinderMatchCount(row.grade, p, 'grade')) matchCounts.grade++;
    });
    binderMatrixSum = calcBinderBonus(matchCounts.team) + calcBinderBonus(matchCounts.position) + calcBinderBonus(matchCounts.player) + calcBinderBonus(matchCounts.year) + calcBinderBonus(matchCounts.grade);
  }
  return binderBase + binderMatrixSum;
}

// 🌟 바인더 검색용 데이터 리스트 (자동완성) 🌟
// 🌟 좌측 검색창에서 쓰는 묶음(groupedTeams)을 재활용해서 리스트를 깔끔하게 통합!
const binderTeamOptions = ref(groupedTeams.map(g => g.name)); 
const binderGradeOptions = ref(['디그니티', '탑클래스', '에이스', '히트', '팀플', '월간MVP', '신인왕', '연도골글', '골든글러브', '국가대표', '올스타', '시즌', '포스트시즌']);
// 🌟 전체 선수 DB를 뒤져서 이름과 연도를 자동으로 뽑아옵니다!
const binderPlayerOptions = computed(() => {
  const names = new Set<string>();
  players.value.forEach(p => names.add(p.name));
  return Array.from(names).sort();
});

const binderYearOptions = computed(() => {
  const years = new Set<string>();
  players.value.forEach(p => {
    getArray(p.year).forEach(y => years.add(y));
  });
  return Array.from(years).sort((a, b) => Number(b) - Number(a));
});  
  
const statsDeck1 = computed(() => getComputedPlayerStats(1));
const statsDeck2 = computed(() => getComputedPlayerStats(2));
const computedStatsMap = { 1: statsDeck1, 2: statsDeck2 };
const computedPlayerStats = computed(() => computedStatsMap[activeDeck.value].value);

const getComputedPlayerStats = (deckId: 1 | 2) => {
  const result: Record<string, { power: number, stats: Record<string, number> }> = {}
  Object.keys(lineups.value[deckId]).forEach(slot => {
    const p = lineups.value[deckId][slot]
    if (!p) return
    const buffs = allPlayerBuffs.value[deckId][slot]
    if (!buffs) return

    // 🌟 안전장치: 구버전 세이브파일 호환을 위한 완벽한 기본값 보장
    if (!buffs.careers) buffs.careers = Array(6).fill(null).map(() => ({ grade: '마스터', statType: '', value: 0 }));
    if (!buffs.selectedSkills) buffs.selectedSkills = [];
    if (!buffs.imprintStats) buffs.imprintStats = {};
    if (!buffs.careerStats) buffs.careerStats = {};
    if (buffs.battingOrder === undefined) buffs.battingOrder = null;

    const isPit = isPitcher(p);
    let baseSum = 0
    const coreStats = isPit ? ['movement', 'longHitSuppression', 'homeRunSuppression', 'control', 'stuff'] : ['contact', 'gapPower', 'homeRunPower', 'plateDiscipline', 'strikeoutAvoidance']
    const nonCoreStats = isPit ? ['defense', 'pitchLimit', 'runnerControl'] : ['stealing', 'baseRunning', 'defense']
    coreStats.forEach(s => baseSum += Number(p[s] || 0))
    nonCoreStats.forEach(s => baseSum += Number(p[s] || 0))
    
    const pTeams = toArray(p.team).map(toLowerCase);
    const isMyTeam = (globalBuffsAll[deckId].preferredTeam || []).some(t => pTeams.includes(t));
    const appliedTeamLevelBuff = getTeamLevelPower(globalBuffsAll[deckId].teamLevel, isMyTeam);
    const growthA = Number(Math.max(0, buffs.playerLevel - 1) * 10) + buffs.collectionBuff + appliedTeamLevelBuff + buffs.careerLevelBuff + (buffs.enhancementLevel * getEnhanceMultiplier(p))
    const autoBinderPower = getPlayerBinderPower(p, deckId);
    const legacyBinderBuff = buffs.binderBuff === 537 ? 0 : (buffs.binderBuff || 0);
    const flatC = legacyBinderBuff + autoBinderPower + globalBuffsAll[deckId].clanBuff + buffs.careerAllStatFlat + getBreakthroughFixed(p, buffs.breakthroughLevel)
    
    const is1st2ndSP = slot === 'SP1' || slot === 'SP2';
    const imprintStarterAddedPower = is1st2ndSP ? buffs.imprintStarterPower : 0;
    
    let autoSynergyFixed = 0, autoSynergyPercent = 0, skillPowerPercent = 0, statSpecificSkillPercents: Record<string, number> = {}
    activeTeamSynergiesMap[deckId].value.forEach(syn => {
      if (isPlayerReceivingSynergy(p, syn.name, deckId)) {
        syn.bonuses.forEach(b => {
           if (b.stat === 'power') {
            if (b.bonus.unit === 'fixed') autoSynergyFixed += b.bonus.value
            else if (b.bonus.unit === 'percent') autoSynergyPercent += b.bonus.value
          }
        })
      }
    })
    
    // 🌟 커리어 장착 시스템 스탯/파워 자동 계산 로직 🌟
    const ST = getSameTeamCount(p, deckId);
    let careerStatBonus: Record<string, number> = {};
    let careerGeneralStat = 0; 
    let careerTeamPowerBonus = 0; 
    const careerTypeCounts: Record<string, number> = {};

    buffs.careers.forEach(c => {
       if (!c.statType) return;
       careerTypeCounts[c.statType] = (careerTypeCounts[c.statType] || 0) + 1;
       if (c.statType === '전체 능력치') careerGeneralStat += c.value;
       else if (c.statType === '동일팀 파워') careerTeamPowerBonus += ST * 2;
       else {
           const key = KOR_STAT_MAP[c.statType];
           if (key) careerStatBonus[key] = (careerStatBonus[key] || 0) + c.value;
       }
    });

    // 🌟 커리어 세트 효과 발동 (3칸 이상)
    Object.entries(careerTypeCounts).forEach(([type, count]) => {
       if (count >= 3) {
           if (type === '전체 능력치') careerGeneralStat += (count === 6 ? 30 : count * 6);
           else if (type === '동일팀 파워') careerTeamPowerBonus += count * (ST * 2);
           else {
               const key = KOR_STAT_MAP[type];
               if (key) careerStatBonus[key] = (careerStatBonus[key] || 0) + (count * 25);
           }
       }
    });

    const autoTeamDignityBuff = calculateTeamPlayerDignityBuff(p, deckId);
    const pGrade = String(p.grade || '').toUpperCase();
    const dynamicHitAceBuff = ['HIT', 'ACE', 'GG'].includes(pGrade) ? ST * 32 : 0;
    
    // 기존의 수동 careerTeamCount를 지우고, 커리어 장착 파워(careerTeamPowerBonus)로 대체!
    const growthB = careerTeamPowerBonus + dynamicHitAceBuff + autoTeamDignityBuff + autoSynergyFixed + imprintStarterAddedPower;
    
    buffs.selectedSkills.forEach(s => {
      if (isSkillActive(s, slot, buffs.battingOrder)) {
         const eff = SKILL_EFFECTS[s]
         if (eff) {
           skillPowerPercent += eff.powerPercent || 0
           for (const [k, v] of Object.entries(eff.stats || {})) statSpecificSkillPercents[k] = (statSpecificSkillPercents[k] || 0) + Number(v)
         }
      }
    })
    const globalPercent = skillPowerPercent + autoSynergyPercent + buffs.ultimateImprintPercent
    const globalPercentPool = coreStats.reduce((acc, s) => acc + Number(p[s]||0), 0) + nonCoreStats.reduce((acc, s) => acc + Number(p[s]||0), 0) + growthA
    const globalBonusTotal = globalPercentPool * (globalPercent / 100)
    
    let managerMainStat = 0; let managerSubStat = 0; let managerMainName = ''; let managerSubName = '';
    if (globalBuffsAll[deckId].managerType) {
      const isMy = globalBuffsAll[deckId].managerType.startsWith('my_');
      const typeStr = globalBuffsAll[deckId].managerType.split('_')[1];
      const table = isMy ? MANAGER_STATS_MY : MANAGER_STATS_COM;
      const level = Math.min(15, Math.max(0, globalBuffsAll[deckId].managerEnhance || 0));
      managerMainStat = table[level].main; managerSubStat = table[level].sub;
      managerMainName = MANAGER_TYPES[typeStr].main; managerSubName = MANAGER_TYPES[typeStr].sub;
    }
    
        // 🌟 감독 전술 지시 상시효과 자동 계산 🌟
    let tacticFlat: Record<string, number> = {};
    let tacticCondExpected: Record<string, number> = {};
    
    if (globalBuffsAll[deckId].tacticLevels) {
        const tLv = globalBuffsAll[deckId].tacticLevels;
        const bOrder = buffs.battingOrder;
        const isSp = slot.startsWith('SP');
        const isRp = slot.startsWith('RP');
        const isBatter = !isPitcher(p);
        const isUpper = bOrder === 1 || bOrder === 2;
        const isCleanup = bOrder === 3 || bOrder === 4 || bOrder === 5;
        const isLower = bOrder === 6 || bOrder === 7 || bOrder === 8 || bOrder === 9;
        const is912 = bOrder === 9 || bOrder === 1 || bOrder === 2;
        const posArr = getArray(p.position).map(normalizePosition);
        const isInfield = posArr.some(x => ['1B','2B','3B','SS'].includes(x));
        
        const rates = globalBuffsAll[deckId].tacticCondRates;
        const addCond = (stat: string, val: number) => { tacticCondExpected[stat] = (tacticCondExpected[stat] || 0) + val; };
        
        // 🌟 상시 효과 (Base) 계산
        if (tLv[0] > 0 && isBatter && isCleanup) tacticFlat.homeRunPower = (tacticFlat.homeRunPower || 0) + TACTICS_INFO[0].baseVals[tLv[0]];
        if (tLv[1] > 0 && isBatter && is912) tacticFlat.contact = (tacticFlat.contact || 0) + TACTICS_INFO[1].baseVals[tLv[1]];
        if (tLv[2] > 0 && isBatter && isLower) tacticFlat.contact = (tacticFlat.contact || 0) + TACTICS_INFO[2].baseVals[tLv[2]];
        if (tLv[3] > 0 && isBatter) tacticFlat.plateDiscipline = (tacticFlat.plateDiscipline || 0) + TACTICS_INFO[3].baseVals[tLv[3]];
        if (tLv[4] > 0 && isBatter) tacticFlat.gapPower = (tacticFlat.gapPower || 0) + TACTICS_INFO[4].baseVals[tLv[4]];
        
        if (tLv[5] > 0 && isSp) tacticFlat.control = (tacticFlat.control || 0) + TACTICS_INFO[5].baseVals[tLv[5]];
        if (tLv[6] > 0 && isRp) tacticFlat.pitcherPower = (tacticFlat.pitcherPower || 0) + TACTICS_INFO[6].baseVals[tLv[6]];
        if (tLv[7] > 0 && isSp) tacticFlat.pitchLimit = (tacticFlat.pitchLimit || 0) + TACTICS_INFO[7].baseVals[tLv[7]];
        
        // 상시(조건연동) 효과 - 8번, 12번은 상시 파워에 반영됨 (유저 조절 비율 곱)
        if (tLv[8] > 0 && (isSp || isRp)) tacticFlat.control = (tacticFlat.control || 0) + TACTICS_INFO[8].baseVals[tLv[8]] * (globalBuffsAll[deckId].tacticBaseRates.scoring / 100);
        if (tLv[12] > 0 && (isSp || isRp)) tacticFlat.longHitSuppression = (tacticFlat.longHitSuppression || 0) + TACTICS_INFO[12].baseVals[tLv[12]] * (globalBuffsAll[deckId].tacticBaseRates.cleanup / 100);
        
        if (tLv[9] > 0 && isRp) tacticFlat.pitcherPower = (tacticFlat.pitcherPower || 0) + TACTICS_INFO[9].baseVals[tLv[9]];
        
        if (tLv[10] > 0 && isBatter && (isUpper || isLower)) tacticFlat.plateDiscipline = (tacticFlat.plateDiscipline || 0) + TACTICS_INFO[10].baseVals[tLv[10]];
        if (tLv[11] > 0 && isBatter && (isUpper || isLower)) { tacticFlat.baseRunning = (tacticFlat.baseRunning || 0) + TACTICS_INFO[11].baseVals[tLv[11]]; tacticFlat.stealing = (tacticFlat.stealing || 0) + TACTICS_INFO[11].baseVals[tLv[11]]; }
        if (tLv[13] > 0) tacticFlat.defense = (tacticFlat.defense || 0) + TACTICS_INFO[13].baseVals[tLv[13]];
        if (tLv[14] > 0 && isBatter && isInfield) tacticFlat.defense = (tacticFlat.defense || 0) + TACTICS_INFO[14].baseVals[tLv[14]];

        // 🌟 조건부 달성 효과 (Conditional) 계산 (깡파워에서는 제외, 세부 스탯과 오각형 레이더 차트에만 기댓값으로 반영)
        if (tLv[0] > 0 && isBatter && isLower) addCond('contact', TACTICS_INFO[0].condVals[tLv[0]] * (rates[0] / 100));
        if (tLv[1] > 0 && isBatter && isCleanup) addCond('gapPower', TACTICS_INFO[1].condVals[tLv[1]] * (rates[1] / 100));
        if (tLv[2] > 0 && isBatter && isUpper) addCond('plateDiscipline', TACTICS_INFO[2].condVals[tLv[2]] * (rates[2] / 100));
        if (tLv[3] > 0 && isBatter) addCond('contact', TACTICS_INFO[3].condVals[tLv[3]] * (rates[3] / 100));
        if (tLv[4] > 0 && isBatter) addCond('contact', TACTICS_INFO[4].condVals[tLv[4]] * (rates[4] / 100));
        
        if (tLv[5] > 0) addCond('defense', TACTICS_INFO[5].condVals[tLv[5]] * (rates[5] / 100));
        if (tLv[6] > 0 && isRp) addCond('movement', TACTICS_INFO[6].condVals[tLv[6]] * (rates[6] / 100));
        if (tLv[7] > 0 && isSp) addCond('pitcherPower', TACTICS_INFO[7].condVals[tLv[7]] * (rates[7] / 100));
        if (tLv[8] > 0 && (isSp || isRp)) addCond('homeRunSuppression', TACTICS_INFO[8].condVals[tLv[8]] * (rates[8] / 100));
        if (tLv[9] > 0 && isRp) addCond('movement', TACTICS_INFO[9].condVals[tLv[9]] * (rates[9] / 100));
        
        if (tLv[10] > 0 && isBatter) addCond('batterPower', TACTICS_INFO[10].condVals[tLv[10]] * (rates[10] / 100));
        if (tLv[11] > 0 && isBatter) { addCond('gapPower', TACTICS_INFO[11].condVals[tLv[11]] * (rates[11] / 100)); addCond('homeRunPower', TACTICS_INFO[11].condVals[tLv[11]] * (rates[11] / 100)); }
        if (tLv[12] > 0 && isRp) addCond('movement', TACTICS_INFO[12].condVals[tLv[12]] * (rates[12] / 100));
        if (tLv[13] > 0 && (slot === 'RP3' || slot === 'RP4')) addCond('pitcherPower', TACTICS_INFO[13].condVals[tLv[13]] * (rates[13] / 100));
        if (tLv[14] > 0 && isSp) { addCond('control', TACTICS_INFO[14].condVals[tLv[14]] * (rates[14] / 100)); addCond('stuff', TACTICS_INFO[14].condVals[tLv[14]] * (rates[14] / 100)); }
    }

    let finalTotal = 0
    const stats: Record<string, number> = {}

    let imprintStatBonus: Record<string, number> = {};
    let imprintGeneralPower = 0; 
    
    if (buffs) {
      const applyImp = (imp: any) => {
         if(!imp) return;
         const mainKey = KOR_STAT_MAP[imp.mainStat];
         if (mainKey) imprintStatBonus[mainKey] = (imprintStatBonus[mainKey] || 0) + imp.mainPower;
         else imprintGeneralPower += imp.mainPower;

         imp.subOptions.forEach((opt: any) => {
           if (opt.type === '전체 능력치') {
             coreStats.forEach(c => imprintStatBonus[c] = (imprintStatBonus[c] || 0) + opt.value);
           } else if (opt.type !== '수익 증가') {
             const subKey = KOR_STAT_MAP[opt.type];
             if (subKey) imprintStatBonus[subKey] = (imprintStatBonus[subKey] || 0) + opt.value;
             else imprintGeneralPower += opt.value;
           }
         });
         if (imp.ultimateBonus && pGrade === imp.ultimateBonus.targetGrade) imprintGeneralPower += imp.ultimateBonus.power;
      };
      applyImp(buffs.imprint1); applyImp(buffs.imprint2);
    }

    // 🌟 커리어 전능 스탯 합산
    let coreStatSum = Number(buffs.imprintCoreStat || 0) + Number(buffs.careerCoreStat || 0) + careerGeneralStat;

    coreStats.forEach(s => {
      const base = Number(p[s] || 0)
      let preSpec = base + (growthA/5) + (growthB/5) + (globalBonusTotal/5)
      let specBonus = preSpec * ((statSpecificSkillPercents[s] || 0) / 100)
      let val = preSpec + specBonus + (flatC/5)
      if (s === managerMainName) val += managerMainStat;
      if (s === managerSubName) val += managerSubStat;
      
      val += Number(buffs.careerStats?.[s] || 0) + Number(buffs.imprintStats?.[s] || 0)
      val += coreStatSum + Number(imprintStatBonus[s] || 0) + Number(careerStatBonus[s] || 0) + Number(tacticFlat[s] || 0)

      finalTotal += val
      // 🌟 로비 파워(finalTotal)에는 안 들어가지만, 유저가 보기 편하게 세부 스탯에는 조건부 기댓값을 얹어줍니다.
      stats[s] = Math.round(val + Number(tacticCondExpected[s] || 0))
    })
    
    nonCoreStats.forEach(s => {
      let base = Number(p[s] || 0)
      if (statSpecificSkillPercents[s]) base += base * (statSpecificSkillPercents[s] / 100)
      let val = base
      if (s === managerMainName) val += managerMainStat;
      if (s === managerSubName) val += managerSubStat;
      val += Number(buffs.careerStats?.[s] || 0) + Number(buffs.imprintStats?.[s] || 0)
      val += Number(imprintStatBonus[s] || 0) + Number(tacticFlat[s] || 0)

      finalTotal += val
      stats[s] = Math.round(val + Number(tacticCondExpected[s] || 0))
    })

    finalTotal += imprintGeneralPower + (tacticFlat.pitcherPower || 0);
    
    // 🌟 10번 전술 조건부 타자 파워 기댓값 추가 (로비 파워 제외, 디스플레이용)
    if (!isPit && tacticCondExpected.batterPower) finalTotal += tacticCondExpected.batterPower;
    if (isPit && tacticCondExpected.pitcherPower) finalTotal += tacticCondExpected.pitcherPower;
    result[slot] = { power: Math.round(finalTotal), stats }
  })
  return result
}

const calculatePlayerPower = (p: Raw, slot: string) => computedStatsMap[activeDeck.value].value[slot]?.power || 0

const getDeckTotalPower = (deckId: 1|2) => {
  let sum = 0;
  const stats = computedStatsMap[deckId].value;
  Object.keys(lineups.value[deckId]).forEach(slot => {
    if (slot.startsWith('BENCH')) return;
    sum += stats[slot]?.power || 0;
  });
  return sum;
}
const teamTotalPower = computed(() => {
  let sum = 0
  Object.keys(lineups.value[activeDeck.value]).forEach(slot => {
    if (slot.startsWith('BENCH')) return 
    sum += computedStatsMap[activeDeck.value].value[slot]?.power || 0
  })
  return sum
})

const isSamePlayer = (p1: Raw, p2: Raw) => {
  return p1.id === p2.id;
}

const getAvailableSlot = (basePos: string): string => {
  if (isManualSelection.value && selectedSlot.value && selectedSlot.value.startsWith(basePos)) {
    if (['SP', 'RP', 'BENCH'].includes(basePos)) return selectedSlot.value;
  }
  if (basePos === 'SP') {
    for (let i = 1; i <= 5; i++) if (!lineup.value[`SP${i}` as keyof typeof lineup.value]) return `SP${i}`
    return 'SP1'
  }
  if (basePos === 'RP') {
    for (let i = 1; i <= 6; i++) if (!lineup.value[`RP${i}` as keyof typeof lineup.value]) return `RP${i}`
    return 'RP1'
  }
  if (basePos === 'BENCH') {
    for (let i = 1; i <= 8; i++) if (!lineup.value[`BENCH${i}` as keyof typeof lineup.value]) return `BENCH${i}`
    return 'BENCH1'
  }
  return basePos
}


const isCardInOtherDeck = (p: Raw, currentDeck: 1 | 2) => {
  const otherDeck = currentDeck === 1 ? 2 : 1;
  return Object.values(lineups.value[otherDeck]).some((otherP: any) => otherP && isSamePlayer(otherP, p));
}

// 🌟 2. 클릭으로 배치할 때 포지션 제한 룰 적용 및 탭 자동 이동
const assignPlayerToSlot = (posOrSlot: string, p: Raw) => {
  if (isCardInOtherDeck(p, activeDeck.value)) {
    showToast(`이미 다른 덱(DH${activeDeck.value === 1 ? 2 : 1})에 배치된 동일한 카드입니다!`, 'error');
    return;
  }
  const targetSlot = getAvailableSlot(posOrSlot)

  
  if (!isValidSlotForPlayer(p, targetSlot)) return;

  Object.keys(lineup.value).forEach(k => { 
    if (lineup.value[k] && isSamePlayer(lineup.value[k]!, p)) lineup.value[k] = null 
  })
  
  lineup.value[targetSlot] = p
  initPlayerBuff(targetSlot, p)
  selectedSlot.value = targetSlot
  isManualSelection.value = false 
  rightPanelTab.value = 'player'

  // 🌟 [추가됨] 선수가 들어간 포지션에 맞춰서 화면 탭을 자동으로 전환!
  if (targetSlot.startsWith('SP') || targetSlot.startsWith('RP')) lineupViewMode.value = 'pitcher';
  else if (targetSlot.startsWith('BENCH')) lineupViewMode.value = 'bench';
  else lineupViewMode.value = 'batter';
}

const clearSlot = (pos: string) => {
  // 1. 라인업에서 선수 이름표만 뗍니다.
  lineup.value[pos] = null
  
  // 🌟 핵심: 기존에는 여기에 delete playerBuffs.value[pos] 같은 코드가 있어서 장비까지 날아갔습니다.
  // 이제 선수를 빼더라도 playerBuffs(각인 장비 데이터)는 절대 삭제하지 않고 칸에 그대로 남겨둡니다!

  // 2. 화면 초기화 (선택된 선수 화면 닫기)
  if (selectedSlot.value === pos) {
    selectedSlot.value = null
    rightPanelTab.value = 'global'
  }
}

// 🌟 드래그 앤 드롭으로 자리 맞바꾸기 로직 🌟
const onDragStart = (e: DragEvent, slot: string) => {
  if (!lineup.value[slot]) {
    e.preventDefault();
    return;
  }
  e.dataTransfer?.setData('text/plain', slot);
  if (e.dataTransfer) e.dataTransfer.effectAllowed = 'move';
}

// 🌟 3. 드래그 앤 드롭 시 포지션 제한 룰 적용 및 탭 자동 이동
const onDrop = (e: DragEvent, targetSlot: string) => {
  const sourceSlot = e.dataTransfer?.getData('text/plain');
  if (!sourceSlot || sourceSlot === targetSlot) return;

  const getGroup = (s: string) => s.startsWith('SP') || s.startsWith('RP') ? 'PITCHER' : s.startsWith('BENCH') ? 'BENCH' : 'BATTER';
  const sourceGroup = getGroup(sourceSlot);
  const targetGroup = getGroup(targetSlot);
  
  if (sourceGroup !== targetGroup && sourceGroup !== 'BENCH' && targetGroup !== 'BENCH') return;

  const sourcePlayer = lineup.value[sourceSlot];
  const targetPlayer = lineup.value[targetSlot];

  if (sourcePlayer && !isValidSlotForPlayer(sourcePlayer, targetSlot)) return;
  if (targetPlayer && !isValidSlotForPlayer(targetPlayer, sourceSlot)) return;

  const tempPlayer = lineup.value[targetSlot];
  lineup.value[targetSlot] = lineup.value[sourceSlot];
  lineup.value[sourceSlot] = tempPlayer;

  const tempBuff = playerBuffs.value[targetSlot];
  playerBuffs.value[targetSlot] = playerBuffs.value[sourceSlot];
  playerBuffs.value[sourceSlot] = tempBuff;

  if (selectedSlot.value === sourceSlot) selectedSlot.value = targetSlot;
  else if (selectedSlot.value === targetSlot) selectedSlot.value = sourceSlot;

  // 🌟 [추가됨] 선수가 드롭된 포지션에 맞춰서 화면 탭을 자동으로 전환!
  if (targetSlot.startsWith('SP') || targetSlot.startsWith('RP')) lineupViewMode.value = 'pitcher';
  else if (targetSlot.startsWith('BENCH')) lineupViewMode.value = 'bench';
  else lineupViewMode.value = 'batter';
}
  
const selectSlot = (slot: string) => { 
  selectedSlot.value = slot; 
  isManualSelection.value = true;
  if (lineup.value[slot]) { 
    rightPanelTab.value = 'player'; 
  } else {
    rightPanelTab.value = 'global';
  }
}

// ========================================================
// 📸 [분리 크롭 & 듀얼 머지 엔진] 얼굴/전투력 숫자 100% 차단
// ========================================================
const ocrFileInput = ref<HTMLInputElement | null>(null)
const isOcrProcessing = ref(false)
const ocrProgressText = ref('')

interface OcrDebugItem {
  slot: string
  imgUrl: string
  rawText: string
  matchedName: string | null
  matchedCard: string | null
}
const ocrDebugList = ref<OcrDebugItem[]>([])

// FC 온라인식 카드 교체 모달 상태
const showCardSwapModal = ref(false)
const swapTargetSlot = ref<string | null>(null)

const isSameCard = (c1: Raw | null, c2: Raw | null) => {
  if (!c1 || !c2) return false
  const n1 = String(c1.name || '').trim()
  const n2 = String(c2.name || '').trim()
  const g1 = getMappedGrade(c1.grade)
  const g2 = getMappedGrade(c2.grade)
  const y1 = String(c1.year || '').replace(/[\[\]\s]/g, '')
  const y2 = String(c2.year || '').replace(/[\[\]\s]/g, '')
  return n1 === n2 && g1 === g2 && y1 === y2
}

const swapCandidates = computed(() => {
  if (!swapTargetSlot.value) return []
  const currentP = lineup.value[swapTargetSlot.value]
  if (!currentP) return []
  const cleanName = String(currentP.name || '').replace(/\s+/g, '')
  
  return players.value.filter(p => {
    const isNameMatch = String(p.name || '').replace(/\s+/g, '') === cleanName
    if (!isNameMatch) return false
    return isValidSlotForPlayer(p, swapTargetSlot.value!)
  })
})

const openCardSwapModal = (slot: string) => {
  swapTargetSlot.value = slot
  showCardSwapModal.value = true
}

const applyCardSwap = (newCard: Raw) => {
  if (!swapTargetSlot.value) return
  const slot = swapTargetSlot.value
  
  Object.keys(lineup.value).forEach(k => {
    if (lineup.value[k] && isSameCard(lineup.value[k]!, newCard) && k !== slot) {
      lineup.value[k] = null
    }
  })

  lineup.value[slot] = newCard
  initPlayerBuff(slot, newCard)
  showToast(`[${newCard.name}] 카드가 성공적으로 교체되었습니다!`, 'success')
  showCardSwapModal.value = false
}

// 🌟 [카드 전체 바운딩 박스 정의] 
const OCR_SLOTS = [
  { pos: 'LF', x: 0.190, y: 0.115, w: 0.160, h: 0.35 },
  { pos: 'CF', x: 0.395, y: 0.115, w: 0.160, h: 0.35 },
  { pos: 'RF', x: 0.600, y: 0.115, w: 0.160, h: 0.35 },
  { pos: 'SS', x: 0.295, y: 0.260, w: 0.160, h: 0.35 },
  { pos: '2B', x: 0.495, y: 0.260, w: 0.160, h: 0.35 },
  { pos: '3B', x: 0.190, y: 0.390, w: 0.160, h: 0.35 },
  { pos: '1B', x: 0.600, y: 0.390, w: 0.160, h: 0.35 },
  { pos: 'C',  x: 0.395, y: 0.670, w: 0.160, h: 0.35 },
  { pos: 'DH', x: 0.510, y: 0.670, w: 0.160, h: 0.35 }
]

const triggerOcrInput = () => {
  ocrFileInput.value?.click()
}

// 🌟 [분리 크롭 & 듀얼 머지] 상단 배지 영역과 하단 이름표 영역만 추출 후 세로로 결합
const cropDualCardImages = (image: HTMLImageElement, slot: typeof OCR_SLOTS[0]) => {
  const canvas = document.createElement('canvas')
  const ctx = canvas.getContext('2d')
  if (!ctx) return { dualUrl: null, nameUrl: null }

  const boxX = Math.round(image.naturalWidth * slot.x)
  const boxY = Math.round(image.naturalHeight * slot.y)
  const boxW = Math.round(image.naturalWidth * slot.w)
  const boxH = Math.round(image.naturalHeight * slot.h)

  // 1. 상단 배지/연도 영역 (카드 상단 5% ~ 30%)
  const badgeSy = boxY + Math.round(boxH * 0.05)
  const badgeSh = Math.round(boxH * 0.25)

  // 2. 하단 이름표 영역 (카드 하단 75% ~ 97%)
  const nameSy = boxY + Math.round(boxH * 0.75)
  const nameSh = Math.round(boxH * 0.22)

  const targetW = Math.round(boxW * 3)
  const badgeHScaled = Math.round(badgeSh * 3)
  const nameHScaled = Math.round(nameSh * 3)

  // 두 영역을 위아래로 붙일 캔버스
  canvas.width = targetW
  canvas.height = badgeHScaled + nameHScaled + 15 // 여백 15px

  ctx.imageSmoothingEnabled = true
  ctx.imageSmoothingQuality = 'high'
  ctx.fillStyle = '#050505'
  ctx.fillRect(0, 0, canvas.width, canvas.height)

  // 상단 배지 그리기
  ctx.drawImage(image, boxX, badgeSy, boxW, badgeSh, 0, 0, targetW, badgeHScaled)
  // 하단 이름표 그리기
  ctx.drawImage(image, boxX, nameSy, boxW, nameSh, 0, badgeHScaled + 15, targetW, nameHScaled)

  // 디버그 인스펙터용 이름표 단독 이미지
  const nameCanvas = document.createElement('canvas')
  const nameCtx = nameCanvas.getContext('2d')
  nameCanvas.width = targetW
  nameCanvas.height = nameHScaled
  if (nameCtx) {
    nameCtx.imageSmoothingEnabled = true
    nameCtx.imageSmoothingQuality = 'high'
    nameCtx.fillStyle = '#050505'
    nameCtx.fillRect(0, 0, nameCanvas.width, nameCanvas.height)
    nameCtx.drawImage(image, boxX, nameSy, boxW, nameSh, 0, 0, targetW, nameHScaled)
  }

  return {
    dualUrl: canvas.toDataURL('image/png'),
    nameUrl: nameCanvas ? nameCanvas.toDataURL('image/png') : null
  }
}

// 🌟 [완전 일치 + 투수 배제 엔진]
const processCardSlot = (rawText: string, targetPos: string): { player: Raw | null; name: string | null } => {
  const cleanText = rawText.replace(/[\s\d'’\[\]\(\)\-\.]/g, '')

  // 1단계: 타자 DB 한정 (투수 카드 'SP', 'RP', 'CP' 배제)
  const batterList = players.value.filter(p => !['SP', 'RP', 'CP'].includes(String(p.position).toUpperCase()))

  // 2단계: 선수 이름 100% 완전 일치
  let matchedName = ''
  for (const p of batterList) {
    const pName = String(p.name || '').trim()
    if (pName.length >= 2 && cleanText.includes(pName)) {
      matchedName = pName
      break
    }
  }

  // 2-1. 오독 대비 보조 매칭 (예: '이증범' 등)
  if (!matchedName) {
    for (const p of batterList) {
      const pName = String(p.name || '').trim()
      if (pName.length === 3 && cleanText.includes(pName[0]) && cleanText.includes(pName[2])) {
        matchedName = pName
        break
      }
    }
  }

  if (!matchedName) return { player: null, name: null }

  // 3단계: 해당 타자의 DB 카드 목록
  const candidates = batterList.filter(p => String(p.name || '').trim() === matchedName)
  if (candidates.length <= 1) return { player: candidates[0] || null, name: matchedName }

  // 4단계: 배지 텍스트 파싱
  const upperRaw = rawText.toUpperCase()

  // 연도 추출 ('99, '24, '83 등)
  const detectedYears: string[] = []
  const quoted = rawText.match(/['’](\d{2})/)
  if (quoted) detectedYears.push(quoted[1])
  const yearMatches = Array.from(rawText.matchAll(/\b([89012]\d)\b/g))
  for (const m of yearMatches) {
    if (!detectedYears.includes(m[1])) detectedYears.push(m[1])
  }

  // 등급 추출
  let detectedGrade = ''
  if (/(?:HIT|H1T|H!T|H\|T|HT|HI7|히트)/i.test(upperRaw)) {
    detectedGrade = 'HIT'
  } else if (/(?:TOP|T0P|TDP|TOR|10P|탑)/i.test(upperRaw)) {
    detectedGrade = 'TOP'
  } else if (/(?:DGN|디그|D6N|IGN|OGN)/i.test(upperRaw)) {
    detectedGrade = 'DGN'
  } else if (/(?:GOLDEN|GLOVE|GG|골글)/i.test(upperRaw)) {
    detectedGrade = 'GG'
  }

  // 5단계: 후보군 스코어링
  let bestCard = candidates[0]
  let maxScore = -999

  for (const card of candidates) {
    let score = 0
    const cardGrade = getMappedGrade(card.grade)
    const cardYears = getArray(card.year).map(y => String(y).replace(/\D/g, '').slice(-2))

    if (detectedGrade) {
      if (cardGrade === detectedGrade) score += 40
      else score -= 20
    } else {
      if (cardGrade === 'HIT' || cardGrade === 'TOP') score += 10
      else if (cardGrade === 'DGN') score -= 10
    }

    if (detectedYears.length > 0 && cardYears.some(y => detectedYears.includes(y))) {
      score += 50
    }

    if (isValidSlotForPlayer(card, targetPos)) {
      score += 5
    }

    if (score > maxScore) {
      maxScore = score
      bestCard = card
    }
  }

  return { player: bestCard, name: matchedName }
}

// 🌟 스크린샷 일괄 등록 실행 함수
const handleOcrUpload = async (event: Event) => {
  const target = event.target as HTMLInputElement
  const file = target.files?.[0]
  if (!file) return

  try {
    isOcrProcessing.value = true
    ocrProgressText.value = 'OCR 엔진을 초기화하고 있습니다...'
    ocrDebugList.value = []

    const img = new Image()
    img.src = URL.createObjectURL(file)
    await img.decode()

    const worker = await createWorker('kor+eng')
    let matchedCount = 0

    for (let i = 0; i < OCR_SLOTS.length; i++) {
      const slot = OCR_SLOTS[i]
      ocrProgressText.value = `[${i + 1}/${OCR_SLOTS.length}] ${slot.pos} 슬롯 분석 중...`

      const { dualUrl, nameUrl } = cropDualCardImages(img, slot)
      if (!dualUrl) continue

      const { data: { text } } = await worker.recognize(dualUrl)
      const { player: matchedPlayer, name: foundName } = processCardSlot(text, slot.pos)

      ocrDebugList.value.push({
        slot: slot.pos,
        imgUrl: nameUrl || dualUrl, // 디버그에는 깔끔한 이름표 영역 표시
        rawText: text.trim().replace(/\n+/g, ' '),
        matchedName: foundName,
        matchedCard: matchedPlayer ? `[${matchedPlayer.grade}] ${matchedPlayer.name}` : null
      })

      if (matchedPlayer) {
        Object.keys(lineup.value).forEach(k => {
          if (lineup.value[k] && isSameCard(lineup.value[k]!, matchedPlayer)) {
            lineup.value[k] = null
          }
        })

        lineup.value[slot.pos] = matchedPlayer
        initPlayerBuff(slot.pos, matchedPlayer)
        matchedCount++
      }
    }

    await worker.terminate()
    URL.revokeObjectURL(img.src)

    lineupViewMode.value = 'batter'
    showToast(`라인업 스캔 완료: 총 ${matchedCount}명이 배치되었습니다!`, 'success')
  } catch (err) {
    console.error('OCR 처리 실패:', err)
    showToast('스크린샷을 인식하는 중 오류가 발생했습니다.', 'error')
  } finally {
    isOcrProcessing.value = false
    ocrProgressText.value = ''
    if (ocrFileInput.value) ocrFileInput.value.value = ''
  }
}
  
const fileInput = ref<HTMLInputElement | null>(null)  

// 🌟 라인업 초기화 (각인 장비는 무조건 유지!) 🌟
const resetLineup = () => {
  if(!confirm(`현재 선택된 [DH${activeDeck.value}] 덱의 모든 선수를 비우시겠습니까?\n(포지션에 장착된 각인 수치는 그대로 유지됩니다)`)) return;
  Object.keys(lineups.value[activeDeck.value]).forEach(slot => {
    lineups.value[activeDeck.value][slot] = null;
    if (allPlayerBuffs.value[activeDeck.value][slot]) {
      const b = playerBuffs.value[slot];
      const savedImprintStats = { ...b.imprintStats }
      playerBuffs.value[slot] = {
        enhancementLevel: 15, breakthroughLevel: 0, careerTeamCount: 0, hitAceBuff: 0,
        imprintStarterPower: b.imprintStarterPower, careerAllStatFlat: 0, 
        imprintCoreStat: b.imprintCoreStat, careerCoreStat: 0,
        selectedSkills: [], 
        battingOrder: b.battingOrder, // 타순 유지
        playerLevel: 100, collectionBuff: 1200, careerLevelBuff: 149,
        binderBuff: 537, ultimateImprintPercent: b.ultimateImprintPercent,
        imprintStats: savedImprintStats, careerStats: {},
        imprint1: b.imprint1, // 🌟 여기서도 각인 1 유지!
        imprint2: b.imprint2  // 🌟 여기서도 각인 2 유지!
      }
    }
  })
  selectedSlot.value = null;
  isManualSelection.value = false;
  rightPanelTab.value = 'global';
}

// 🌟 다중 페이지 저장 (이름 지정) 🌟
const saveToLocalStorage = () => {
  const saveName = prompt('저장할 라인업 이름을 입력하세요:\n(예: 국대전용, 홈런타자세팅 등)');
  if (!saveName) return;
  const saveData = { lineups: lineups.value, allPlayerBuffs: allPlayerBuffs.value, globalBuffsAll: globalBuffsAll, imprintInventory: imprintInventory.value }
  let saves = JSON.parse(localStorage.getItem('9up_multi_saves') || '{}')
  saves[saveName] = saveData
  localStorage.setItem('9up_multi_saves', JSON.stringify(saves))
  showToast(`'${saveName}' 라인업이 브라우저에 저장되었습니다.`, 'success');
}

// 🌟 다중 페이지 불러오기 및 관리 (모달 UI 엔진) 🌟
const showSaveManager = ref(false);
const savedLineupsList = ref<string[]>([]);

const openSaveManager = () => {
  const saves = JSON.parse(localStorage.getItem('9up_multi_saves') || '{}');
  savedLineupsList.value = Object.keys(saves);
  showSaveManager.value = true;
};

const loadSpecificSave = (saveName: string) => {
  const saves = JSON.parse(localStorage.getItem('9up_multi_saves') || '{}');
  if (saves[saveName]) {
    applyLoadedData(saves[saveName]);
    showToast(`'${saveName}' 라인업을 성공적으로 불러왔습니다.`, 'success');
    showSaveManager.value = false;
  }
};

const deleteSpecificSave = (saveName: string) => {
  if (!confirm(`정말로 '${saveName}' 라인업을 삭제하시겠습니까?\n(삭제 후에는 복구할 수 없습니다)`)) return;
  const saves = JSON.parse(localStorage.getItem('9up_multi_saves') || '{}');
  delete saves[saveName]; 
  localStorage.setItem('9up_multi_saves', JSON.stringify(saves));
  savedLineupsList.value = Object.keys(saves); // 삭제 후 리스트 실시간 갱신
};

// 🌟 파일 내보내기 (다중 저장 리스트까지 싹 다 파일 안에 압축 저장!) 🌟
const exportToFile = () => {
  const defaultName = `9up_lineup_${new Date().toISOString().slice(0,10)}`
  const fileName = prompt('저장할 파일 이름을 입력하세요:', defaultName)
  if (!fileName) return; 
  
  // 브라우저에 있는 다중 저장 리스트 가져오기
  const multiSaves = JSON.parse(localStorage.getItem('9up_multi_saves') || '{}');

  // 현재 데이터 + 각인 보관함 + 다중 저장 리스트 전부 묶기!
  const saveData = { 
    lineups: lineups.value, 
    allPlayerBuffs: allPlayerBuffs.value, 
    globalBuffsAll: globalBuffsAll, 
    imprintInventory: imprintInventory.value,
    multiSaves: multiSaves 
  }
  
  const blob = new Blob([JSON.stringify(saveData, null, 2)], { type: 'application/json' })
  const url = URL.createObjectURL(blob)
  const a = document.createElement('a')
  a.href = url
  a.download = fileName.endsWith('.json') ? fileName : `${fileName}.json`
  document.body.appendChild(a)
  a.click()
  document.body.removeChild(a)
  URL.revokeObjectURL(url)
}

const triggerFileInput = () => { fileInput.value?.click() }

// 🌟 2. 파일 불러오기 기능 (다중 저장 복구 안내 메시지 추가) 🌟
const importFromFile = (event: Event) => {
  const target = event.target as HTMLInputElement
  const file = target.files?.[0]
  if (!file) return
  
  const reader = new FileReader()
  reader.onload = (e) => {
    try {
      const result = e.target?.result as string
      if (!result) throw new Error("파일이 비어있습니다.");
      
      const data = JSON.parse(result)
      applyLoadedData(data)
      
      // 파일 안에 다중 저장 데이터가 포함되어 있다면 알림창에 내용 추가!
      const msg = data.multiSaves && Object.keys(data.multiSaves).length > 0
        ? '✅ 파일 불러오기 완료!\n(다중 저장 목록도 무사히 복구되었습니다!)'
        : '✅ 파일에서 라인업을 성공적으로 불러왔습니다!';
      showToast(msg, 'success');
      
    } catch (err: any) {
      console.error("파일 파싱 에러:", err)
      showToast('파일 불러오기 실패: 형식이 맞지 않거나 손상된 파일입니다.', 'error');
    } finally {
      if (fileInput.value) fileInput.value.value = ''
    }
  }
  reader.readAsText(file)
}

// 🌟 새로고침/재접속 시 데이터가 날아가지 않도록 실시간 백업 🌟
// 감시망(watch)에 imprintInventory를 추가해서 각인을 만들거나 삭제할 때마다 즉시 자동 저장됩니다.
watch([lineups, allPlayerBuffs, globalBuffsAll, imprintInventory], () => {
  const autoSaveData = { lineups: lineups.value, allPlayerBuffs: allPlayerBuffs.value, globalBuffsAll: globalBuffsAll, imprintInventory: imprintInventory.value }
  localStorage.setItem('9up_auto_save', JSON.stringify(autoSaveData))
}, { deep: true })
  
// 🌟 1. 포지션 룰 적용 (일반=주포지션 1개, 디그니티=멀티포지션 전체)
const getPlayerPositions = (p: Raw) => {
  if (!p) return [];
  const allPos = Array.from(new Set(getArray(p.position).map(normalizePosition))).filter(Boolean);
  
  const grade = String(p.grade || '').toUpperCase();
  if (grade === 'DGN' || grade === 'DIGNITY') {
    return allPos; // 디그니티는 멀티 포지션 전부 허용
  }
  return allPos.length > 0 ? [allPos[0]] : []; // 그 외 카드는 1포지션(주포지션)만 허용
}

// 🌟 [수정됨] 투수/타자 포지션 꼼꼼하게 검증 + 알람 끄기
const isValidSlotForPlayer = (p: Raw | null, slot: string) => {
  if (!p) return true; // 빈 칸은 문제 없음
  if (slot.startsWith('BENCH')) return true; // 벤치는 누구나 휴식 가능
  
  const validPositions = getPlayerPositions(p);

  if (isPitcher(p)) {
    // 투수인데 타자 칸(DH 포함)으로 가는 것 차단
    if (!slot.startsWith('SP') && !slot.startsWith('RP')) return false;
    
    // 선발/계투 확실히 구분 (SP는 SP칸에만, RP는 RP칸에만)
    if (slot.startsWith('SP') && !validPositions.includes('SP')) return false;
    if (slot.startsWith('RP') && !validPositions.includes('RP')) return false;
    
    return true;
  } else {
    // 타자가 투수 칸으로 가는 것 차단
    if (slot.startsWith('SP') || slot.startsWith('RP')) return false; 
    if (slot === 'DH') return true; // 타자는 지명타자(DH) 무조건 가능
    
    return validPositions.includes(slot);
  }
}

const hideImage = (e: Event) => {
  if (e && e.target) {
    const img = e.target as HTMLImageElement;
    console.error("🚨 [이미지 로딩 실패] 파일을 찾지 못했습니다! 실제 폴더/파일명 대소문자를 확인하세요 ➔", img.src);
    // 이미지를 투명하게 숨기지 않고 눈에 띄게 빨간색 테두리를 쳐줍니다.
    img.style.display = 'block'; 
    img.style.border = "2px solid red"; 
  }
}

const activeFilterCount = computed(() => {
  let count = 0;
  count += searchQuery.position.length;
  count += searchQuery.team.length;
  count += searchQuery.synergy.length;
  count += searchQuery.grade.length;
  count += searchQuery.rarity !== null ? 1 : 0;
  return count;
});

// 🌟 1. 강화스킬(패시브)을 장착 스킬 목록에서 완벽 분리
const getAvailableSkills = (p: Raw | null) => {
  if (!p) return [];
  const excluded = ['야전사령관', '인사이드 워크', '투수 리드', '친화력', '도루 저지'];
  // 기존: ...getArray(p.enhancedSkill) 가 섞여 있어서 중복 버그 발생!
  // 수정: 장착 가능한 일반 skill 데이터만 가져오도록 깔끔하게 분리!
  const rawSkills = [...getArray(p.skill)];
  return Array.from(new Set(rawSkills.filter(s => !excluded.includes(s))));
}

const togglePlayerSkill = (sk: string) => {
  if (!selectedSlot.value || !lineup.value[selectedSlot.value] || !playerBuffs.value[selectedSlot.value]) return;
  const p = lineup.value[selectedSlot.value];
  const buffs = playerBuffs.value[selectedSlot.value];
  const arr = buffs.selectedSkills;
  const rarity = parseInt(String(p?.rarity || 1), 10) || 1;
  const max = Math.min(3, Math.max(1, rarity - 1));
  
  const idx = arr.indexOf(sk);
  if (idx > -1) {
    arr.splice(idx, 1);
  } else if (arr.length < max) {
    arr.push(sk);
  }
}

const getMaxSkillCount = (p: Raw | null) => {
  if (!p) return 0;
  return Math.min(3, Math.max(1, (parseInt(String(p?.rarity || 1), 10) || 1) - 1));
}

const formatBonuses = (bonuses: { stat: string, bonus: JsonBonus }[]) => {
  if (!bonuses || !Array.isArray(bonuses)) return '';
  return bonuses.map(b => {
    const statName = b.stat === 'power' ? '파워' : STAT_LABELS[b.stat] || b.stat;
    const val = b.bonus.value;
    const unit = b.bonus.unit === 'percent' ? '%' : '';
    return `${statName} +${val}${unit}`;
  }).join(', ');
}

onMounted(async () => {
  // 🌟 사이트 켜자마자 자동 백업된 데이터 복구 🌟
  const saved = localStorage.getItem('9up_auto_save')
  if (saved) {
    try { applyLoadedData(JSON.parse(saved)) } catch(e) {}
  }

  try {
    const [csvRes, synRes, teamRes, skillRes, enhRes] = await Promise.all([
      fetch('/DB/player_sorted.csv', { cache: 'no-store' }), fetch('/DB/synergys.json', { cache: 'no-store' }), fetch('/DB/setting.json', { cache: 'no-store' }), fetch('/DB/normal_skill.json', { cache: 'no-store' }), fetch('/DB/enhanced_skill.json', { cache: 'no-store' })
    ])
    if (teamRes.ok) teamData.value = await teamRes.json()
    if (skillRes.ok) normalSkillData.value = await skillRes.json()
    if (enhRes.ok) enhancedSkillData.value = await enhRes.json()
    const text = await csvRes.text()
    
    // 원본 코드로 깔끔하게 복구
    Papa.parse(text, { header: true, skipEmptyLines: true, complete: ({ data }) => (players.value = data as Raw[]) })

    if (synRes.ok) {
        const synJson = await synRes.json()
        synergys.value = (Array.isArray(synJson) ? synJson : []).filter((it: any) => Array.isArray(it?.conditions) && it.conditions.length > 0)
        const options: string[] = Array.isArray(synJson) ? synJson.map((item: any) => (typeof item === 'string' ? item : item?.synergy)).filter(Boolean) : []
        synergyOptions.value = Array.from(new Set(options.map(s => String(s).trim()))).sort((a,b)=>a.localeCompare(b))
    }
  } catch(e) { 
    console.error(e) 
  } finally { 
    isLoading.value = false 
  }
})

// 🌟 2. 9up 인게임 고증: 선수 이미지 주소 생성 엔진 (스마트 예외 처리 방식)
const getPlayerImage = (p: Raw | null) => {
  if (!p) return '';
  let grade = String(p.grade || '').trim().toUpperCase();
  const gradeMap: Record<string, string> = {
    '디그니티':'DGN', '탑클래스':'TOP', '에이스':'ACE', '히트':'HIT', '팀플':'TEA',
    '월간MVP':'MMVP', '월간':'MMVP', '신인왕':'ROY', '연도골글':'GGY', '연글':'GGY', 
    '골든글러브':'GG', '골글':'GG', '국가대표':'NT', '올스타':'ASG', '시즌':'SEA', '포스트시즌':'POS'
  };
  grade = gradeMap[grade] || grade;

  let rawTeam = String(Array.isArray(p.team) ? p.team[0] : (p.team || '')).trim().toUpperCase();
  let engTeam = rawTeam;
  const teamKorToEng: Record<string, string> = {
    '두산':'DOOSAN', '기아':'KIA', '해태':'HAITAI', '삼성':'SAMSUNG', '키움':'KIWOOM', '히어로즈':'HEROES', '넥센':'NEXEN', '엘지':'LG', '청룡':'MBC', '롯데':'LOTTE', '한화':'HANWHA', '빙그레':'BINGGRAE', '엔씨':'NC', '케이티':'KT', '현대':'HYUNDAI', '태평양':'PACIFIC', '청보':'CHUNGBO', '삼미':'SAMMI', '쌍방울':'SSANGBANGWOOL'
  };
  if (teamKorToEng[rawTeam]) engTeam = teamKorToEng[rawTeam];

  if (grade === 'DGN') {
    const playerId = String(p.id || p.playerId || 'unknown_id');
    
    // 🌟 [핵심] 겹치는 디그니티 선수 ID 명단 (손승락 등)
    // 이 배열 안에 있는 ID만 팀 이름을 붙여서 찾고, 나머지는 기존처럼 숫자(ID)로만 찾습니다!
    const multiTeamDignityIds = ['11054']; // <-- 예시: ['10123', '20444']
    
    if (multiTeamDignityIds.includes(playerId)) {
      let dgnTeam = engTeam;
      if (dgnTeam === 'OB') dgnTeam = 'DOOSAN';
      if (dgnTeam === 'HAITAI') dgnTeam = 'KIA';
      if (dgnTeam === 'SK') dgnTeam = 'SSG';
      if (dgnTeam === 'NEXEN' || dgnTeam === 'HEROES') dgnTeam = 'KIWOOM';
      if (dgnTeam === 'MBC') dgnTeam = 'LG';
      if (dgnTeam === 'BINGGRAE') dgnTeam = 'HANWHA';
      
      return `/assets/playercards/DGN/${dgnTeam}_${playerId}.png`;
    } else {
      // 🌟 나머지 48명의 일반 디그니티 선수들은 이름 변경 없이 그대로 사용!
      return `/assets/playercards/DGN/${playerId}.png`;
    }
  } 
  
  return `/assets/playercards/commonCard_${grade}_${engTeam}.png`;
}
  
</script>

<template>
  <div class="bg-neutral-50 dark:bg-neutral-900 h-screen overflow-hidden transition-colors flex flex-col font-sans">
    
    <!-- 헤더 영역 -->
    <header class="bg-gradient-to-r from-blue-700 to-indigo-800 text-white shadow-md flex-shrink-0 z-20">
      <div class="w-full px-2 py-1.5 flex items-center justify-between">
        <div class="flex items-center gap-3">
          <Calculator class="w-5 h-5 text-blue-200" />
          <h1 class="text-lg font-bold tracking-tight">9UP 팀 파워 시뮬레이터</h1>
        </div>
        <div class="flex items-center gap-2">
          <input type="file" ref="fileInput" accept=".json" class="hidden" @change="importFromFile" />
          
          <!-- 상단 5종 버튼 메뉴바 -->
          <div class="flex items-center bg-black/20 rounded-lg p-0.5 border border-white/10 shadow-inner">
             <button @click="saveToLocalStorage" class="p-1.5 text-blue-200 hover:text-white hover:bg-white/10 rounded-md transition-colors flex items-center gap-1" title="브라우저에 이름 지정하여 저장"><Save class="w-3.5 h-3.5" /><span class="text-[10px] font-bold hidden sm:block">다중 저장</span></button>
             <button @click="openSaveManager" class="p-1.5 text-blue-200 hover:text-white hover:bg-white/10 rounded-md transition-colors flex items-center gap-1" title="저장된 라인업 관리(불러오기/삭제)"><FolderOpen class="w-3.5 h-3.5" /><span class="text-[10px] font-bold hidden sm:block">불러오기</span></button>
             <div class="w-px h-3 bg-white/20 mx-1"></div>
             <button @click="exportToFile" class="p-1.5 text-emerald-200 hover:text-white hover:bg-white/10 rounded-md transition-colors flex items-center gap-1" title="PC에 파일로 내보내기"><Download class="w-3.5 h-3.5" /><span class="text-[10px] font-bold hidden sm:block">파일 저장</span></button>
             <button @click="triggerFileInput" class="p-1.5 text-emerald-200 hover:text-white hover:bg-white/10 rounded-md transition-colors flex items-center gap-1" title="PC에서 파일 불러오기"><Upload class="w-3.5 h-3.5" /><span class="text-[10px] font-bold hidden sm:block">파일 열기</span></button>
             <div class="w-px h-3 bg-white/20 mx-1"></div>

             <!-- 🌟 신규 추가: 스크린샷 자동 등록 버튼 -->
             <input type="file" ref="ocrFileInput" accept="image/*" class="hidden" @change="handleOcrUpload" />
             <button @click="triggerOcrInput" :disabled="isOcrProcessing" class="p-1.5 text-amber-300 hover:text-white hover:bg-white/10 rounded-md transition-colors flex items-center gap-1" title="인게임 라인업 스크린샷으로 자동 등록">
               <Camera class="w-3.5 h-3.5" />
               <span class="text-[10px] font-bold hidden sm:block">스크린샷 등록</span>
             </button>

             <div class="w-px h-3 bg-white/20 mx-1"></div>
             <button @click="resetLineup" class="p-1.5 text-rose-300 hover:text-white hover:bg-rose-500/50 rounded-md transition-colors flex items-center gap-1" title="각인 유지하고 라인업 초기화"><X class="w-3.5 h-3.5" /><span class="text-[10px] font-bold hidden sm:block">초기화</span></button>
          </div>

          <div class="flex items-center bg-black/20 rounded-xl px-3 py-1.5 border border-white/10 shadow-inner gap-4">
            <div class="flex flex-col items-end">
              <span class="text-indigo-200 text-[10px] font-bold leading-none mb-0.5">DH1 팀 파워</span>
              <span class="text-base font-black text-amber-300 tabular-nums tracking-tight leading-none">{{ getDeckTotalPower(1).toLocaleString() }}</span>
            </div>
            <div class="w-px h-6 bg-white/20"></div>
            <div class="flex flex-col items-end">
              <span class="text-emerald-200 text-[10px] font-bold leading-none mb-0.5">DH2 팀 파워</span>
              <span class="text-base font-black text-emerald-300 tabular-nums tracking-tight leading-none">{{ getDeckTotalPower(2).toLocaleString() }}</span>
            </div>
          </div>
        </div>
      </div>
    </header>

    <div class="w-full px-1 py-1.5 flex-1 flex flex-col min-h-0">
      <div v-if="isLoading" class="flex h-full items-center justify-center">
        <div class="animate-spin rounded-full border-4 border-neutral-300 dark:border-neutral-600 border-t-blue-600 h-10 w-10"></div>
      </div>

      <div v-else class="flex flex-col lg:flex-row gap-1.5 flex-1 min-h-0">
        
<!-- ========================================== -->
        <!-- 왼쪽: 선수 검색 (320px 다이어트 고정) -->
        <!-- ========================================== -->
        <section class="lg:w-[320px] xl:w-[340px] flex-shrink-0 flex flex-col rounded-2xl bg-white dark:bg-neutral-800 border border-neutral-200 dark:border-neutral-700 min-h-0 shadow-sm overflow-hidden">
          <div class="p-3 sm:p-4 border-b border-neutral-200 dark:border-neutral-700 flex flex-col gap-3 bg-neutral-50/50 dark:bg-neutral-800/50">
            
            <!-- 🌟 수정됨: 필터 초기화 버튼 추가 영역 -->
            <div class="flex items-center justify-between">
              <h2 class="font-black text-[15px] sm:text-[17px] text-neutral-800 dark:text-neutral-100 tracking-tight">선수 검색</h2>
              <div class="flex items-center gap-1.5">
                <button v-if="activeFilterCount > 0 || searchQuery.search || searchQuery.rarity !== null" @click="resetFilters" class="flex items-center gap-0.5 px-1.5 py-0.5 text-[10px] font-black text-rose-500 bg-rose-50 hover:bg-rose-100 hover:text-rose-600 border border-rose-200 rounded-md transition-colors shadow-sm">
                  <X class="w-3 h-3" /> 초기화
                </button>
                <span class="text-xs font-semibold text-neutral-400 bg-white dark:bg-neutral-700 px-2 py-0.5 rounded-full border border-neutral-200 dark:border-neutral-600 shadow-sm">{{ filteredPlayers.length.toLocaleString() }}명</span>
              </div>
            </div>
            
            <div class="relative group">
              <div class="absolute inset-y-0 left-0 pl-3 flex items-center pointer-events-none">
                <Search class="h-4 w-4 text-neutral-400 group-focus-within:text-blue-500 transition-colors" />
              </div>
              <input v-model="searchQuery.search" type="text" class="block w-full pl-9 pr-3 py-2 border border-neutral-300 dark:border-neutral-600 rounded-xl leading-5 bg-white dark:bg-neutral-700 placeholder-neutral-400 focus:outline-none focus:ring-2 focus:ring-blue-500 focus:border-blue-500 transition-all shadow-sm text-sm" placeholder="이름, 팀, 포지션, 시너지..." @input="currentPage = 1" />
            </div>

            <!-- 상세 필터 토글 -->
            <button @click="advancedFilterOpen = !advancedFilterOpen" class="flex items-center justify-between w-full px-3 py-2 text-sm bg-white dark:bg-neutral-700 border border-neutral-300 dark:border-neutral-600 rounded-xl hover:bg-neutral-50 dark:hover:bg-neutral-600 transition-colors shadow-sm group">
              <div class="flex items-center gap-2 font-bold text-neutral-600 dark:text-neutral-300 group-hover:text-blue-600 transition-colors">
                <Filter class="w-4 h-4" />
                <span>상세 필터</span>
              </div>
              <div class="flex items-center gap-2">
                <span v-if="activeFilterCount > 0" class="flex h-5 w-5 items-center justify-center rounded-full bg-blue-100 dark:bg-blue-900 text-[10px] font-bold text-blue-600 dark:text-blue-300">{{ activeFilterCount }}</span>
                <ChevronRightIcon class="w-4 h-4 text-neutral-400 transition-transform" :class="{ 'rotate-90': advancedFilterOpen }" />
              </div>
            </button>

            <div v-show="advancedFilterOpen" class="flex flex-col gap-3 pt-2 max-h-[40vh] overflow-y-auto custom-scrollbar pr-1">
              
              <!-- 🌟 희귀도 (선수검색 페이지 스타일 1열 버튼 적용) -->
              <div>
                <label class="block text-[11px] font-bold text-neutral-500 mb-1.5 ml-1">레어도</label>
                <div class="flex items-center justify-between gap-2 bg-white dark:bg-neutral-800 border border-neutral-200 dark:border-neutral-600 rounded-lg p-2">
                  <div class="flex items-center gap-1.5 overflow-x-auto no-scrollbar px-1 flex-1">
                    <div
                      v-for="i in 6" :key="'star-'+i"
                      class="p-1 rounded-md cursor-pointer transition-colors duration-200 hover:bg-neutral-100 dark:hover:bg-neutral-700"
                      :title="`${i}성`"
                      @click="searchQuery.rarity === i ? searchQuery.rarity = null : searchQuery.rarity = i"
                    >
                      <Star
                        class="w-5 h-5 transition-colors duration-200"
                        :class="i <= (searchQuery.rarity ?? 0) ? 'text-amber-400 hover:text-amber-500 fill-amber-400' : 'text-neutral-300 dark:text-neutral-600 hover:text-amber-300 fill-transparent'"
                      />
                    </div>
                  </div>
                  <button
                    v-if="searchQuery.rarity !== null"
                    @click="searchQuery.rarity = null"
                    class="px-2 py-1 text-[10px] rounded-md border border-neutral-200 dark:border-neutral-600 text-neutral-600 dark:text-neutral-300 bg-neutral-50 dark:bg-neutral-700 hover:bg-neutral-100 dark:hover:bg-neutral-600 transition-colors shrink-0"
                  >
                    해제
                  </button>
                </div>
              </div>

              <!-- 🌟 2. 등급 (4칸 3줄) -->
              <div>
                <label class="block text-[11px] font-bold text-neutral-500 mb-1.5 ml-1">등급</label>
                <div class="grid grid-cols-4 gap-1">
                   <button v-for="g in ['DIGNITY', 'TOP CLASS', 'GOLDEN GLOVE', 'ACE PITCHER', 'HIT BATTER', 'GG OF THE YEAR', 'MONTHLY MVP', 'ROOKIE OF THE YEAR', 'TEAM PLAYER', 'POST SEASON', 'ALLSTAR', 'SEASON']" :key="g" 
                           @click="searchQuery.grade.includes(g) ? searchQuery.grade = searchQuery.grade.filter(x => x !== g) : searchQuery.grade.push(g)" 
                           :class="{'ring-2 ring-blue-500 bg-blue-50 dark:bg-blue-900/30': searchQuery.grade.includes(g)}" 
                           class="relative border border-neutral-200 dark:border-neutral-600 rounded-lg p-0 hover:bg-neutral-50 dark:hover:bg-neutral-700 transition-all flex items-center justify-center bg-white dark:bg-neutral-800 h-10 overflow-hidden">
                     <!-- 🌟 수정됨: p-0으로 버튼 여백 삭제, h-10으로 크기 확장, scale-[1.35]로 이미지 강제 줌인! -->
                     <img v-if="getGradeImage(g)" :src="getGradeImage(g)" :alt="g" class="absolute inset-0 w-full h-full object-contain scale-[1.35] drop-shadow-sm pointer-events-none" @error="hideImage" />
                     <span v-else class="text-[9px] font-bold z-10">{{ g }}</span>
                   </button>
                </div>
              </div>

              <!-- 포지션 -->
              <div>
                <label class="block text-[11px] font-bold text-neutral-500 mb-1.5 ml-1">포지션</label>
                <div class="grid grid-cols-4 gap-1">
                  <button v-for="pos in ['1B', '2B', '3B', 'C', 'CF', 'DH', 'LF', 'RF', 'RP', 'SP', 'SS']" :key="pos" 
                          @click="searchQuery.position.includes(pos) ? searchQuery.position = searchQuery.position.filter(x => x !== pos) : searchQuery.position.push(pos)" 
                          :class="{'ring-2 ring-blue-500 bg-blue-50 dark:bg-blue-900/30 text-blue-700 font-bold': searchQuery.position.includes(pos)}" 
                          class="border border-neutral-200 dark:border-neutral-600 rounded-lg py-1 text-[11px] hover:bg-neutral-50 dark:hover:bg-neutral-700 transition-all bg-white dark:bg-neutral-800 text-neutral-600 dark:text-neutral-300">
                    {{ pos }}
                  </button>
                </div>
              </div>

              <!-- 팀 -->
              <div>
                <label class="block text-[11px] font-bold text-neutral-500 mb-1.5 ml-1">팀</label>
                <div class="grid grid-cols-4 gap-1">
                  <button v-for="group in groupedTeams" :key="'filter_'+group.name" 
                          @click="toggleTeamGroup(group)" 
                          :class="{'ring-2 ring-blue-500 bg-blue-50 dark:bg-blue-900/30': isTeamGroupSelected(group)}" 
                          class="border border-neutral-200 dark:border-neutral-600 rounded-lg p-0.5 hover:bg-neutral-50 dark:hover:bg-neutral-700 transition-all flex items-center justify-center bg-white dark:bg-neutral-800 h-7">
                    <img v-if="getTeamLogoUrl(group.id[0])" :src="getTeamLogoUrl(group.id[0])" :alt="group.name" class="h-4 w-auto object-contain" />
                    <span v-else class="text-[9px]">{{ group.name }}</span>
                  </button>
                </div>
              </div>

              <!-- 🌟 시너지 & 스킬 검색 (드롭다운) -->
              <div class="grid grid-cols-1 gap-3">
                
                <!-- 시너지 검색 -->
                <div>
                  <label class="block text-[11px] font-bold text-neutral-500 mb-1.5 ml-1">시너지 검색</label>
                  <div ref="synergyWrapperRef" class="relative flex flex-col gap-1">
                    <input v-model="synergySearchText" @focus="isSynergyDropdownOpen = true" placeholder="시너지를 검색하세요" 
                           class="w-full px-2 py-1.5 text-[11px] border border-neutral-300 dark:border-neutral-600 rounded-lg bg-white dark:bg-neutral-700 outline-none focus:border-blue-500 focus:ring-1 focus:ring-blue-500 shadow-sm" />
                    
                    <div v-show="isSynergyDropdownOpen" class="absolute top-8 left-0 z-50 w-full bg-white dark:bg-neutral-800 border border-blue-200 dark:border-blue-800 rounded-lg shadow-xl max-h-40 overflow-y-auto custom-scrollbar">
                      <button v-for="s in filteredSynergyOptions" :key="s" 
                              @click="toggleSynergyFilter(s); synergySearchText=''; isSynergyDropdownOpen=false;" 
                              class="w-full text-left px-3 py-1.5 text-[11px] text-neutral-700 dark:text-neutral-300 hover:bg-blue-50 dark:hover:bg-blue-900/30 transition-colors border-b border-neutral-100 last:border-0">
                        {{ s }}
                      </button>
                      <div v-if="filteredSynergyOptions.length === 0" class="px-3 py-2 text-center text-[10px] text-neutral-400">검색 결과가 없습니다.</div>
                    </div>

                    <div class="flex flex-wrap gap-1 mt-1">
                       <span v-for="syn in searchQuery.synergy" :key="syn" class="px-2 py-1 text-[10px] bg-blue-100 dark:bg-blue-900/50 text-blue-700 dark:text-blue-300 rounded-md flex items-center gap-1 font-bold border border-blue-200 dark:border-blue-800">
                         {{ syn }} <button @click="toggleSynergyFilter(syn)" class="hover:text-red-500 font-black ml-0.5">&times;</button>
                       </span>
                    </div>
                  </div>
                </div>

                <!-- 🌟 스킬 검색 (인게임 이미지 그리드 적용) -->
                <div>
                  <label class="block text-[11px] font-bold text-amber-500 mb-1.5 ml-1 flex items-center gap-1"><Star class="w-3 h-3 text-amber-400"/>스킬 필터</label>
                  
                  <div class="overflow-y-auto overscroll-contain rounded-lg border border-neutral-200 dark:border-neutral-600 p-1.5 pb-4 bg-neutral-50/50 dark:bg-neutral-800/50 max-h-56 custom-scrollbar">
                    <div class="grid grid-cols-4 gap-1.5">
                      <button
                        v-for="sk in skillOptions" :key="sk"
                        @click="toggleSkillFilter(sk)"
                        @mouseenter="showSkillTooltip($event, sk)"
                        @mouseleave="hideSkillTooltip"
                        class="group relative inline-flex flex-col items-center justify-center gap-1 rounded-xl border py-1.5 text-[10px] font-medium select-none focus:outline-none transition-all duration-200"
                        :class="searchQuery.skill.includes(sk)
                          ? 'bg-amber-500 text-white border-amber-500 shadow-md dark:bg-amber-600 dark:border-amber-600'
                          : 'bg-white dark:bg-neutral-700 text-neutral-700 dark:text-neutral-200 border-neutral-200 dark:border-neutral-600 hover:bg-amber-50 dark:hover:bg-neutral-600'"
                      >
                        <!-- 🌟 실제 스킬 이미지가 렌더링 되는 곳 -->
                        <div class="w-8 h-8 rounded-lg" :class="['bg-neutral-100 dark:bg-neutral-600', searchQuery.skill.includes(sk) ? 'ring-2 ring-white/50 bg-white/20' : '', `bg-${matchSkillInfo(sk)}`]"></div>
                        <span class="block w-full text-center font-semibold truncate px-0.5">{{ sk }}</span>
                      </button>
                    </div>
                  </div>
                </div>
              </div>
            </div>
          </div>

          <!-- 선수 목록 페이지네이션 -->
          <div class="flex items-center justify-between px-4 py-2 border-b border-neutral-100 dark:border-neutral-700 bg-neutral-50/50 dark:bg-neutral-800/30">
            <span class="text-[11px] font-bold text-neutral-500">{{ currentPage }} / {{ totalPages || 1 }} 페이지</span>
            <div class="flex gap-1">
              <button @click="currentPage--" :disabled="currentPage === 1" class="px-2 py-1 text-[11px] border border-neutral-200 dark:border-neutral-600 rounded-md disabled:opacity-30 bg-white dark:bg-neutral-700 hover:bg-neutral-50 dark:hover:bg-neutral-600 font-semibold transition-colors">이전</button>
              <button @click="currentPage++" :disabled="currentPage >= totalPages" class="px-2 py-1 text-[11px] border border-neutral-200 dark:border-neutral-600 rounded-md disabled:opacity-30 bg-white dark:bg-neutral-700 hover:bg-neutral-50 dark:hover:bg-neutral-600 font-semibold transition-colors">다음</button>
            </div>
          </div>

          <!-- 🌟 선수 리스트 (분열 버그 방지 고유 Key 적용됨) -->
          <div class="flex-1 overflow-y-auto p-2 sm:p-3 space-y-2 custom-scrollbar relative">
             <div v-if="paginatedPlayers.length === 0" class="absolute inset-0 flex flex-col items-center justify-center text-neutral-400">
                <Users class="w-10 h-10 mb-2 opacity-20" />
                <p class="text-sm font-bold">검색 결과가 없습니다.</p>
             </div>
             
             <!-- 🌟 고유키(Key)를 완벽하게 조합하여 분신술 렌더링을 차단합니다. -->
             <div v-for="(p, index) in paginatedPlayers" :key="(p.id || p.playerId || '') + '_' + p.name + '_' + p.grade + '_' + index" class="border border-neutral-200 dark:border-neutral-700 rounded-xl p-2 bg-white dark:bg-neutral-800 shadow-sm hover:shadow-md transition-all group flex flex-col gap-2">
                <div class="flex items-center gap-3">
                   <div class="w-12 h-12 rounded-lg border border-neutral-200 dark:border-neutral-700 bg-neutral-50 dark:bg-neutral-700/50 flex items-center justify-center flex-shrink-0 overflow-hidden shadow-inner">
                      <img v-if="getGradeImage(p.grade)" :src="getGradeImage(p.grade)" :alt="p.grade" class="w-10 object-contain drop-shadow-sm" @error="hideImage" />
                      <span v-else class="text-[9px] font-black text-neutral-400">{{ p.grade }}</span>
                   </div>
                   <div class="flex-1 min-w-0">
                     <div class="flex items-center gap-1.5 mb-0.5">
                        <span class="font-black text-[14px] text-neutral-900 dark:text-neutral-100 truncate tracking-tight">{{ p.name }}</span>
                        <div class="flex">
                           <Star v-for="i in Number(p.rarity || 1)" :key="i" class="w-2.5 h-2.5 fill-amber-400 text-amber-400" />
                        </div>
                     </div>
                     <div class="flex items-center gap-1 text-[11px] text-neutral-500 font-medium">
                        <img v-if="getTeamLogoUrl(Array.isArray(p.team) ? p.team[0] : p.team)" :src="getTeamLogoUrl(Array.isArray(p.team) ? p.team[0] : p.team)" class="h-3.5 w-auto" @error="hideImage" />
                        <span v-else>{{ Array.isArray(p.team) ? p.team[0] : p.team }}</span>
                        <span class="font-bold text-neutral-700 dark:text-neutral-300 ml-0.5 whitespace-nowrap">{{ findTeamName(Array.isArray(p.team) ? p.team[0] : p.team) }}</span>
                        <!-- 🌟 선수 연도 표시 추가 -->
                        <span v-if="p.year && String(p.year) !== 'NaN' && String(p.year) !== '0'" class="font-bold text-neutral-500 ml-0.5 whitespace-nowrap truncate">
                           · [{{ String(p.year).replace(/[\[\]]/g, '').split(',').join(', ') }}]
                        </span>
                     </div>
                   </div>
                </div>
                
                <div class="flex flex-wrap gap-1 mt-0.5 pl-[60px]">
                   <button v-for="pos in getPlayerPositions(p)" :key="pos" draggable="true" @dragstart="onDragStart($event, pos)" 
                           @click="assignPlayerToSlot(pos, p)" 
                           class="px-2 py-0.5 rounded-md text-[10px] font-bold border cursor-grab active:cursor-grabbing hover:-translate-y-0.5 transition-transform bg-white dark:bg-neutral-700 shadow-sm" :class="isPitcher(p) ? 'border-rose-200 text-rose-600 dark:border-rose-800 dark:text-rose-400' : 'border-indigo-200 text-indigo-600 dark:border-indigo-800 dark:text-indigo-400'">
                     {{ pos }}
                   </button>
                   <button v-if="!isPitcher(p)" draggable="true" @dragstart="onDragStart($event, 'DH')" 
                           @click="assignPlayerToSlot('DH', p)" 
                           class="px-2 py-0.5 rounded-md text-[10px] font-bold border border-emerald-200 text-emerald-600 bg-white shadow-sm cursor-grab hover:-translate-y-0.5 transition-transform dark:bg-neutral-700 dark:border-emerald-800 dark:text-emerald-400">DH</button>
                   <button draggable="true" @dragstart="onDragStart($event, 'BENCH')" 
                           @click="assignPlayerToSlot('BENCH', p)" 
                           class="px-2 py-0.5 rounded-md text-[10px] font-bold border border-neutral-200 text-neutral-500 bg-white shadow-sm cursor-grab hover:-translate-y-0.5 transition-transform dark:bg-neutral-700 dark:border-neutral-600 dark:text-neutral-400">벤치</button>
                </div>
             </div>
          </div>
        </section>
        
        <!-- ========================================== -->
        <!-- 중앙: 라인업 보드 (flex-1 독식) -->
        <!-- ========================================== -->
        <section class="flex-1 flex flex-col rounded-2xl bg-white dark:bg-neutral-800 border border-neutral-200 dark:border-neutral-700 min-h-0 shadow-sm overflow-hidden relative">
          <div class="flex items-center bg-neutral-100 dark:bg-neutral-700/50 p-1 border-b border-neutral-200 dark:border-neutral-700 flex-shrink-0 gap-1.5 overflow-x-auto no-scrollbar">
            <div class="flex bg-neutral-200/80 dark:bg-neutral-800 rounded-lg p-0.5 shrink-0">
              <button @click="activeDeck = 1" :class="activeDeck === 1 ? 'bg-indigo-600 text-white shadow-sm font-bold' : 'text-neutral-500 hover:bg-neutral-300 dark:hover:bg-neutral-700 font-medium'" class="px-3 py-1.5 text-[11px] sm:text-xs rounded-md transition-all whitespace-nowrap">DH1 덱</button>
              <button @click="activeDeck = 2" :class="activeDeck === 2 ? 'bg-emerald-600 text-white shadow-sm font-bold' : 'text-neutral-500 hover:bg-neutral-300 dark:hover:bg-neutral-700 font-medium'" class="px-3 py-1.5 text-[11px] sm:text-xs rounded-md transition-all whitespace-nowrap">DH2 덱</button>
            </div>
            <div class="w-px h-4 bg-neutral-300 dark:bg-neutral-600 shrink-0 hidden sm:block"></div>
            <div class="flex flex-1 gap-0.5">
              <button @click="lineupViewMode = 'batter'" :class="lineupViewMode === 'batter' ? 'bg-white dark:bg-neutral-600 shadow-sm font-bold text-blue-600' : 'text-neutral-500'" class="flex-1 py-1.5 text-[11px] sm:text-xs rounded-lg transition-all whitespace-nowrap">타자 라인업</button>
              <button @click="lineupViewMode = 'pitcher'" :class="lineupViewMode === 'pitcher' ? 'bg-white dark:bg-neutral-600 shadow-sm font-bold text-blue-600' : 'text-neutral-500'" class="flex-1 py-1.5 text-[11px] sm:text-xs rounded-lg transition-all whitespace-nowrap">투수 라인업</button>
              <button @click="lineupViewMode = 'bench'" :class="lineupViewMode === 'bench' ? 'bg-white dark:bg-neutral-600 shadow-sm font-bold text-blue-600' : 'text-neutral-500'" class="flex-1 py-1.5 text-[11px] sm:text-xs rounded-lg transition-all whitespace-nowrap">벤치</button>
            </div>
          </div>

          <div class="flex-1 overflow-hidden p-2 bg-neutral-50/30 dark:bg-neutral-900/30 flex flex-col items-center justify-center">
            
            <!-- ⚾ 타자 다이아몬드 UI -->
            <div v-if="lineupViewMode === 'batter'" class="w-full h-full flex flex-col justify-center items-center gap-1 sm:gap-2 py-1">
               <div class="flex-1 w-full flex justify-center items-center gap-1 sm:gap-2 min-h-0">
                 <div v-for="pos in ['LF', 'CF', 'RF']" :key="pos" @dragover.prevent @drop="onDrop($event, pos)" class="flex-1 max-w-[24%] h-full flex justify-center items-center min-w-0 min-h-0">
                   <div v-if="!lineup[pos]" class="relative h-full max-w-full aspect-[5/7] border border-dashed rounded-xl flex flex-col items-center justify-center cursor-pointer transition-all border-neutral-300 dark:border-neutral-600 bg-neutral-50/50 dark:bg-neutral-800/30 hover:bg-neutral-100 dark:hover:bg-neutral-700/50 text-neutral-400" :class="{'border-indigo-500 bg-indigo-50 dark:bg-indigo-900/30': selectedSlot === pos}" @click="selectSlot(pos)"><span class="text-[12px] font-bold">{{ pos }}</span></div>
                   <div v-else draggable="true" @dragstart="onDragStart($event, pos)" class="relative h-full max-w-full aspect-[5/7] border rounded-xl flex flex-col items-center p-0 cursor-pointer transition-all shadow-sm group overflow-hidden bg-white dark:bg-neutral-800" :class="{'border-indigo-500 ring-2 ring-indigo-400': selectedSlot === pos, 'border-neutral-200 dark:border-neutral-600': selectedSlot !== pos}" @click="selectSlot(pos)">
                      <!-- ✅ 수정 후 (× 버튼 바로 위에 🔁 버튼 추가) -->
<div class="absolute top-2 left-2 text-xs font-black text-white drop-shadow-[0_1px_3px_rgba(0,0,0,1)] z-10">{{ pos }}</div>
<button class="absolute top-1.5 right-8 w-6 h-6 rounded-full bg-black/50 text-white hover:bg-indigo-600 flex items-center justify-center text-[11px] opacity-0 group-hover:opacity-100 transition-opacity z-20 backdrop-blur-sm shadow-sm" @click.stop="openCardSwapModal(pos)" title="다른 시즌 카드로 교체">
  <RefreshCw class="w-3 h-3" />
</button>
<button class="absolute top-1.5 right-1.5 w-6 h-6 rounded-full bg-black/50 text-white hover:bg-red-500 flex items-center justify-center text-[14px] opacity-0 group-hover:opacity-100 transition-opacity z-20 backdrop-blur-sm" @click.stop="clearSlot(pos)">×</button>                      <div v-if="playerBuffs[pos]?.battingOrder && !isPitcher(lineup[pos])" class="absolute top-2 left-1/2 -translate-x-1/2 text-[10px] font-bold bg-orange-600 text-white px-2 py-0.5 rounded shadow-md z-10">{{ playerBuffs[pos].battingOrder }}번</div>
                      <img :src="getPlayerImage(lineup[pos])" class="absolute inset-0 w-full h-full object-contain" @error="hideImage" />
                      <div class="absolute bottom-0 inset-x-0 h-[45%] bg-gradient-to-t from-black/95 via-black/50 to-transparent flex flex-col justify-end items-center pb-2 px-1 pointer-events-none">
                         <div class="text-[11px] sm:text-[13px] font-bold text-white w-full flex items-baseline justify-center truncate drop-shadow-md leading-tight">
                           {{ lineup[pos].name }}
                           <span v-if="lineup[pos].year && String(lineup[pos].year) !== 'NaN' && String(lineup[pos].year) !== '0' && !['TOP', '탑클래스'].includes(String(lineup[pos].grade).toUpperCase())" class="text-neutral-300 ml-0.5 drop-shadow-sm">'{{ String(lineup[pos].year).replace(/[\[\]]/g, '').split(',')[0].trim().slice(-2) }}</span>
                         </div>
                         <div class="text-[13px] sm:text-[15px] font-black text-amber-400 tracking-tight drop-shadow-md leading-tight mt-0.5">{{ calculatePlayerPower(lineup[pos], pos).toLocaleString() }}</div>
                      </div>
                   </div>
                 </div>
               </div>

               <div class="flex-1 w-full flex justify-center items-center gap-1 sm:gap-2 min-h-0">
                 <div v-for="pos in ['3B', 'SS', '2B', '1B']" :key="pos" @dragover.prevent @drop="onDrop($event, pos)" class="flex-1 max-w-[24%] h-full flex justify-center items-center min-w-0 min-h-0">
                   <div v-if="!lineup[pos]" class="relative h-full max-w-full aspect-[5/7] border border-dashed rounded-xl flex flex-col items-center justify-center cursor-pointer transition-all border-neutral-300 dark:border-neutral-600 bg-neutral-50/50 dark:bg-neutral-800/30 hover:bg-neutral-100 dark:hover:bg-neutral-700/50 text-neutral-400" :class="{'border-indigo-500 bg-indigo-50 dark:bg-indigo-900/30': selectedSlot === pos}" @click="selectSlot(pos)"><span class="text-[12px] font-bold">{{ pos }}</span></div>
                   <div v-else draggable="true" @dragstart="onDragStart($event, pos)" class="relative h-full max-w-full aspect-[5/7] border rounded-xl flex flex-col items-center p-0 cursor-pointer transition-all shadow-sm group overflow-hidden bg-white dark:bg-neutral-800" :class="{'border-indigo-500 ring-2 ring-indigo-400': selectedSlot === pos, 'border-neutral-200 dark:border-neutral-600': selectedSlot !== pos}" @click="selectSlot(pos)">
                      <!-- ✅ 수정 후 -->
<div class="absolute top-2 left-2 text-xs font-black text-white drop-shadow-[0_1px_3px_rgba(0,0,0,1)] z-10">{{ pos }}</div>
<button class="absolute top-1.5 right-8 w-6 h-6 rounded-full bg-black/50 text-white hover:bg-indigo-600 flex items-center justify-center text-[11px] opacity-0 group-hover:opacity-100 transition-opacity z-20 backdrop-blur-sm shadow-sm" @click.stop="openCardSwapModal(pos)" title="다른 시즌 카드로 교체">
  <RefreshCw class="w-3 h-3" />
</button>
<button class="absolute top-1.5 right-1.5 w-6 h-6 rounded-full bg-black/50 text-white hover:bg-red-500 flex items-center justify-center text-[14px] opacity-0 group-hover:opacity-100 transition-opacity z-20 backdrop-blur-sm" @click.stop="clearSlot(pos)">×</button>                      <div v-if="playerBuffs[pos]?.battingOrder && !isPitcher(lineup[pos])" class="absolute top-2 left-1/2 -translate-x-1/2 text-[10px] font-bold bg-orange-600 text-white px-2 py-0.5 rounded shadow-md z-10">{{ playerBuffs[pos].battingOrder }}번</div>
                      <img :src="getPlayerImage(lineup[pos])" class="absolute inset-0 w-full h-full object-contain" @error="hideImage" />
                      <div class="absolute bottom-0 inset-x-0 h-[45%] bg-gradient-to-t from-black/95 via-black/50 to-transparent flex flex-col justify-end items-center pb-2 px-1 pointer-events-none">
                         <div class="text-[11px] sm:text-[13px] font-bold text-white w-full flex items-baseline justify-center truncate drop-shadow-md leading-tight">
                           {{ lineup[pos].name }}
                           <span v-if="lineup[pos].year && String(lineup[pos].year) !== 'NaN' && String(lineup[pos].year) !== '0' && !['TOP', '탑클래스'].includes(String(lineup[pos].grade).toUpperCase())" class="text-neutral-300 ml-0.5 drop-shadow-sm">'{{ String(lineup[pos].year).replace(/[\[\]]/g, '').split(',')[0].trim().slice(-2) }}</span>
                         </div>
                         <div class="text-[13px] sm:text-[15px] font-black text-amber-400 tracking-tight drop-shadow-md leading-tight mt-0.5">{{ calculatePlayerPower(lineup[pos], pos).toLocaleString() }}</div>
                      </div>
                   </div>
                 </div>
               </div>

               <!-- 🌟 포수(C), 지명타자(DH), 타순 변경 버튼 -->
               <div class="flex-1 w-full flex justify-center items-center gap-6 sm:gap-10 min-h-0">
                 <div v-for="pos in ['C', 'DH']" :key="pos" @dragover.prevent @drop="onDrop($event, pos)" class="flex-1 max-w-[24%] h-full flex justify-center items-center min-w-0 min-h-0">
                   <div v-if="!lineup[pos]" class="relative h-full max-w-full aspect-[5/7] border border-dashed rounded-xl flex flex-col items-center justify-center cursor-pointer transition-all border-neutral-300 dark:border-neutral-600 bg-neutral-50/50 dark:bg-neutral-800/30 hover:bg-neutral-100 dark:hover:bg-neutral-700/50 text-neutral-400" :class="{'border-indigo-500 bg-indigo-50 dark:bg-indigo-900/30': selectedSlot === pos}" @click="selectSlot(pos)"><span class="text-[12px] font-bold">{{ pos }}</span></div>
                   <div v-else draggable="true" @dragstart="onDragStart($event, pos)" class="relative h-full max-w-full aspect-[5/7] border rounded-xl flex flex-col items-center p-0 cursor-pointer transition-all shadow-sm group overflow-hidden bg-white dark:bg-neutral-800" :class="{'border-indigo-500 ring-2 ring-indigo-400': selectedSlot === pos, 'border-neutral-200 dark:border-neutral-600': selectedSlot !== pos}" @click="selectSlot(pos)">
                      <!-- ✅ 수정 후 -->
<div class="absolute top-2 left-2 text-xs font-black text-white drop-shadow-[0_1px_3px_rgba(0,0,0,1)] z-10">{{ pos }}</div>
<button class="absolute top-1.5 right-8 w-6 h-6 rounded-full bg-black/50 text-white hover:bg-indigo-600 flex items-center justify-center text-[11px] opacity-0 group-hover:opacity-100 transition-opacity z-20 backdrop-blur-sm shadow-sm" @click.stop="openCardSwapModal(pos)" title="다른 시즌 카드로 교체">
  <RefreshCw class="w-3 h-3" />
</button>
<button class="absolute top-1.5 right-1.5 w-6 h-6 rounded-full bg-black/50 text-white hover:bg-red-500 flex items-center justify-center text-[14px] opacity-0 group-hover:opacity-100 transition-opacity z-20 backdrop-blur-sm" @click.stop="clearSlot(pos)">×</button>                      <div v-if="playerBuffs[pos]?.battingOrder && !isPitcher(lineup[pos])" class="absolute top-2 left-1/2 -translate-x-1/2 text-[10px] font-bold bg-orange-600 text-white px-2 py-0.5 rounded shadow-md z-10">{{ playerBuffs[pos].battingOrder }}번</div>
                      <img :src="getPlayerImage(lineup[pos])" class="absolute inset-0 w-full h-full object-contain" @error="hideImage" />
                      <div class="absolute bottom-0 inset-x-0 h-[45%] bg-gradient-to-t from-black/95 via-black/50 to-transparent flex flex-col justify-end items-center pb-2 px-1 pointer-events-none">
                         <div class="text-[11px] sm:text-[13px] font-bold text-white w-full flex items-baseline justify-center truncate drop-shadow-md leading-tight">
                           {{ lineup[pos].name }}
                           <span v-if="lineup[pos].year && String(lineup[pos].year) !== 'NaN' && String(lineup[pos].year) !== '0' && !['TOP', '탑클래스'].includes(String(lineup[pos].grade).toUpperCase())" class="text-neutral-300 ml-0.5 drop-shadow-sm">'{{ String(lineup[pos].year).replace(/[\[\]]/g, '').split(',')[0].trim().slice(-2) }}</span>
                         </div>
                         <div class="text-[13px] sm:text-[15px] font-black text-amber-400 tracking-tight drop-shadow-md leading-tight mt-0.5">{{ calculatePlayerPower(lineup[pos], pos).toLocaleString() }}</div>
                      </div>
                   </div>
                 </div>

                 <!-- 🌟 타순 변경 전용 카드 버튼 -->
                 <div class="flex-1 max-w-[24%] h-full flex justify-center items-center min-w-0 min-h-0 pl-2 sm:pl-4">
                   <button @click="showBattingOrderManager = true" class="relative w-full h-full max-w-full aspect-[5/7] border-2 border-indigo-300 dark:border-indigo-700 border-dashed rounded-xl flex flex-col items-center justify-center cursor-pointer transition-all bg-indigo-50/50 dark:bg-indigo-900/20 hover:bg-indigo-100 dark:hover:bg-indigo-900/50 text-indigo-500 dark:text-indigo-400 group shadow-sm">
                      <div class="w-8 h-8 sm:w-10 sm:h-10 rounded-full bg-indigo-200 dark:bg-indigo-800 flex items-center justify-center mb-1 sm:mb-2 group-hover:scale-110 transition-transform">
                         <Users class="w-4 h-4 sm:w-5 sm:h-5 text-indigo-700 dark:text-indigo-300" />
                      </div>
                      <span class="text-[10px] sm:text-xs font-black tracking-tight">타순 변경</span>
                   </button>
                 </div>
               </div>
            </div>

            <!-- ⚾ 투수 UI -->
            <div v-else-if="lineupViewMode === 'pitcher'" class="w-full h-full flex flex-col justify-start gap-4 sm:gap-6 py-2">
              <div class="w-full flex-1 flex flex-col min-h-0">
                <h3 class="text-xs font-bold text-neutral-500 px-2 mb-2 shrink-0">선발 투수</h3>
                <div class="flex-1 w-full flex justify-center items-start gap-1 sm:gap-1.5 min-h-0">
                  <div v-for="(role, index) in ['1선발', '2선발', '3선발', '4선발', '5선발']" :key="'SP'+(index+1)" @dragover.prevent @drop="onDrop($event, 'SP'+(index+1))" class="flex-1 max-w-[19.6%] h-full flex flex-col justify-start items-center min-w-0 min-h-0 gap-1.5">
                   <div class="text-[11px] sm:text-[12px] font-black text-indigo-700 dark:text-indigo-400 tracking-tight shrink-0">{{ role }}</div>
                   <div v-if="!lineup['SP'+(index+1)]" class="relative h-full max-w-full aspect-[5/7] border border-dashed rounded-xl flex flex-col items-center justify-center cursor-pointer transition-all border-neutral-300 dark:border-neutral-600 bg-neutral-50/50 dark:bg-neutral-800/30 hover:bg-neutral-100 dark:hover:bg-neutral-700/50 text-neutral-400" :class="{'border-indigo-500 bg-indigo-50 dark:bg-indigo-900/30': selectedSlot === 'SP'+(index+1)}" @click="selectSlot('SP'+(index+1))">
                      <span class="text-[24px] font-black opacity-30">+</span>
                   </div>
                   <div v-else draggable="true" @dragstart="onDragStart($event, 'SP'+(index+1))" class="relative h-full max-w-full aspect-[5/7] border rounded-xl flex flex-col items-center p-0 cursor-pointer transition-all shadow-sm group overflow-hidden bg-neutral-100 dark:bg-neutral-800" :class="{'border-indigo-500 ring-2 ring-indigo-400': selectedSlot === 'SP'+(index+1), 'border-neutral-200 dark:border-neutral-600': selectedSlot !== 'SP'+(index+1)}" @click="selectSlot('SP'+(index+1))">
                      <button class="absolute top-1.5 right-1.5 w-6 h-6 rounded-full bg-black/50 text-white hover:bg-red-500 flex items-center justify-center text-[14px] opacity-0 group-hover:opacity-100 transition-opacity z-20 backdrop-blur-sm" @click.stop="clearSlot('SP'+(index+1))">×</button>
                      <img :src="getPlayerImage(lineup['SP'+(index+1)])" class="absolute inset-0 w-full h-full object-cover" @error="hideImage" />
                      <div class="absolute bottom-0 inset-x-0 h-[45%] bg-gradient-to-t from-black/95 via-black/50 to-transparent flex flex-col justify-end items-center pb-2 px-1 pointer-events-none">
                         <div class="text-[11px] sm:text-[13px] font-bold text-white w-full flex items-baseline justify-center truncate drop-shadow-md leading-tight">
                           {{ lineup['SP'+(index+1)].name }}
                           <span v-if="lineup['SP'+(index+1)].year && String(lineup['SP'+(index+1)].year) !== 'NaN' && String(lineup['SP'+(index+1)].year) !== '0' && !['TOP', '탑클래스'].includes(String(lineup['SP'+(index+1)].grade).toUpperCase())" class="text-neutral-300 ml-0.5 drop-shadow-sm">'{{ String(lineup['SP'+(index+1)].year).replace(/[\[\]]/g, '').split(',')[0].trim().slice(-2) }}</span>
                         </div>
                         <div class="text-[13px] sm:text-[15px] font-black text-amber-400 tracking-tight drop-shadow-md leading-tight mt-0.5">{{ calculatePlayerPower(lineup['SP'+(index+1)], 'SP'+(index+1)).toLocaleString() }}</div>
                      </div>
                   </div>
                  </div>
                </div>
              </div>
              
              <div class="w-full flex-1 flex flex-col min-h-0">
                <h3 class="text-xs font-bold text-neutral-500 px-2 mb-2 shrink-0">계투 및 마무리</h3>
                <div class="flex-1 w-full flex justify-center items-start gap-1 sm:gap-1.5 min-h-0">
                  <div v-for="(role, index) in ['승리 계투', '숏 릴리프', '셋업', '마무리', '롱 맨', '추격조']" :key="'RP'+(index+1)" @dragover.prevent @drop="onDrop($event, 'RP'+(index+1))" class="flex-1 max-w-[16.4%] h-full flex flex-col justify-start items-center min-w-0 min-h-0 gap-1.5">
                   <div class="text-[10px] sm:text-[11px] font-black text-indigo-700 dark:text-indigo-400 tracking-tight shrink-0 whitespace-nowrap">{{ role }}</div>
                   <div v-if="!lineup['RP'+(index+1)]" class="relative h-full max-w-full aspect-[5/7] border border-dashed rounded-xl flex flex-col items-center justify-center cursor-pointer transition-all border-neutral-300 dark:border-neutral-600 bg-neutral-50/50 dark:bg-neutral-800/30 hover:bg-neutral-100 dark:hover:bg-neutral-700/50 text-neutral-400" :class="{'border-indigo-500 bg-indigo-50 dark:bg-indigo-900/30': selectedSlot === 'RP'+(index+1)}" @click="selectSlot('RP'+(index+1))">
                       <span class="text-[20px] font-black opacity-30">+</span>
                   </div>
                   <div v-else draggable="true" @dragstart="onDragStart($event, 'RP'+(index+1))" class="relative h-full max-w-full aspect-[5/7] border rounded-xl flex flex-col items-center p-0 cursor-pointer transition-all shadow-sm group overflow-hidden bg-neutral-100 dark:bg-neutral-800" :class="{'border-indigo-500 ring-2 ring-indigo-400': selectedSlot === 'RP'+(index+1), 'border-neutral-200 dark:border-neutral-600': selectedSlot !== 'RP'+(index+1)}" @click="selectSlot('RP'+(index+1))">
                      <button class="absolute top-1.5 right-1.5 w-6 h-6 rounded-full bg-black/50 text-white hover:bg-red-500 flex items-center justify-center text-[14px] opacity-0 group-hover:opacity-100 transition-opacity z-20 backdrop-blur-sm" @click.stop="clearSlot('RP'+(index+1))">×</button>
                      <img :src="getPlayerImage(lineup['RP'+(index+1)])" class="absolute inset-0 w-full h-full object-cover" @error="hideImage" />
                      <div class="absolute bottom-0 inset-x-0 h-[45%] bg-gradient-to-t from-black/95 via-black/50 to-transparent flex flex-col justify-end items-center pb-1.5 px-1 pointer-events-none">
                         <div class="text-[10px] sm:text-[12px] font-bold text-white w-full flex items-baseline justify-center truncate drop-shadow-md leading-tight">
                           {{ lineup['RP'+(index+1)].name }}
                           <span v-if="lineup['RP'+(index+1)].year && String(lineup['RP'+(index+1)].year) !== 'NaN' && String(lineup['RP'+(index+1)].year) !== '0' && !['TOP', '탑클래스'].includes(String(lineup['RP'+(index+1)].grade).toUpperCase())" class="text-[8px] sm:text-[9px] text-neutral-300/90 ml-0.5 tracking-tighter font-medium drop-shadow-sm">'{{ String(lineup['RP'+(index+1)].year).replace(/[\[\]]/g, '').split(',')[0].trim().slice(-2) }}</span>
                         </div>
                         <div class="text-[11px] sm:text-[14px] font-black text-amber-400 tracking-tight drop-shadow-md leading-tight mt-0.5">{{ calculatePlayerPower(lineup['RP'+(index+1)], 'RP'+(index+1)).toLocaleString() }}</div>
                      </div>
                   </div>
                  </div>
                </div>
              </div>
            </div>

            <!-- ⚾ 벤치 UI -->
            <div v-else class="w-full h-full flex flex-col justify-start gap-2 py-2">
               <div class="w-full shrink-0"><h3 class="text-xs font-bold text-neutral-500 px-2 mb-1">벤치 멤버</h3></div>
               <div class="w-full flex-1 flex flex-col gap-2 sm:gap-4 min-h-0">
                 <div class="flex-1 w-full flex justify-center items-start gap-1 sm:gap-2 min-h-0">
                    <div v-for="i in 4" :key="'BENCH'+i" @dragover.prevent @drop="onDrop($event, 'BENCH'+i)" class="flex-1 max-w-[24%] h-full flex justify-center items-start min-w-0 min-h-0">
                       <div v-if="!lineup['BENCH'+i]" class="relative h-full max-w-full aspect-[5/7] border border-dashed rounded-xl flex flex-col items-center justify-center cursor-pointer transition-all border-neutral-300 dark:border-neutral-600 bg-neutral-50/50 dark:bg-neutral-800/30 hover:bg-neutral-100 dark:hover:bg-neutral-700/50 text-neutral-400" :class="{'border-indigo-500 bg-indigo-50 dark:bg-indigo-900/30': selectedSlot === 'BENCH'+i}" @click="selectSlot('BENCH'+i)"><span class="text-[12px] font-bold">{{ 'BENCH'+i }}</span></div>
                     <div v-else draggable="true" @dragstart="onDragStart($event, 'BENCH'+i)" class="relative h-full max-w-full aspect-[5/7] border rounded-xl flex flex-col items-center p-0 cursor-pointer transition-all shadow-sm group overflow-hidden bg-white dark:bg-neutral-800" :class="{'border-indigo-500 ring-2 ring-indigo-400': selectedSlot === 'BENCH'+i, 'border-neutral-200 dark:border-neutral-600': selectedSlot === 'BENCH'+i}" @click="selectSlot('BENCH'+i)">
                        <button class="absolute top-1.5 right-1.5 w-6 h-6 rounded-full bg-black/50 text-white hover:bg-red-500 flex items-center justify-center text-[14px] opacity-0 group-hover:opacity-100 transition-opacity z-20 backdrop-blur-sm" @click.stop="clearSlot('BENCH'+i)">×</button>
                        <img :src="getPlayerImage(lineup['BENCH'+i])" class="absolute inset-0 w-full h-full object-cover object-top" @error="hideImage" />
                        <div class="absolute bottom-0 inset-x-0 h-[45%] bg-gradient-to-t from-black/95 via-black/50 to-transparent flex flex-col justify-end items-center pb-2 px-1 pointer-events-none">
                           <div class="text-[11px] sm:text-[13px] font-bold text-white w-full flex items-baseline justify-center truncate drop-shadow-md leading-tight">
                             {{ lineup['BENCH'+i].name }}
                             <span v-if="lineup['BENCH'+i].year && String(lineup['BENCH'+i].year) !== 'NaN' && String(lineup['BENCH'+i].year) !== '0' && !['TOP', '탑클래스'].includes(String(lineup['BENCH'+i].grade).toUpperCase())" class="text-neutral-300 ml-0.5 drop-shadow-sm">'{{ String(lineup['BENCH'+i].year).replace(/[\[\]]/g, '').split(',')[0].trim().slice(-2) }}</span>
                           </div>
                           <div class="text-[13px] sm:text-[15px] font-black text-neutral-300 tracking-tight drop-shadow-md leading-tight mt-0.5">{{ calculatePlayerPower(lineup['BENCH'+i], 'BENCH'+i).toLocaleString() }}</div>
                        </div>
                     </div>
                    </div>
                 </div>
                 <div class="flex-1 w-full flex justify-center items-start gap-1 sm:gap-2 min-h-0">
                    <div v-for="i in 4" :key="'BENCH'+(i+4)" @dragover.prevent @drop="onDrop($event, 'BENCH'+(i+4))" class="flex-1 max-w-[24%] h-full flex justify-center items-start min-w-0 min-h-0">
                       <div v-if="!lineup['BENCH'+(i+4)]" class="relative h-full max-w-full aspect-[5/7] border border-dashed rounded-xl flex flex-col items-center justify-center cursor-pointer transition-all border-neutral-300 dark:border-neutral-600 bg-neutral-50/50 dark:bg-neutral-800/30 hover:bg-neutral-100 dark:hover:bg-neutral-700/50 text-neutral-400" :class="{'border-indigo-500 bg-indigo-50 dark:bg-indigo-900/30': selectedSlot === 'BENCH'+(i+4)}" @click="selectSlot('BENCH'+(i+4))"><span class="text-[12px] font-bold">{{ 'BENCH'+(i+4) }}</span></div>
                     <div v-else draggable="true" @dragstart="onDragStart($event, 'BENCH'+(i+4))" class="relative h-full max-w-full aspect-[5/7] border rounded-xl flex flex-col items-center p-0 cursor-pointer transition-all shadow-sm group overflow-hidden bg-white dark:bg-neutral-800" :class="{'border-indigo-500 ring-2 ring-indigo-400': selectedSlot === 'BENCH'+(i+4), 'border-neutral-200 dark:border-neutral-600': selectedSlot !== 'BENCH'+(i+4)}" @click="selectSlot('BENCH'+(i+4))">
                        <button class="absolute top-1.5 right-1.5 w-6 h-6 rounded-full bg-black/50 text-white hover:bg-red-500 flex items-center justify-center text-[14px] opacity-0 group-hover:opacity-100 transition-opacity z-20 backdrop-blur-sm" @click.stop="clearSlot('BENCH'+(i+4))">×</button>
                        <img :src="getPlayerImage(lineup['BENCH'+(i+4)])" class="absolute inset-0 w-full h-full object-cover object-top" @error="hideImage" />
                        <div class="absolute bottom-0 inset-x-0 h-[45%] bg-gradient-to-t from-black/95 via-black/50 to-transparent flex flex-col justify-end items-center pb-2 px-1 pointer-events-none">
                           <div class="text-[11px] sm:text-[13px] font-bold text-white w-full flex items-baseline justify-center truncate drop-shadow-md leading-tight">
                             {{ lineup['BENCH'+(i+4)].name }}
                             <span v-if="lineup['BENCH'+(i+4)].year && String(lineup['BENCH'+(i+4)].year) !== 'NaN' && String(lineup['BENCH'+(i+4)].year) !== '0' && !['TOP', '탑클래스'].includes(String(lineup['BENCH'+(i+4)].grade).toUpperCase())" class="text-neutral-300 ml-0.5 drop-shadow-sm">'{{ String(lineup['BENCH'+(i+4)].year).replace(/[\[\]]/g, '').split(',')[0].trim().slice(-2) }}</span>
                           </div>
                           <div class="text-[13px] sm:text-[15px] font-black text-neutral-300 tracking-tight drop-shadow-md leading-tight mt-0.5">{{ calculatePlayerPower(lineup['BENCH'+(i+4)], 'BENCH'+(i+4)).toLocaleString() }}</div>
                        </div>
                     </div>
                    </div>
                 </div>
               </div>
            </div>
          </div>
        </section>

        <!-- ========================================== -->
        <!-- 오른쪽: 설정 탭 (폭을 살짝 늘리고, 여백을 줄여 파란박스를 거대하게!) -->
        <!-- ========================================== -->
        <section class="lg:w-[380px] xl:w-[400px] flex-shrink-0 flex flex-col rounded-2xl bg-white dark:bg-neutral-800 border border-neutral-200 dark:border-neutral-700 min-h-0 shadow-sm overflow-hidden">
          <div class="flex items-center bg-neutral-100 dark:bg-neutral-700/50 p-1 border-b border-neutral-200 dark:border-neutral-700 flex-shrink-0">
            <button @click="rightPanelTab = 'global'" :class="rightPanelTab === 'global' ? 'bg-white shadow-sm font-bold text-indigo-600' : 'text-neutral-500'" class="flex-1 py-2 text-xs rounded-lg transition-all flex items-center justify-center gap-1"><Users class="w-3 h-3"/> 공통 버프 설정</button>
            <button @click="rightPanelTab = 'player'" :class="rightPanelTab === 'player' ? 'bg-white shadow-sm font-bold text-indigo-600' : 'text-neutral-500'" class="flex-1 py-2 text-xs rounded-lg transition-all flex items-center justify-center gap-1"><UserCheck class="w-3 h-3"/> 선수 개인 설정</button>
          </div>

          <!-- 🌟 핵심 변경: p-4 lg:p-5 였던 여백을 p-1.5 sm:p-2로 대폭 깎아서 파란 박스를 화면 꽉 차게 만듭니다! -->
          <div class="flex-1 overflow-y-auto p-1.5 sm:p-2 custom-scrollbar">
            
            <!-- 글로벌 탭 -->
            <div v-if="rightPanelTab === 'global'" class="space-y-2.5 animate-in fade-in">
              
              <!-- 선호 구단 선택 -->
              <div class="bg-indigo-50 dark:bg-indigo-900/10 p-3 rounded-xl border border-indigo-100 dark:border-indigo-800 shadow-sm flex-shrink-0">
                <h3 class="text-sm font-bold text-indigo-800 dark:text-indigo-300 mb-2 flex items-center gap-1"><Shield class="w-4 h-4"/> 선호 구단(자팀) 설정</h3>
                <div class="grid grid-cols-6 gap-1.5">
                  <button v-for="group in groupedTeams" :key="'pref'+group.name" @click="globalBuffs.preferredTeam = group.id"
                      :class="globalBuffs.preferredTeam[0] === group.id[0] ? 'bg-indigo-200 dark:bg-indigo-800 border-indigo-500 shadow-md ring-2 ring-indigo-400' : 'bg-white dark:bg-neutral-700 border-neutral-200 dark:border-neutral-600 opacity-60 hover:opacity-100'"
                      class="p-1 flex items-center justify-center rounded-lg border transition-all">
                    <img v-if="getTeamLogoUrl(group.id[0])" :src="getTeamLogoUrl(group.id[0])" class="w-8 h-8 object-contain" />
                  </button>
                </div>
              </div>

              <!-- 시너지 마스터리 설정 -->
              <div class="bg-indigo-50 dark:bg-indigo-900/10 p-3 rounded-xl border border-indigo-100 dark:border-indigo-800 flex-shrink-0">
                <h3 class="text-sm font-bold text-indigo-800 dark:text-indigo-300 mb-2 flex items-center gap-1"><Sparkles class="w-4 h-4"/> 시너지 마스터리 설정</h3>
                <datalist id="synergy-list">
                  <option v-for="s in synergyOptions" :key="s" :value="s" />
                </datalist>
                <div class="space-y-1.5">
                  <div v-for="i in 5" :key="i" class="flex items-center gap-2">
                    <span class="text-[10px] font-bold text-neutral-500 w-6">칸 {{ i }}</span>
                    <input list="synergy-list" v-model="globalBuffsAll[activeDeck].synergyMasteries[i-1]" placeholder="시너지 검색..." class="w-full px-2 py-1.5 bg-white border rounded text-xs"/>
                    <label class="flex items-center justify-center cursor-pointer bg-white px-2 py-1.5 rounded border transition-colors shrink-0 select-none"
                           :class="globalBuffsAll[activeDeck].amplifiedMasteryIndex === i - 1 ? 'border-indigo-500 bg-indigo-100 dark:bg-indigo-900/50 shadow-inner' : 'border-neutral-200 hover:bg-neutral-50'">
                      <input type="checkbox" :checked="globalBuffsAll[activeDeck].amplifiedMasteryIndex === i - 1" @change="globalBuffsAll[activeDeck].amplifiedMasteryIndex = $event.target.checked ? i - 1 : -1" class="hidden" />
                      <span class="text-[10px] font-black tracking-tight" :class="globalBuffsAll[activeDeck].amplifiedMasteryIndex === i - 1 ? 'text-indigo-700 dark:text-indigo-300' : 'text-neutral-400'">증폭</span>
                    </label>
                  </div>
                  <div class="text-[9px] text-indigo-600 font-bold mt-1">※ '증폭' 버튼을 누르면 해당 시너지가 증폭됩니다. (최대 1개 지정, 다시 누르면 해제)</div>
                </div>
              </div>
              
              <!-- 공통 버프 및 바인더 설정 -->
              <div class="bg-indigo-50 dark:bg-indigo-900/10 p-3 rounded-xl border border-indigo-100 dark:border-indigo-800 flex-shrink-0">
                <h3 class="text-sm font-bold text-indigo-800 dark:text-indigo-300 mb-2 flex items-center gap-1"><Users class="w-4 h-4"/> 공통 버프 및 바인더 설정</h3>
                
                <div class="grid grid-cols-3 gap-2 mb-3">
                  <div class="flex flex-col gap-1"><label class="text-[10px] font-bold text-neutral-500">팀 레벨 (1~100)</label><input type="number" min="1" max="100" v-model.number="globalBuffsAll[activeDeck].teamLevel" class="w-full px-2 py-1 text-center bg-white border rounded text-xs"/></div>
                  <div class="flex flex-col gap-1"><label class="text-[10px] font-bold text-neutral-500">클랜 레벨 파워</label><input type="number" v-model.number="globalBuffsAll[activeDeck].clanBuff" class="w-full px-2 py-1 text-center bg-white border rounded text-xs"/></div>
                  <div class="flex flex-col gap-1"><label class="text-[10px] font-bold text-indigo-600">바인더 레벨</label><input type="number" min="1" max="100" v-model.number="globalBuffsAll[activeDeck].binderLevel" class="w-full px-2 py-1 text-center bg-indigo-100 border border-indigo-300 rounded text-xs font-black text-indigo-800 outline-none"/></div>
                </div>
                
                <!-- 바인더 세부 설정 -->
                <div class="border border-indigo-200 bg-white dark:bg-neutral-800 rounded-lg p-1.5 shadow-sm">
                  <div class="text-[11px] font-bold text-indigo-700 dark:text-indigo-300 mb-1.5 text-center bg-indigo-50 dark:bg-indigo-900/30 py-1 rounded">바인더 세부 설정 (5x5 매트릭스)</div>
                  
                  <datalist id="binder-team-list"><option v-for="t in binderTeamOptions" :key="t" :value="t"></option></datalist>
                  <datalist id="binder-player-list"><option v-for="p in binderPlayerOptions" :key="p" :value="p"></option></datalist>
                  <datalist id="binder-year-list"><option v-for="y in binderYearOptions" :key="y" :value="y"></option></datalist>
                  <datalist id="binder-grade-list"><option v-for="g in binderGradeOptions" :key="g" :value="g"></option></datalist>

                  <div class="grid grid-cols-6 gap-0.5 mb-1 items-center text-center">
                    <div></div>
                    <div class="text-[9px] font-bold text-neutral-500 bg-neutral-100 dark:bg-neutral-700 rounded py-0.5">팀</div>
                    <div class="text-[9px] font-bold text-neutral-500 bg-neutral-100 dark:bg-neutral-700 rounded py-0.5">포지션</div>
                    <div class="text-[9px] font-bold text-neutral-500 bg-neutral-100 dark:bg-neutral-700 rounded py-0.5">인물</div>
                    <div class="text-[9px] font-bold text-neutral-500 bg-neutral-100 dark:bg-neutral-700 rounded py-0.5">연도</div>
                    <div class="text-[9px] font-bold text-neutral-500 bg-neutral-100 dark:bg-neutral-700 rounded py-0.5">등급</div>
                  </div>
                  
                  <template v-for="(row, idx) in globalBuffs.binderMatrix" :key="'bm'+idx">
                    <div class="grid grid-cols-6 gap-0.5 mb-1 items-center">
                      <div class="text-[9px] font-black text-indigo-400 text-center">{{ idx + 1 }}번</div>
                      <input type="text" list="binder-team-list" v-model="row.team" placeholder="구단" class="w-full text-center text-[10px] border rounded py-1 bg-neutral-50 outline-none focus:border-indigo-500 focus:bg-white" />
                      <select v-model="row.position" class="w-full text-center text-[10px] border rounded py-1 bg-neutral-50 outline-none focus:border-indigo-500 focus:bg-white">
                         <option value="">비워둠</option><option value="선발">선발</option><option value="계투">계투</option><option value="내야">내야</option><option value="외야 & 지명">외야&지명</option>
                      </select>
                      <input type="text" list="binder-player-list" v-model="row.player" placeholder="이름" class="w-full text-center text-[10px] border rounded py-1 bg-neutral-50 outline-none focus:border-indigo-500 focus:bg-white" />
                      <input type="text" list="binder-year-list" v-model="row.year" placeholder="연도" class="w-full text-center text-[10px] border rounded py-1 bg-neutral-50 outline-none focus:border-indigo-500 focus:bg-white" />
                      <input type="text" list="binder-grade-list" v-model="row.grade" placeholder="등급" class="w-full text-center text-[10px] border rounded py-1 bg-neutral-50 outline-none focus:border-indigo-500 focus:bg-white" />
                    </div>
                  </template>
                </div>
              </div>
              
              <!-- 자동 계산된 팀플/디그니티 합계 -->
              <div class="bg-indigo-50 dark:bg-indigo-900/10 p-3 rounded-xl border border-indigo-100 dark:border-indigo-800 shadow-sm flex-shrink-0">
                <h3 class="text-[11px] font-bold text-indigo-800 dark:text-indigo-300 mb-2 flex items-center gap-1"><Zap class="w-3 h-3"/> 팀플/디그니티 버프 자동 적용</h3>
                <div class="grid grid-cols-2 gap-2">
                  <template v-for="teamId in Array.from(new Set(Object.values(lineup).filter(Boolean).flatMap(p => toArray(p.team).map(toLowerCase))))" :key="teamId">
                     <div v-if="calculateTeamPlayerDignityBuff({ team: teamId }, activeDeck) > 0" class="flex justify-between items-center text-[10px] bg-white dark:bg-neutral-800 px-2 py-1.5 rounded border shadow-sm flex-shrink-0">
                       <span class="font-bold text-neutral-700">{{ findTeamName(teamId) }}</span>
                       <span class="text-indigo-600 font-black">+{{ calculateTeamPlayerDignityBuff({ team: teamId }, activeDeck) }}</span>
                     </div>
                  </template>
                </div>
              </div>
              
              <!-- 감독 카드 설정 -->
              <div class="bg-indigo-50 dark:bg-indigo-900/10 p-3 rounded-xl border border-indigo-100 dark:border-indigo-800 shadow-sm flex-shrink-0">
                 <h3 class="text-sm font-bold text-indigo-800 dark:text-indigo-300 mb-2 flex items-center gap-1"><UserCheck class="w-4 h-4"/> 감독 카드 설정</h3>
                 <div class="grid grid-cols-2 gap-2">
                   <div class="flex flex-col gap-1">
                     <label class="text-[10px] font-bold text-neutral-500">감독 유형</label>
                     <select v-model="globalBuffsAll[activeDeck].managerType" class="w-full px-2 py-1.5 bg-white border rounded text-xs font-medium">
                       <option value="">미장착</option>
                       <option value="my_1st">자팀 1st ({{ STAT_LABELS[MANAGER_TYPES['1st'].main] }} / {{ STAT_LABELS[MANAGER_TYPES['1st'].sub] }})</option>
                       <option value="com_1st">공통 1st ({{ STAT_LABELS[MANAGER_TYPES['1st'].main] }} / {{ STAT_LABELS[MANAGER_TYPES['1st'].sub] }})</option>
                       <option value="my_2nd">자팀 2nd ({{ STAT_LABELS[MANAGER_TYPES['2nd'].main] }} / {{ STAT_LABELS[MANAGER_TYPES['2nd'].sub] }})</option>
                       <option value="com_2nd">공통 2nd ({{ STAT_LABELS[MANAGER_TYPES['2nd'].main] }} / {{ STAT_LABELS[MANAGER_TYPES['2nd'].sub] }})</option>
                       <option value="my_3rd">자팀 3rd ({{ STAT_LABELS[MANAGER_TYPES['3rd'].main] }} / {{ STAT_LABELS[MANAGER_TYPES['3rd'].sub] }})</option>
                       <option value="com_3rd">공통 3rd ({{ STAT_LABELS[MANAGER_TYPES['3rd'].main] }} / {{ STAT_LABELS[MANAGER_TYPES['3rd'].sub] }})</option>
                       <option value="my_4th">자팀 4th ({{ STAT_LABELS[MANAGER_TYPES['4th'].main] }} / {{ STAT_LABELS[MANAGER_TYPES['4th'].sub] }})</option>
                       <option value="com_4th">공통 4th ({{ STAT_LABELS[MANAGER_TYPES['4th'].main] }} / {{ STAT_LABELS[MANAGER_TYPES['4th'].sub] }})</option>
                     </select>
                   </div>
                   <div class="flex flex-col gap-1">
                     <label class="text-[10px] font-bold text-neutral-500">강화 레벨 (0~15)</label>
                     <input type="number" min="0" max="15" v-model.number="globalBuffsAll[activeDeck].managerEnhance" class="w-full px-2 py-1.5 text-center bg-white border rounded text-xs"/>
                   </div>
                 </div>
              </div>
              
              <!-- 🌟 감독 전술 지시 (스킬트리) -->
              <div class="bg-indigo-50 dark:bg-indigo-900/10 p-3 rounded-xl border border-indigo-100 dark:border-indigo-800 shadow-sm flex-shrink-0 mt-3">
                <div class="flex justify-between items-center mb-2">
                  <h3 class="text-sm font-bold text-indigo-800 dark:text-indigo-300 flex items-center gap-1"><TrendingUp class="w-4 h-4"/> 전술 지시 스킬트리</h3>
                  <div class="text-[10px] sm:text-xs font-bold bg-white px-2 py-1 rounded shadow-sm border transition-colors" :class="remainingTacticPt < 0 ? 'text-red-500 border-red-300 bg-red-50' : 'text-indigo-600 border-indigo-200'">
                     PT: {{ usedTacticPt }} / {{ totalTacticPt }} (잔여: {{ remainingTacticPt }})
                  </div>
                </div>

                <div class="grid grid-cols-2 gap-2 mb-3">
                  <div class="flex flex-col gap-1">
                    <label class="text-[10px] font-bold text-neutral-500">감독 돌파 단계 (0~6)</label>
                    <input type="number" min="0" max="6" v-model.number="globalBuffsAll[activeDeck].managerBreakthrough" class="w-full px-2 py-1.5 text-center bg-white border rounded text-xs"/>
                  </div>
                  <!-- 감독 강화는 기존 위에 있는 managerEnhance 변수 사용 -->
                  <div class="flex flex-col gap-1 opacity-70">
                    <label class="text-[10px] font-bold text-neutral-500">감독 강화 단계 (0~15)</label>
                    <div class="w-full px-2 py-1.5 text-center bg-neutral-100 border rounded text-xs text-neutral-500 font-bold tracking-tight">※위탁 연동 ({{globalBuffsAll[activeDeck].managerEnhance}}강)</div>
                  </div>
                </div>

                <!-- 궁극의 커스텀 UI: 모달 대신 아코디언으로 17개 옵션 전부 조절 가능하게 변경 -->
                <div class="flex flex-col mb-2 bg-white rounded border border-indigo-100 shadow-sm overflow-hidden">
                   <details class="group">
                     <summary class="text-[11px] font-bold text-indigo-700 p-2 cursor-pointer bg-indigo-50/50 flex justify-between items-center outline-none">
                       <span>🎯 궁극의 커스텀: 조건부 발생 확률 상세 조절</span>
                       <ChevronRightIcon class="w-3 h-3 transition-transform group-open:rotate-90"/>
                     </summary>
                     <div class="p-2 flex flex-col gap-2 max-h-[250px] overflow-y-auto custom-scrollbar border-t border-indigo-100 bg-neutral-50/50">
                       <div class="text-[9px] text-neutral-500 font-bold mb-1 leading-tight border-b pb-1">
                         ※ 기댓값을 위해 엑셀 원본의 발생 확률×잔여이닝(%)이 기본값으로 세팅되어 있습니다.<br>
                         ※ 개인 덱 상황에 맞춰 자유롭게 세부 확률을 조절해 보세요.
                       </div>
                       
                       <div class="flex flex-col gap-1.5">
                         <div class="text-[10px] font-black text-indigo-900 border-b border-indigo-100 pb-0.5">■ 상시 효과 (상황 연동)</div>
                         <div class="flex justify-between items-center text-[10px]">
                           <span class="text-neutral-700">8번. 득점권 상황 도래 확률</span>
                           <div class="flex items-center gap-1"><input type="number" step="0.1" v-model.number="globalBuffsAll[activeDeck].tacticBaseRates.scoring" class="w-12 text-center border rounded p-0.5 outline-none font-bold text-indigo-600">%</div>
                         </div>
                         <div class="flex justify-between items-center text-[10px]">
                           <span class="text-neutral-700">12번. 클린업 타선 상대 확률</span>
                           <div class="flex items-center gap-1"><input type="number" step="0.1" v-model.number="globalBuffsAll[activeDeck].tacticBaseRates.cleanup" class="w-12 text-center border rounded p-0.5 outline-none font-bold text-indigo-600">%</div>
                         </div>
                       </div>
                       
                       <div class="flex flex-col gap-1.5 mt-2">
                         <div class="text-[10px] font-black text-amber-600 border-b border-amber-100 pb-0.5">■ 조건 달성 효과</div>
                         <div v-for="(tac, i) in TACTICS_INFO" :key="'prob'+i" class="flex justify-between items-center text-[10px]">
                           <span class="text-neutral-700 truncate pr-2 flex-1" :title="tac.descCond(tac.condVals[1])">{{ i+1 }}번. {{ tac.name }} 성공</span>
                           <div class="flex items-center gap-1 shrink-0"><input type="number" step="0.1" v-model.number="globalBuffsAll[activeDeck].tacticCondRates[i]" class="w-12 text-center border rounded p-0.5 outline-none font-bold text-amber-600">%</div>
                         </div>
                       </div>
                     </div>
                   </details>
                </div>

                <!-- 전술 목록 (스크롤) -->
                <div class="flex flex-col gap-2 max-h-[300px] overflow-y-auto custom-scrollbar pr-1">
                   <div v-for="(tac, i) in TACTICS_INFO" :key="'tac'+i" class="bg-white border rounded-lg p-2 shadow-sm transition-all" :class="globalBuffsAll[activeDeck].tacticLevels[i] > 0 ? 'border-indigo-300 bg-indigo-50/20' : 'border-neutral-200'">
                      <div class="flex justify-between items-center mb-1">
                         <div class="flex items-center gap-1.5">
                           <span class="text-[9px] px-1.5 py-0.5 bg-neutral-100 rounded font-black border" :class="tac.type==='타자'?'text-orange-600 border-orange-200':tac.type==='투수'?'text-blue-600 border-blue-200':'text-emerald-600 border-emerald-200'">{{ tac.type }}</span>
                           <span class="text-[11px] sm:text-xs font-bold text-neutral-800">{{ tac.name }}</span>
                         </div>
                         <select v-model.number="globalBuffsAll[activeDeck].tacticLevels[i]" class="text-[11px] sm:text-xs border rounded p-1 font-bold outline-none" :class="[globalBuffsAll[activeDeck].managerEnhance < tac.req[globalBuffsAll[activeDeck].tacticLevels[i]] ? 'text-red-500 border-red-300' : 'text-indigo-700 border-indigo-200 bg-indigo-50', globalBuffsAll[activeDeck].tacticLevels[i] === 0 ? 'text-neutral-500 bg-white border-neutral-200' : '']">
                            <option :value="0">Lv.0</option>
                            <option :value="1" :disabled="globalBuffsAll[activeDeck].managerEnhance < tac.req[1]">Lv.1 ({{ tac.pt[1] }}pt<template v-if="globalBuffsAll[activeDeck].managerEnhance < tac.req[1]"> / 🔒{{tac.req[1]}}강 필요</template>)</option>
                            <option :value="2" :disabled="globalBuffsAll[activeDeck].managerEnhance < tac.req[2]">Lv.2 ({{ tac.pt[2] }}pt<template v-if="globalBuffsAll[activeDeck].managerEnhance < tac.req[2]"> / 🔒{{tac.req[2]}}강 필요</template>)</option>
                            <option :value="3" :disabled="globalBuffsAll[activeDeck].managerEnhance < tac.req[3]">Lv.3 ({{ tac.pt[3] }}pt<template v-if="globalBuffsAll[activeDeck].managerEnhance < tac.req[3]"> / 🔒{{tac.req[3]}}강 필요</template>)</option>
                            <option :value="4" :disabled="globalBuffsAll[activeDeck].managerEnhance < tac.req[4]">Lv.4 ({{ tac.pt[4] }}pt<template v-if="globalBuffsAll[activeDeck].managerEnhance < tac.req[4]"> / 🔒{{tac.req[4]}}강 필요</template>)</option>
                            <option :value="5" :disabled="globalBuffsAll[activeDeck].managerEnhance < tac.req[5]">Lv.5 ({{ tac.pt[5] }}pt<template v-if="globalBuffsAll[activeDeck].managerEnhance < tac.req[5]"> / 🔒{{tac.req[5]}}강 필요</template>)</option>
                         </select>
                      </div>
                      
                      <!-- 스탯 설명 박스 -->
                      <div class="flex flex-col gap-0.5 p-1.5 rounded border transition-colors" :class="globalBuffsAll[activeDeck].tacticLevels[i] > 0 ? 'bg-indigo-50 border-indigo-100' : 'bg-neutral-50 border-neutral-100'">
                         <div class="text-[10px] font-bold flex items-center justify-between" :class="globalBuffsAll[activeDeck].tacticLevels[i] > 0 ? 'text-indigo-700' : 'text-neutral-500'">
                            <span>[상시] {{ tac.descBase(tac.baseVals[globalBuffsAll[activeDeck].tacticLevels[i]]) }}</span>
                         </div>
                         <div class="text-[10px] font-medium flex items-center justify-between" :class="globalBuffsAll[activeDeck].tacticLevels[i] > 0 ? 'text-indigo-900/70' : 'text-neutral-400'">
                            <span>[조건] {{ tac.descCond(tac.condVals[globalBuffsAll[activeDeck].tacticLevels[i]]) }}</span>
                         </div>
                      </div>
                      
                      <!-- 경고 메시지 -->
                      <div v-if="globalBuffsAll[activeDeck].managerEnhance < tac.req[globalBuffsAll[activeDeck].tacticLevels[i]]" class="text-[9px] text-red-500 font-bold mt-1 bg-red-50 px-1.5 py-0.5 rounded border border-red-100">
                         ⚠️ 강화 부족: 인게임 발동 불가 (최소 {{ tac.req[globalBuffsAll[activeDeck].tacticLevels[i]] }}강 필요)
                      </div>
                   </div>
                </div>
              </div>
              
              <!-- 활성화된 팀 시너지 영역 -->
              <div class="bg-indigo-50 dark:bg-indigo-900/10 p-3 rounded-xl border border-indigo-100 dark:border-indigo-800 shadow-sm flex-shrink-0">
                 <div class="flex flex-col mb-2">
                   <h3 class="text-sm font-bold text-indigo-800 dark:text-indigo-300 flex items-center gap-1 mb-2">
                      <Users class="w-4 h-4"/> 현재 활성화된 팀 시너지 ({{ filteredActiveTeamSynergies.length }}개)
                   </h3>
                   <div class="flex bg-white p-1 rounded-lg border border-indigo-100 flex-shrink-0">
                     <button v-for="cat in ['전체', '기본', '출신', '기록', '인물']" :key="cat" @click="activeSynergyCategory = cat" :class="activeSynergyCategory === cat ? 'bg-indigo-100 font-bold text-indigo-700 shadow-sm' : 'text-neutral-500 hover:bg-neutral-50'" class="flex-1 py-1 text-[10px] sm:text-xs rounded-md transition-all">{{ cat }}</button>
                   </div>
                 </div>

                 <div class="flex flex-col gap-1.5 max-h-48 overflow-y-auto custom-scrollbar pr-1">
                   <div v-for="syn in filteredActiveTeamSynergies" :key="syn.name" class="flex flex-col text-xs bg-white rounded-lg border border-indigo-100 shadow-sm overflow-hidden flex-shrink-0">
                     <div class="flex justify-between items-center px-2 py-1.5 cursor-pointer hover:bg-neutral-50 transition-colors" @click="expandedSynergy = expandedSynergy === syn.name ? null : syn.name">
                       <span class="font-bold text-neutral-700 text-[11px]">{{ syn.name }}</span>
                       <div class="flex items-center gap-1 flex-shrink-0 whitespace-nowrap"><span class="text-indigo-600 font-bold text-[10px]">{{ formatBonuses(syn.bonuses) }}</span><ChevronRightIcon class="w-3 h-3 text-neutral-400" /></div>
                     </div>
                     <div v-if="expandedSynergy === syn.name" class="px-2 py-1.5 bg-neutral-50 border-t">
                        <div class="text-[9px] text-neutral-500 mb-1">적용 선수 ({{ syn.matchedPlayers.length }}명):</div>
                        <div class="flex flex-wrap gap-1"><span v-for="pName in syn.matchedPlayers" :key="pName" class="text-[9px] bg-indigo-100 px-1 rounded">{{ pName }}</span></div>
                     </div>
                   </div>
                   <div v-if="filteredActiveTeamSynergies.length === 0" class="text-[10px] text-neutral-400 text-center py-2">해당 분류에 활성화된 시너지가 없습니다.</div>
                 </div>

                 <div v-if="filteredPendingTeamSynergies.length > 0" class="mt-3 pt-2 border-t border-indigo-200 flex-shrink-0">
                   <h3 class="text-[11px] font-bold text-neutral-500 mb-1.5 flex items-center gap-1">
                      <Users class="w-3 h-3"/> 발동 대기 중인 시너지 ({{ filteredPendingTeamSynergies.length }}개)
                   </h3>
                   <div class="flex flex-col gap-1 max-h-48 overflow-y-auto custom-scrollbar pr-1">
                     <div v-for="syn in filteredPendingTeamSynergies" :key="'pend'+syn.name" class="flex flex-col text-[10px] bg-neutral-100 rounded border shadow-sm overflow-hidden flex-shrink-0">
                       <div class="flex justify-between items-center px-2 py-1.5 cursor-pointer hover:bg-neutral-200 transition-colors" @click="expandedPendingSynergy = expandedPendingSynergy === syn.name ? null : syn.name">
                         <span class="font-medium text-neutral-600">{{ syn.name }}</span>
                         <div class="flex items-center gap-1 flex-shrink-0 whitespace-nowrap">
                           <span class="text-red-500 font-bold bg-red-50 px-1 py-0.5 rounded">{{ syn.current }}/{{ syn.required }}명</span>
                           <ChevronRightIcon class="w-3 h-3 text-neutral-400" />
                         </div>
                       </div>
                       <div v-if="expandedPendingSynergy === syn.name" class="px-2 py-1.5 bg-neutral-50 border-t">
                          <div class="text-[9px] text-neutral-500 mb-1">현재 보유 선수:</div>
                          <div class="flex flex-wrap gap-1"><span v-for="pName in syn.matchedPlayers" :key="pName" class="text-[9px] bg-red-100 text-red-700 px-1 rounded">{{ pName }}</span></div>
                       </div>
                     </div>
                   </div>
                 </div>
              </div>
            </div>

            <!-- 플레이어 탭 -->
            <div v-else-if="selectedSlot && lineup[selectedSlot] && playerBuffs[selectedSlot]" class="space-y-2.5 animate-in fade-in flex flex-col h-full">
              <!-- 🌟 1. 선수 프로필 영역 (글자 극대화) -->
              <div class="flex items-center gap-3 p-2 bg-neutral-100 dark:bg-neutral-700/50 rounded-xl flex-shrink-0 border border-neutral-200">
                <img :src="getGradeImage(lineup[selectedSlot].grade)" class="w-14 h-14 object-contain drop-shadow" @error="hideImage"/>
                <div>
                  <div class="font-bold text-lg text-neutral-900 dark:text-neutral-100 leading-tight">{{ lineup[selectedSlot].name }}</div>
                  <div class="text-[13px] text-neutral-500 mt-0.5">{{ selectedSlot }} 슬롯 배치됨</div>
                </div>
                <div class="ml-auto text-right">
                  <div class="text-xs font-bold text-indigo-500">개별 총 파워</div>
                  <div class="text-3xl font-black tabular-nums text-indigo-600 dark:text-indigo-400 tracking-tighter">{{ calculatePlayerPower(lineup[selectedSlot], selectedSlot).toLocaleString() }}</div>
                </div>
              </div>

              <!-- 🌟 2. 전체 스크롤 영역 (여백 깎고 글자는 키움) -->
              <div class="flex-1 overflow-y-auto custom-scrollbar pr-1.5 pb-2 space-y-2.5">
                
                <!-- 🌟 최종 세부 능력치 (오각형 레이더 차트 적용) -->
                <div v-if="computedPlayerStats[selectedSlot]" class="bg-neutral-50 dark:bg-[#222222] p-4 rounded-xl border border-neutral-200 dark:border-neutral-700/80 shadow-sm flex-shrink-0 flex flex-col items-center relative overflow-hidden">
                  
                  <!-- 상단 종합 능력 타이틀 -->
                  <div class="w-full flex justify-between items-center mb-6 z-10 px-1">
                    <span class="text-[15px] font-bold text-neutral-800 dark:text-neutral-100">종합 능력</span>
                    <span class="text-2xl font-black text-blue-600 dark:text-blue-400 tracking-tighter">{{ calculatePlayerPower(lineup[selectedSlot], selectedSlot).toLocaleString() }}</span>
                  </div>

                  <!-- 🌟 레이더 차트 SVG -->
                  <div class="relative w-full max-w-[280px] aspect-square flex items-center justify-center mb-4 z-10 mx-auto">
                     <svg viewBox="0 0 200 200" class="w-full h-full overflow-visible">
                        <!-- 차트를 화면 중앙으로 맞추기 위한 위치 보정 -->
                        <g transform="translate(0, -5)">
                           <!-- 배경 거미줄 (4단계: 500, 1000, 1500, 2000) -->
                           <polygon v-for="level in 4" :key="'web'+level" :points="getRadarWebPoints(level)" fill="none" stroke="currentColor" class="text-neutral-300 dark:text-neutral-600/70" stroke-width="0.7" />
                           <!-- 중심선 -->
                           <line v-for="angle in [0, 72, 144, 216, 288]" :key="'line'+angle" x1="100" y1="100" :x2="100 + 60 * Math.cos((angle - 90) * Math.PI / 180)" :y2="100 + 60 * Math.sin((angle - 90) * Math.PI / 180)" stroke="currentColor" class="text-neutral-300 dark:text-neutral-600/70" stroke-width="0.7" />
                           
                           <!-- 실제 능력치 다각형 (파란색) -->
                           <polygon :points="getRadarStatPoints(selectedSlot)" fill="rgba(59, 130, 246, 0.4)" stroke="#3b82f6" stroke-width="2" stroke-linejoin="round" />
                           
                           <!-- 꼭짓점 점 (파란색) -->
                           <circle v-for="(pt, i) in getRadarStatDots(selectedSlot)" :key="'dot'+i" :cx="pt.x" :cy="pt.y" r="3.5" fill="#3b82f6" />
                           
                           <!-- 레이더 차트 스탯 텍스트 -->
                           <text v-for="(lbl, i) in getRadarLabels(selectedSlot)" :key="'lbl'+i" :x="lbl.x" :y="lbl.y" class="fill-neutral-700 dark:fill-neutral-200" font-size="10" font-weight="bold" text-anchor="middle" dominant-baseline="middle">{{ lbl.text }}</text>
                        </g>
                     </svg>
                  </div>

                  <!-- 🌟 하단 스탯 박스 그리드 (인게임 고증: 레이더 차트 배열 순서 적용) -->
                  <div class="w-full flex flex-col gap-2.5 z-10">
                     
                     <!-- 코어 5대 스탯 (상단 4개, 하단 1개 배치) -->
                     <div class="grid grid-cols-2 gap-2">
                        <div v-for="stat in (isPitcher(lineup[selectedSlot]) ? radarPitcherStats.slice(0,4) : radarBatterStats.slice(0,4))" :key="stat" 
                             class="flex justify-between items-center bg-white dark:bg-[#333333] rounded-lg px-3 py-2.5 border border-neutral-200 dark:border-neutral-700 shadow-sm">
                           <span class="text-xs font-bold text-neutral-500 dark:text-neutral-400">{{ STAT_LABELS[stat] || stat }}</span>
                           <span class="text-sm font-black text-neutral-800 dark:text-white">{{ computedPlayerStats[selectedSlot].stats[stat] }}</span>
                        </div>
                     </div>
                     <div class="flex justify-between items-center bg-white dark:bg-[#333333] rounded-lg px-3 py-2.5 border border-neutral-200 dark:border-neutral-700 shadow-sm w-[calc(50%-4px)]">
                        <span class="text-xs font-bold text-neutral-500 dark:text-neutral-400">{{ STAT_LABELS[(isPitcher(lineup[selectedSlot]) ? radarPitcherStats : radarBatterStats)[4]] }}</span>
                        <span class="text-sm font-black text-neutral-800 dark:text-white">{{ computedPlayerStats[selectedSlot].stats[(isPitcher(lineup[selectedSlot]) ? radarPitcherStats : radarBatterStats)[4]] }}</span>
                     </div>
                     
                     <!-- 하단 논코어 스탯 (수비/주루 등, 3개 나란히 배치) -->
                     <div class="grid grid-cols-3 gap-2 mt-1">
                        <div v-for="stat in (isPitcher(lineup[selectedSlot]) ? pitcherStats.slice(5) : batterStats.slice(5))" :key="'noncore'+stat" 
                             class="flex flex-col items-center justify-center bg-white dark:bg-[#333333] rounded-lg py-2 border border-neutral-200 dark:border-neutral-700 shadow-sm">
                           <span class="text-[11px] font-bold text-neutral-500 dark:text-neutral-400 mb-0.5">{{ STAT_LABELS[stat] || stat }}</span>
                           <!-- 수비 스탯은 노란색으로 특별 강조! -->
                           <span class="text-sm font-black text-neutral-800 dark:text-white" :class="stat === 'defense' ? 'text-amber-500 dark:text-amber-400' : ''">{{ computedPlayerStats[selectedSlot].stats[stat] }}</span>
                        </div>
                     </div>
                  </div>
                </div>

                <!-- 타순 설정 -->
                <div v-if="!isPitcher(lineup[selectedSlot])" class="bg-orange-50 p-2 rounded-xl border border-orange-100 flex-shrink-0">
                  <h3 class="text-sm font-bold text-orange-800 mb-1.5">타순 설정</h3>
                  <select v-model.number="playerBuffs[selectedSlot].battingOrder" class="w-full py-1.5 px-2 rounded-lg border border-orange-200 bg-white text-sm outline-none font-medium">
                    <option :value="null">타순 미지정</option>
                    <option v-for="i in 9" :key="i" :value="i">{{ i }}번 타자</option>
                  </select>
                </div>

                <!-- 카드 강화 -->
                <div class="bg-emerald-50 p-2 rounded-xl border border-emerald-100 shadow-sm flex-shrink-0">
                  <h3 class="text-sm font-bold text-emerald-800 flex items-center gap-1 mb-1.5"><ArrowUpCircle class="w-4 h-4"/> 카드 강화</h3>
                  <div class="flex flex-wrap gap-1">
                    <button v-for="lvl in (getMaxEnhance(lineup[selectedSlot]) + 1)" :key="'enh'+lvl"
                      @click="playerBuffs[selectedSlot].enhancementLevel = lvl-1"
                      :class="playerBuffs[selectedSlot].enhancementLevel === lvl-1 ? 'bg-emerald-600 text-white border-emerald-600' : 'bg-white text-neutral-600 border-neutral-300'"
                      class="w-9 h-8 flex items-center justify-center text-xs font-bold border rounded flex-shrink-0">+{{ lvl-1 }}</button>
                  </div>
                </div>

                <!-- 한계 돌파 -->
                <div v-if="getMaxBreakthrough(lineup[selectedSlot]) > 0" class="bg-fuchsia-50 p-2 rounded-xl border border-fuchsia-100 shadow-sm flex-shrink-0">
                  <h3 class="text-sm font-bold text-fuchsia-800 flex items-center gap-1 mb-1.5"><Sparkles class="w-4 h-4"/> 한계 돌파</h3>
                  <div class="flex flex-wrap gap-1">
                    <button v-for="lvl in (getMaxBreakthrough(lineup[selectedSlot]) + 1)" :key="'brk'+lvl"
                      @click="playerBuffs[selectedSlot].breakthroughLevel = lvl-1"
                      :class="playerBuffs[selectedSlot].breakthroughLevel === lvl-1 ? 'bg-fuchsia-600 text-white border-fuchsia-600' : 'bg-white text-neutral-600 border-neutral-300'"
                      class="px-3 h-8 flex items-center justify-center text-xs font-bold border rounded flex-shrink-0">{{ lvl-1 === 0 ? '돌파 안함' : (lvl-1) + '돌' }}</button>
                  </div>
                </div>

                <!-- 선수 개인 성장 및 커리어 장착 -->
                <div class="bg-sky-50 p-2 rounded-xl border border-sky-100 flex-shrink-0">
                  <h3 class="text-[15px] font-bold text-sky-800 mb-2 flex items-center gap-1"><Zap class="w-4 h-4"/> 선수 기본 성장</h3>
                  <div class="grid grid-cols-2 gap-2 mb-3">
                    <div class="flex flex-col gap-0.5"><label class="text-xs font-bold text-neutral-500">선수 레벨</label><input type="number" v-model.number="playerBuffs[selectedSlot].playerLevel" class="w-full px-2 py-1.5 text-center bg-white border rounded text-sm font-semibold"/></div>
                    <div class="flex flex-col gap-0.5"><label class="text-xs font-bold text-neutral-500">도감 파워</label><input type="number" v-model.number="playerBuffs[selectedSlot].collectionBuff" class="w-full px-2 py-1.5 text-center bg-white border rounded text-sm font-semibold"/></div>
                    <div class="flex flex-col gap-0.5"><label class="text-xs font-bold text-neutral-500">커리어 레벨 파워</label><input type="number" v-model.number="playerBuffs[selectedSlot].careerLevelBuff" class="w-full px-2 py-1.5 text-center bg-white border rounded text-sm font-semibold"/></div>
                    <div class="flex flex-col gap-0.5">
                      <label class="text-xs font-bold text-indigo-500 flex items-center justify-center gap-1">바인더 파워 <Zap class="w-3 h-3"/></label>
                      <div class="w-full px-2 py-1.5 text-center bg-indigo-50 border border-indigo-200 rounded text-sm font-black text-indigo-700 shadow-inner">+{{ getPlayerBinderPower(lineup[selectedSlot], activeDeck) }}</div>
                    </div>
                  </div>

                  <!-- 커리어 6칸 세팅 UI -->
                  <div class="bg-white p-2 rounded-xl border border-neutral-200 shadow-sm">
                    <div class="flex justify-between items-center mb-1.5">
                      <h3 class="text-sm font-bold text-neutral-700 flex items-center gap-1">🎖️ 커리어 장착 (총 6칸)</h3>
                      <button @click="showCareerManager = true" class="text-xs bg-emerald-100 text-emerald-700 px-2.5 py-1 rounded hover:bg-emerald-200 transition-colors font-bold">커리어 편집</button>
                    </div>
                    <div class="grid grid-cols-6 gap-1">
                      <div v-for="(c, i) in (playerBuffs[selectedSlot]?.careers || [])" :key="i" 
                           class="border rounded flex flex-col items-center justify-center p-1 h-14 transition-colors"
                           :class="c.statType ? 'bg-emerald-50 border-emerald-300' : 'bg-neutral-50 border-dashed'">
                        <span class="text-[10px] font-black" :class="getCareerGradeColor(c.grade)">{{ c.grade }}</span>
                        <span class="text-xs font-bold text-neutral-800 mt-0.5 truncate w-full text-center tracking-tight">{{ c.statType || '빈칸' }}</span>
                      </div>
                    </div>
                    <div class="mt-1.5 text-xs text-neutral-500 bg-neutral-50 p-1.5 rounded border border-neutral-100 tracking-tight leading-snug">
                       <span class="font-bold text-emerald-600">발동된 세트효과:</span> {{ getCareerSetEffectText(playerBuffs[selectedSlot]?.careers) || '없음 (같은 옵션 3칸 이상 시 발동)' }}
                    </div>
                  </div>
                </div>

                <!-- 보유 시너지 현황 -->
                <div class="bg-indigo-50 dark:bg-indigo-900/10 p-2 rounded-xl border border-indigo-100 shadow-sm flex-shrink-0">
                  <div class="flex flex-col mb-2">
                    <h3 class="text-[15px] font-bold text-indigo-800 dark:text-indigo-300 mb-1.5 flex items-center gap-1"><Sparkles class="w-4 h-4"/> 개인 시너지 현황</h3>
                    <div class="flex bg-white p-0.5 rounded-lg border border-indigo-100 flex-shrink-0">
                      <button v-for="cat in ['전체', '기본', '출신', '기록', '인물']" :key="cat" @click="playerSynergyCategory = cat" :class="playerSynergyCategory === cat ? 'bg-indigo-100 font-bold text-indigo-700 shadow-sm' : 'text-neutral-500 hover:bg-neutral-50'" class="flex-1 py-1 text-xs rounded transition-all">{{ cat }}</button>
                    </div>
                  </div>

                  <div class="flex flex-col gap-1 mb-2 max-h-48 overflow-y-auto custom-scrollbar pr-1">
                     <div v-if="filteredPlayerActiveSynergies.length > 0" class="flex flex-col gap-0.5">
                       <div class="text-xs font-black text-indigo-600 mb-0.5">🟢 활성화 됨</div>
                       <div v-for="(rawSyn, idx) in filteredPlayerActiveSynergies" :key="'act_'+idx" class="flex justify-between items-center text-xs bg-white px-2 py-1.5 rounded border border-indigo-200 shadow-sm flex-shrink-0">
                          <span class="font-bold text-indigo-800">{{ rawSyn }}</span>
                          <span class="text-indigo-500 font-black whitespace-nowrap ml-1">적용중</span>
                       </div>
                     </div>
                     <div v-if="filteredPlayerInactiveSynergies.length > 0" class="flex flex-col gap-0.5 mt-1.5">
                       <div class="text-xs font-black text-neutral-500 mb-0.5 border-t border-indigo-100 pt-1.5">🔴 발동 대기 (필요 인원)</div>
                       <div v-for="(rawSyn, idx) in filteredPlayerInactiveSynergies" :key="'inact_'+idx" class="flex justify-between items-center text-xs bg-neutral-100 px-2 py-1.5 rounded border border-neutral-200 opacity-70 flex-shrink-0">
                          <span class="text-neutral-600 font-medium">{{ rawSyn }}</span>
                          <span class="text-red-600 font-bold bg-red-100 px-1.5 py-0.5 rounded whitespace-nowrap ml-1 border border-red-200">{{ getPendingSynergyText(rawSyn) }}</span>
                       </div>
                     </div>
                     <div v-if="filteredPlayerActiveSynergies.length === 0 && filteredPlayerInactiveSynergies.length === 0" class="text-xs text-neutral-400 text-center py-2">해당 시너지가 없습니다.</div>
                  </div>

                  <div class="grid grid-cols-2 gap-2 mt-2 pt-2 border-t border-indigo-100">
                    <div class="flex flex-col gap-0.5">
                      <label class="text-[11px] font-bold text-neutral-500">총 시너지 깡파워</label>
                      <div class="w-full px-2 py-1.5 text-center bg-indigo-100 border border-indigo-200 rounded text-sm font-black text-indigo-800">+{{ getPlayerSynergySum(lineup[selectedSlot], 'fixed', activeDeck) }}</div>
                    </div>
                    <div class="flex flex-col gap-0.5">
                      <label class="text-[11px] font-bold text-neutral-500">총 시너지 %파워</label>
                      <div class="w-full px-2 py-1.5 text-center bg-indigo-100 border border-indigo-200 rounded text-sm font-black text-indigo-800">+{{ getPlayerSynergySum(lineup[selectedSlot], 'percent', activeDeck) }}%</div>
                    </div>
                  </div>
                </div>
                
                <!-- 각인 장착 슬롯 -->
                <div class="bg-white p-2 rounded-xl border border-neutral-200 shadow-sm flex-shrink-0">
                  <div class="flex justify-between items-center mb-1.5">
                    <h3 class="text-sm font-bold text-neutral-800 flex items-center gap-1">🛡️ 각인 장착 (최대 2개)</h3>
                    <button @click="showImprintManager = true" class="text-xs bg-indigo-100 text-indigo-700 px-2.5 py-1 rounded hover:bg-indigo-200 transition-colors font-bold">대장간(보관함)</button>
                  </div>
                  
                  <div class="grid grid-cols-2 gap-1.5">
                    <div v-for="slotNum in [1, 2]" :key="'imp_slot_'+slotNum" 
                         class="border rounded-lg p-2 min-h-[85px] flex flex-col justify-center cursor-pointer transition-colors relative"
                         :class="playerBuffs[selectedSlot]?.[`imprint${slotNum}`] ? 'border-indigo-400 bg-indigo-50/50' : 'border-dashed border-neutral-300 hover:bg-neutral-50'"
                         @click="openEquipModal(selectedSlot, slotNum)">
                       
                       <div v-if="playerBuffs[selectedSlot]?.[`imprint${slotNum}`]" class="w-full relative">
                         <button @click.stop="unequipImprint(selectedSlot, slotNum)" class="absolute -top-2 -right-2 w-5 h-5 bg-red-100 text-red-500 rounded-full flex items-center justify-center text-xs hover:bg-red-500 hover:text-white transition-colors z-10 shadow-sm">×</button>
                         
                         <div class="flex items-center gap-1 mb-1">
                           <div class="text-[10px] font-black px-1.5 py-0.5 rounded border" :class="getGradeColor(playerBuffs[selectedSlot][`imprint${slotNum}`].grade)">{{ playerBuffs[selectedSlot][`imprint${slotNum}`].grade }}</div>
                           <div class="text-xs font-bold text-neutral-800 truncate">{{ playerBuffs[selectedSlot][`imprint${slotNum}`].name }}</div>
                         </div>
                         
                         <div class="flex flex-col text-[10px] text-neutral-600 gap-0.5 mt-1 leading-snug">
                           <div class="font-black text-indigo-700">주옵({{ playerBuffs[selectedSlot][`imprint${slotNum}`].mainStat }}): +{{ playerBuffs[selectedSlot][`imprint${slotNum}`].mainPower }}</div>
                           <div v-for="(opt, idx) in playerBuffs[selectedSlot][`imprint${slotNum}`].subOptions" :key="idx" class="font-medium">- {{ opt.type }} +{{ opt.value }}</div>
                           <div v-if="playerBuffs[selectedSlot][`imprint${slotNum}`].ultimateBonus" class="text-red-600 font-bold mt-0.5">
                             [등급효과] {{ playerBuffs[selectedSlot][`imprint${slotNum}`].ultimateBonus.targetGrade }}+{{ playerBuffs[selectedSlot][`imprint${slotNum}`].ultimateBonus.power }}
                           </div>
                         </div>
                       </div>
                       <div v-else class="text-xs text-neutral-400 font-bold text-center">+ 각인 {{ slotNum }} 장착</div>
                    </div>
                  </div>
                </div>

                <!-- 🌟 스킬 장착 (패시브 분리 + 인게임 이미지 그리드 적용) -->
                <div class="flex-shrink-0 bg-white dark:bg-neutral-800 p-2.5 rounded-xl border border-neutral-200 dark:border-neutral-700 shadow-sm">
                  <div class="flex items-center justify-between mb-2">
                    <h3 class="text-[15px] font-bold text-neutral-900 dark:text-neutral-100 flex items-center gap-1"><Star class="w-4 h-4 text-amber-400"/> 스킬 장착</h3>
                    <span class="text-xs bg-amber-100 text-amber-800 dark:bg-amber-900/30 dark:text-amber-300 px-2 py-0.5 rounded font-black">{{ playerBuffs[selectedSlot].selectedSkills.length }} / {{ getMaxSkillCount(lineup[selectedSlot]) }}</span>
                  </div>
                  
                  <!-- 1. 카드 고유 패시브 (강화스킬) 영역 - 이미지 및 레벨별 효과 표시 -->
                  <div class="flex flex-col gap-2 mb-3">
                    <template v-for="enh in getArray(lineup[selectedSlot]?.enhancedSkill)" :key="'enh_'+enh">
                      <div class="flex gap-2.5 p-2.5 rounded-xl border border-indigo-200 dark:border-indigo-800/50 bg-gradient-to-r from-indigo-50/50 to-blue-50/50 dark:from-indigo-900/10 dark:to-blue-900/10 shadow-sm cursor-default relative overflow-hidden">
                        <!-- 배경 장식 (선택) -->
                        <div class="absolute -right-4 -top-4 opacity-5 pointer-events-none">
                          <Sparkles class="w-20 h-20" />
                        </div>
                        
                        <!-- 스킬 이미지 -->
                        <div class="w-12 h-12 shrink-0 rounded-lg overflow-hidden relative flex items-center justify-center shadow-sm border border-neutral-200 dark:border-neutral-700 bg-white dark:bg-neutral-800">
                          <div v-if="matchEnhancedSkillImage(enh)" :class="matchEnhancedSkillImage(enh)" style="position: absolute; top: 50%; left: 50%; transform: translate(-50%, -50%) scale(0.98);"></div>
                          <Sparkles v-else class="w-6 h-6 text-indigo-400" />
                        </div>
                        
                        <!-- 텍스트 영역 -->
                        <div class="flex flex-col min-w-0 flex-1 justify-center">
                          <div class="flex items-center gap-1.5 mb-1">
                            <span class="text-sm font-black text-indigo-900 dark:text-indigo-300 tracking-tight">{{ enh }}</span>
                            <span class="text-[9px] font-bold bg-white dark:bg-neutral-800 text-indigo-600 dark:text-indigo-400 px-1.5 py-0.5 rounded border border-indigo-200 dark:border-indigo-800 shadow-sm ml-auto shrink-0 tracking-tighter">
                              Lv.{{ playerBuffs[selectedSlot].enhancementLevel }} 효과
                            </span>
                          </div>
                          <div class="text-[11px] font-medium text-neutral-600 dark:text-neutral-400 leading-snug break-keep whitespace-pre-wrap">
                            {{ getEnhancedSkillEffect(enh, playerBuffs[selectedSlot].enhancementLevel) }}
                          </div>
                        </div>
                      </div>
                    </template>
                  </div>

                  <!-- 🌟 2. 클릭해서 장착할 수 있는 스킬 목록 (이미지 갤러리) -->
                  <div class="grid grid-cols-4 sm:grid-cols-5 gap-1.5 pb-4">
                    <button 
                      v-for="sk in getAvailableSkills(lineup[selectedSlot])" 
                      :key="sk"
                      @click="togglePlayerSkill(sk)"
                      @mouseenter="showSkillTooltip($event, sk)"
                      @mouseleave="hideSkillTooltip"

                      :class="[
                        playerBuffs[selectedSlot].selectedSkills.includes(sk) 
                          ? 'bg-indigo-600 text-white border-indigo-600 shadow-md dark:bg-indigo-700 dark:border-indigo-700' 
                          : 'bg-neutral-50 border-neutral-200 text-neutral-700 hover:bg-neutral-100 dark:bg-neutral-700 dark:border-neutral-600 dark:text-neutral-200 dark:hover:bg-neutral-600',
                        playerBuffs[selectedSlot].selectedSkills.includes(sk) && !isSkillActive(sk, selectedSlot, playerBuffs[selectedSlot].battingOrder) 
                          ? 'bg-red-500 border-red-600 text-white dark:bg-red-600 dark:border-red-700' : ''
                      ]"
                      class="group relative inline-flex flex-col items-center justify-center gap-1 rounded-xl border py-1.5 text-[10px] font-medium select-none transition-all duration-200"
                    >
                      <div class="w-8 h-8 rounded-md" :class="['bg-neutral-200 dark:bg-neutral-600', playerBuffs[selectedSlot].selectedSkills.includes(sk) ? 'ring-2 ring-white/50 bg-white/20' : '', `bg-${matchSkillInfo(sk)}`]"></div>
                      <span class="block w-full text-center font-semibold truncate px-0.5" 
                            :class="playerBuffs[selectedSlot].selectedSkills.includes(sk) ? 'text-white' : 'text-neutral-700 dark:text-neutral-300'">{{ sk }}</span>
                      
                      <!-- 조건 불일치 경고 뱃지 -->
                      <span v-if="playerBuffs[selectedSlot].selectedSkills.includes(sk) && !isSkillActive(sk, selectedSlot, playerBuffs[selectedSlot].battingOrder)" 
                            class="absolute -top-1.5 -right-1.5 bg-white text-red-600 border border-red-500 text-[9px] font-black px-1 rounded-full shadow-sm whitespace-nowrap z-10 scale-90 origin-bottom-left">
                        불일치
                      </span>
                    </button>
                  </div>
                </div>
                </div>
                </div>
            
            <div v-else class="flex h-full items-center justify-center text-neutral-400 text-sm flex-col gap-2">
              <UserCheck class="w-10 h-10 opacity-20"/>
              중앙에서 선수를 클릭해주세요.
            </div>
          </div>
        </section>
        </div> <!-- 🌟 복구 1. 좌/우/중앙 Flex 분할 컨테이너 닫기 -->
    </div> <!-- 🌟 복구 2. 전체 패널 영역 닫기 -->
  </div> <!-- 🌟 복구 3. 메인 배경 화면 닫기 -->

  <!-- 🌟 각인 생성 및 보관함 (대장간) 모달 -->
  <div v-if="showImprintManager" class="fixed inset-0 z-50 flex items-center justify-center bg-black/50 p-4">
    <div class="bg-white dark:bg-neutral-900 w-full max-w-lg rounded-2xl shadow-xl overflow-hidden flex flex-col max-h-[85vh]">
      <div class="flex justify-between items-center p-4 border-b dark:border-neutral-800 bg-neutral-50 dark:bg-neutral-900">
        <h2 class="text-base font-bold text-neutral-800 dark:text-neutral-200">🛡️ 각인 보관함 (대장간)</h2>
        <button @click="showImprintManager = false" class="text-neutral-500 hover:text-neutral-800 dark:hover:text-white text-2xl font-bold">&times;</button>
      </div>
      
      <div class="p-4 border-b dark:border-neutral-800 flex flex-col gap-3 bg-white dark:bg-neutral-800/80">
        <div class="flex gap-2 items-center">
          <select v-model="newImprint.role" @change="handleRoleChange" class="text-xs border rounded p-2 font-bold bg-neutral-100 text-neutral-700 outline-none">
            <option value="타자">⚾ 타자용</option>
            <option value="투수">⚾ 투수용</option>
          </select>
          <select v-model="newImprint.grade" @change="updateSubOptionsCount" class="text-xs border rounded p-2 font-bold" :class="getGradeColor(newImprint.grade)">
            <option value="노말">노말</option>
            <option value="고급">고급</option>
            <option value="특별">특별</option>
            <option value="레전드">레전드</option>
            <option value="얼티밋">얼티밋</option>
          </select>
          <input v-model="newImprint.name" type="text" placeholder="각인 이름" class="flex-1 text-xs border rounded p-2">
        </div>
        
        <div class="flex items-center gap-2 border rounded p-2 bg-indigo-50/50 border-indigo-100">
          <span class="text-[10px] text-indigo-700 font-bold shrink-0">주옵션 설정 :</span>
          <select v-model="newImprint.mainStat" class="text-xs font-bold bg-white border rounded p-1 text-indigo-700 outline-none flex-1">
             <template v-if="newImprint.role === '타자'">
               <option value="컨택">컨택</option><option value="갭파워">갭파워</option><option value="홈런파워">홈런파워</option><option value="선구">선구</option><option value="삼진회피">삼진회피</option>
             </template>
             <template v-else>
               <option value="무브먼트">무브먼트</option><option value="장타억제">장타억제</option><option value="홈런억제">홈런억제</option><option value="컨트롤">컨트롤</option><option value="스터프">스터프</option>
             </template>
          </select>
          <span class="text-[11px] text-neutral-400 font-black">+</span>
          <input v-model="newImprint.mainPower" type="number" class="w-16 text-xs bg-white border rounded p-1 outline-none text-right font-black text-indigo-600">
        </div>

        <div v-if="newImprint.subOptions.length > 0" class="flex flex-col gap-1.5 bg-neutral-50 p-2 rounded border">
          <div class="text-[10px] font-bold text-neutral-600 mb-1">부가 효과 설정 (레전드/얼티밋 기본 3줄)</div>
          <div v-for="(opt, idx) in newImprint.subOptions" :key="idx" class="flex gap-2">
            <select v-model="opt.type" class="text-xs border rounded p-1.5 flex-1 text-neutral-700 font-medium">
              <template v-if="newImprint.role === '타자'">
                <option value="컨택">컨택</option><option value="갭파워">갭파워</option><option value="홈런파워">홈런파워</option><option value="선구">선구</option><option value="삼진회피">삼진회피</option>
              </template>
              <template v-else>
                <option value="무브먼트">무브먼트</option><option value="장타억제">장타억제</option><option value="홈런억제">홈런억제</option><option value="컨트롤">컨트롤</option><option value="스터프">스터프</option><option value="한계투구 증가">한계투구 증가</option><option value="1~2선발시 파워증가">1~2선발시 파워증가</option>
              </template>
              <option value="수비">수비</option><option value="전체 능력치">전체 능력치 (코어 5종 +수치)</option><option value="조건부 파워">조건부 파워 (박빙/주자 등)</option><option value="수익 증가">경기 총 수익 증가</option>
            </select>
            <input v-model="opt.value" type="number" placeholder="수치" class="w-20 text-xs border rounded p-1.5 text-center">
          </div>
        </div>

        <div v-if="newImprint.grade === '얼티밋'" class="flex items-center gap-2 bg-red-50 border border-red-200 p-2 rounded">
          <span class="text-[10px] font-bold text-red-600 shrink-0">등급 효과(얼티밋)</span>
          <select v-model="newImprint.ultimateBonus.targetGrade" class="text-xs border rounded p-1 text-red-800">
            <option value="DGN">디그니티 (DGN)</option><option value="TOP">탑클래스 (TOP)</option><option value="GG">골글 (GG)</option><option value="ACE">에이스 (ACE)</option><option value="HIT">히트 (HIT)</option><option value="GGY">연도골글 (GGY)</option><option value="MMVP">월간MVP (MMVP)</option><option value="ROY">신인왕 (ROY)</option><option value="TEA">팀플 (TEA)</option>
          </select>
          <span class="text-[10px] text-neutral-500 font-black">+</span>
          <input v-model="newImprint.ultimateBonus.power" type="number" class="w-16 text-xs border rounded p-1 text-center font-bold text-red-600 outline-none bg-white">
        </div>
        <button @click="createImprint" class="w-full bg-indigo-600 text-white px-4 py-2 rounded font-bold hover:bg-indigo-700 text-sm mt-1 transition-colors">이 설정으로 각인 만들기</button>
      </div>

      <div class="p-4 overflow-y-auto flex-1 custom-scrollbar bg-neutral-100 dark:bg-neutral-900">
        <h3 class="text-xs font-bold text-neutral-600 mb-2">보관 중인 각인 ({{ imprintInventory.length }}개)</h3>
        <div class="flex flex-col gap-2">
          <div v-for="imp in imprintInventory" :key="imp.id" class="flex justify-between items-center border border-neutral-200 rounded-lg p-3 bg-white shadow-sm">
            <div class="flex flex-col">
              <div class="flex items-center gap-1.5 mb-1">
                <span class="text-[9px] font-black px-1.5 py-0.5 rounded" :class="imp.role === '타자' ? 'bg-orange-100 text-orange-700' : 'bg-blue-100 text-blue-700'">{{ imp.role }}</span>
                <span class="text-[9px] font-black px-1.5 py-0.5 rounded border" :class="getGradeColor(imp.grade)">{{ imp.grade }}</span>
                <span class="font-bold text-sm">{{ imp.name }}</span>
                <span class="text-[10px] text-indigo-600 font-bold ml-1">주옵({{ imp.mainStat }})+{{ imp.mainPower }}</span>
              </div>
              <div class="text-[10px] text-neutral-500 flex flex-wrap gap-x-2">
                <span v-for="(opt, idx) in imp.subOptions" :key="idx" :class="{'text-orange-500 font-bold': opt.type === '전체 능력치'}">
                  {{ opt.type }}+{{ opt.value }}
                </span>
                <span v-if="imp.ultimateBonus" class="text-red-500 font-bold">
                  [등급효과] {{ imp.ultimateBonus.targetGrade }}+{{ imp.ultimateBonus.power }}
                </span>
              </div>
            </div>
            <button @click="deleteImprint(imp.id)" class="text-[10px] bg-red-100 text-red-600 px-3 py-1.5 rounded font-bold hover:bg-red-200 transition-colors">삭제</button>
          </div>
        </div>
      </div>
    </div>
  </div>

  <!-- 🌟 각인 장착 모달 -->
  <div v-if="showImprintEquipper" class="fixed inset-0 z-50 flex items-center justify-center bg-black/50 p-4">
    <div class="bg-white dark:bg-neutral-900 w-full max-w-sm rounded-2xl shadow-xl overflow-hidden flex flex-col max-h-[70vh]">
      <div class="flex justify-between items-center p-4 border-b bg-neutral-50">
        <h2 class="text-sm font-bold text-neutral-800">
          각인 선택 ({{ equipTarget?.pos?.startsWith('SP') || equipTarget?.pos?.startsWith('RP') ? '투수' : '타자' }}용 슬롯 {{ equipTarget?.slot }})
        </h2>
        <button @click="showImprintEquipper = false" class="text-neutral-500 text-2xl font-bold">&times;</button>
      </div>
      <div class="p-4 overflow-y-auto flex-1 custom-scrollbar">
        <div v-for="imp in imprintInventory.filter(i => i.role === (equipTarget?.pos?.startsWith('SP') || equipTarget?.pos?.startsWith('RP') ? '투수' : '타자'))" 
             :key="'eq_'+imp.id" @click="equipImprint(imp)" 
             class="flex justify-between items-center border rounded-lg p-2.5 mb-2 bg-white shadow-sm cursor-pointer hover:border-indigo-500 hover:bg-indigo-50 transition-all">
          <div class="flex flex-col">
            <div class="flex items-center gap-1.5 mb-1">
              <span class="text-[8px] font-black px-1.5 py-0.5 rounded border" :class="getGradeColor(imp.grade)">{{ imp.grade }}</span>
              <span class="font-bold text-xs">{{ imp.name }}</span>
            </div>
            <span class="text-indigo-600 text-[10px] font-bold">주옵({{ imp.mainStat }})+{{ imp.mainPower }} / 부옵 {{ imp.subOptions.length }}개</span>
          </div>
          <span class="text-[10px] bg-indigo-600 text-white px-2.5 py-1 rounded font-bold">장착</span>
        </div>
        
        <div v-if="imprintInventory.filter(i => i.role === (equipTarget?.pos?.startsWith('SP') || equipTarget?.pos?.startsWith('RP') ? '투수' : '타자')).length === 0" class="text-center text-neutral-400 text-xs py-6">
          이 선수에게 장착할 수 있는 [{{ equipTarget?.pos?.startsWith('SP') || equipTarget?.pos?.startsWith('RP') ? '투수' : '타자' }}용] 각인이 없습니다.<br>대장간에서 생성해 주세요!
        </div>
      </div>
    </div>
  </div>
  <!-- 🌟 커리어 장착 (편집) 모달 🌟 -->
  <div v-if="showCareerManager && selectedSlot" class="fixed inset-0 z-50 flex items-center justify-center bg-black/50 p-4">
    <div class="bg-white dark:bg-neutral-900 w-full max-w-md rounded-2xl shadow-xl overflow-hidden flex flex-col">
      <div class="flex justify-between items-center p-4 border-b bg-neutral-50 dark:bg-neutral-800">
         <h2 class="text-sm font-bold text-neutral-800 dark:text-neutral-100">🎖️ 커리어 설정 ({{ lineup[selectedSlot].name }})</h2>
         <button @click="showCareerManager = false" class="text-neutral-500 text-2xl font-bold">&times;</button>
      </div>
      <div class="p-4 flex flex-col gap-2 overflow-y-auto bg-neutral-100 dark:bg-neutral-900">
         <div v-for="(c, i) in playerBuffs[selectedSlot].careers" :key="i" class="flex items-center gap-2 border p-2 rounded-lg bg-white dark:bg-neutral-800 shadow-sm">
            <span class="text-xs font-black text-neutral-400 w-4">{{ i+1 }}</span>
            <select v-model="c.grade" @change="onCareerChange(c)" class="text-xs border rounded p-1 font-bold outline-none" :class="getCareerGradeColor(c.grade)">
               <option value="루키">루키</option><option value="엘리트">엘리트</option><option value="프로">프로</option><option value="마스터">마스터</option>
            </select>
            <select v-model="c.statType" @change="onCareerChange(c)" class="text-xs border rounded p-1 flex-1 font-bold outline-none text-neutral-700">
               <option value="">옵션 선택</option>
               <option v-for="opt in getAvailableCareerStatTypes(lineup[selectedSlot], c.grade)" :key="opt" :value="opt">{{ opt }}</option>
            </select>
            <select v-if="c.statType && c.statType !== '동일팀 파워'" v-model.number="c.value" class="text-xs border rounded p-1 w-16 text-center font-black text-indigo-600 outline-none">
               <option v-for="v in getAvailableCareerValues(c.grade, c.statType)" :key="v" :value="v">+{{ v }}</option>
            </select>
            <span v-else-if="c.statType === '동일팀 파워'" class="text-[10px] font-bold text-emerald-600 w-16 text-center bg-emerald-50 rounded py-1">자동 계산</span>
            <span v-else class="w-16"></span>
         </div>
         
         <button @click="setAllCareersToMaster" class="mt-2 text-xs bg-indigo-600 text-white py-2.5 rounded-lg font-bold hover:bg-indigo-700 shadow-md transition-colors">🚀 일괄 마스터(6칸) 자동 등급 세팅</button>
      </div>
    </div>
  </div>
<!-- 🌟 라인업 타순 일괄 변경 모달 (인게임 UI 완벽 적용) 🌟 -->
  <div v-if="showBattingOrderManager" class="fixed inset-0 z-50 flex items-center justify-center bg-black/80 p-2 sm:p-4 backdrop-blur-sm">
    <div class="bg-neutral-900 w-full max-w-5xl rounded-2xl shadow-2xl overflow-hidden flex flex-col max-h-[90vh] border border-neutral-700">
      
      <!-- 헤더 -->
      <div class="flex justify-between items-center p-3 sm:p-4 border-b border-neutral-800 bg-black">
         <h2 class="text-base sm:text-lg font-black tracking-tight flex items-center gap-2 text-white">⚾ 라인업 타순 일괄 설정</h2>
         <button @click="showBattingOrderManager = false" class="text-neutral-400 hover:text-white text-3xl font-bold leading-none transition-colors">&times;</button>
      </div>
      
      <!-- 바디 -->
      <div class="p-3 sm:p-5 flex flex-col gap-4 overflow-y-auto custom-scrollbar bg-neutral-900">
         
         <div class="text-[11px] sm:text-xs font-bold text-indigo-300 bg-indigo-900/30 p-3 rounded-xl border border-indigo-800/50 shadow-sm leading-relaxed">
            💡 <strong>자리 교체(Swap):</strong> 드롭다운 번호를 바꾸면, 기존 선수와 자동으로 자리를 바꿉니다!<br>
            💡 <strong class="text-amber-400">타순 스킬 램프:</strong> 타순 전용 스킬이 장착된 선수는 조건 일치 시 카드 내부에 <span class="text-amber-400 drop-shadow-md">황금색 램프</span>가 켜집니다.
         </div>
         
         <!-- 🌟 이름, 파워를 내부로 넣어서 카드를 큼직하게 키운 랩핑 그리드 -->
         <div class="grid grid-cols-3 sm:grid-cols-4 lg:grid-cols-5 gap-2.5 sm:gap-4">
            <div v-for="pos in sortedBattersForOrder" :key="pos" 
                 draggable="true"
                 @dragstart="onBattingOrderDragStart($event, pos)"
                 @dragover.prevent
                 @drop="onBattingOrderDrop($event, pos)"
                 class="flex flex-col bg-neutral-800/80 p-1.5 sm:p-2 rounded-xl border border-neutral-700 shadow-lg relative transition-all hover:border-indigo-500 hover:bg-neutral-800 group cursor-grab active:cursor-grabbing">
               
               <!-- 상단: 포지션 & 타순 선택 (공간 최소화) -->
               <div class="flex justify-between items-center mb-1.5 px-0.5 sm:px-1">
                  <div class="text-[10px] sm:text-xs font-black" :class="playerBuffs[pos]?.battingOrder ? 'text-indigo-400' : 'text-neutral-500'">{{ pos }}</div>
                  <select :value="playerBuffs[pos]?.battingOrder" @change="handleBattingOrderChange(pos, Number($event.target.value) || null)" class="border border-neutral-600 rounded bg-black px-1 sm:px-1.5 py-0.5 text-[10px] sm:text-xs font-bold text-white outline-none focus:ring-1 focus:ring-indigo-500 cursor-pointer text-center">
                     <option :value="null">대기</option>
                     <option v-for="i in 9" :key="i" :value="i">{{ i }}번</option>
                  </select>
               </div>
               
               <!-- 중앙: 카드 본체 (이름, 파워, 스킬 모두 내부로 삽입) -->
               <div class="relative w-full aspect-[5/7] rounded-lg border border-neutral-700 overflow-hidden bg-black shadow-inner">
                  <img :src="getPlayerImage(lineup[pos])" class="absolute inset-0 w-full h-full object-contain" @error="hideImage"/>
                  
                  <!-- 좌측 상단: 현재 타순 뱃지 -->
                  <div class="absolute top-1 left-1 w-5 h-5 sm:w-6 sm:h-6 rounded bg-black/70 backdrop-blur-md text-white font-black flex items-center justify-center text-[10px] sm:text-xs border border-white/20 z-10 shadow-sm">
                     {{ playerBuffs[pos]?.battingOrder || '-' }}
                  </div>
                  
                  <!-- 하단 정보 오버레이 (인게임 방식: 위부터 스킬 -> 파워 -> 이름) -->
                  <div class="absolute bottom-0 inset-x-0 h-[60%] bg-gradient-to-t from-black/95 via-black/60 to-transparent flex flex-col justify-end items-center pb-1.5 px-1 pointer-events-none">
                     
                     <!-- 🌟 타순 스킬 표시기 (조건부 점등) -->
                     <div class="flex flex-col items-center gap-0.5 mb-1 z-10 w-full">
                        <template v-for="sk in playerBuffs[pos]?.selectedSkills || []" :key="sk">
                           <div v-if="['1번', '2번', '3번', '4번', '5번', '6번', '7번', '8번', '9번', '테이블세터', '클린업', '하위타선'].includes(sk)" 
                                class="flex items-center justify-center px-1.5 py-0.5 rounded text-[9px] sm:text-[10px] font-black tracking-tighter border transition-all duration-300 w-[90%] max-w-[50px] truncate"
                                :class="isSkillActive(sk, pos, playerBuffs[pos]?.battingOrder) 
                                   ? 'bg-gradient-to-b from-amber-500 to-amber-700 text-amber-100 border-amber-300 shadow-[0_0_6px_rgba(251,191,36,0.8)]' 
                                   : 'bg-gradient-to-b from-neutral-800 to-neutral-950 text-neutral-500 border-neutral-600 opacity-80'">
                              {{ 
                                sk === '테이블세터' ? '1·2' : 
                                sk === '클린업' ? '3·4·5' : 
                                sk === '하위타선' ? '6~9' : 
                                sk.replace('번', '') 
                              }}
                           </div>
                        </template>
                     </div>
                     
                     <!-- 파워 (노란색) -->
                     <div class="text-[12px] sm:text-[14px] font-black text-amber-400 tracking-tight drop-shadow-md leading-none mb-0.5">
                       {{ calculatePlayerPower(lineup[pos], pos).toLocaleString() }}
                     </div>
                     
                     <!-- 이름 (흰색) -->
                     <div class="text-[11px] sm:text-[13px] font-bold text-white w-full text-center truncate drop-shadow-md leading-none">
                       {{ lineup[pos]?.name }}
                     </div>
                  </div>
               </div>
               
            </div>
         </div>
         
         <!-- 빈 화면 -->
         <div v-if="sortedBattersForOrder.length === 0" class="py-12 flex flex-col items-center justify-center text-neutral-500">
             <div class="text-4xl mb-3 opacity-50">⚾</div>
             <div class="text-sm font-bold">라인업에 배치된 타자가 없습니다.</div>
         </div>
      </div>
      
      <!-- 푸터 -->
      <div class="p-3 border-t border-neutral-800 bg-black flex justify-end">
         <button @click="showBattingOrderManager = false" class="px-8 py-2 bg-indigo-600 hover:bg-indigo-500 text-white font-black rounded-lg shadow-md transition-colors text-sm">완료</button>
      </div>
    </div>
  </div>

  <!-- 🌟 라인업 저장소 관리 (불러오기/삭제) 모달 🌟 -->
  <div v-if="showSaveManager" class="fixed inset-0 z-50 flex items-center justify-center bg-black/60 p-4 backdrop-blur-sm">
    <div class="bg-white dark:bg-neutral-900 w-full max-w-md rounded-2xl shadow-2xl overflow-hidden flex flex-col max-h-[80vh] border border-neutral-200 dark:border-neutral-700">
      
      <div class="flex justify-between items-center p-4 border-b border-neutral-200 dark:border-neutral-800 bg-neutral-50 dark:bg-black">
         <h2 class="text-base font-black tracking-tight flex items-center gap-2 text-neutral-800 dark:text-white">
            <FolderOpen class="w-5 h-5 text-indigo-500" /> 라인업 불러오기 및 관리
         </h2>
         <button @click="showSaveManager = false" class="text-neutral-400 hover:text-neutral-800 dark:hover:text-white text-2xl font-bold leading-none transition-colors">&times;</button>
      </div>
      
      <div class="p-4 flex flex-col gap-2 overflow-y-auto custom-scrollbar bg-neutral-100 dark:bg-neutral-900">
         <!-- 저장된 라인업이 없을 때 -->
         <div v-if="savedLineupsList.length === 0" class="py-12 flex flex-col items-center justify-center text-neutral-400">
             <FolderOpen class="w-12 h-12 mb-3 opacity-30" />
             <div class="text-sm font-bold">브라우저에 저장된 라인업이 없습니다.</div>
         </div>
         
         <!-- 라인업 목록 (불러오기 / 삭제 버튼 포함) -->
         <div v-for="(name, idx) in savedLineupsList" :key="name" class="flex justify-between items-center bg-white dark:bg-neutral-800 p-3 rounded-xl border border-neutral-200 dark:border-neutral-700 shadow-sm transition-all hover:border-indigo-400 hover:shadow-md">
            <div class="flex items-center gap-3 overflow-hidden">
               <div class="w-6 h-6 shrink-0 rounded-full bg-indigo-100 dark:bg-indigo-900/50 text-indigo-600 dark:text-indigo-400 flex items-center justify-center text-xs font-black">{{ idx + 1 }}</div>
               <span class="font-bold text-neutral-800 dark:text-neutral-200 text-sm truncate" :title="name">{{ name }}</span>
            </div>
            
            <div class="flex items-center gap-1.5 shrink-0 pl-2">
               <button @click="loadSpecificSave(name)" class="px-3 py-1.5 bg-indigo-50 dark:bg-indigo-900/30 text-indigo-600 dark:text-indigo-400 hover:bg-indigo-600 hover:text-white dark:hover:bg-indigo-500 rounded-lg text-xs font-black transition-colors border border-indigo-200 dark:border-indigo-800 shadow-sm">불러오기</button>
               <button @click="deleteSpecificSave(name)" class="px-3 py-1.5 bg-rose-50 dark:bg-rose-900/30 text-rose-600 dark:text-rose-400 hover:bg-rose-600 hover:text-white dark:hover:bg-rose-500 rounded-lg text-xs font-black transition-colors border border-rose-200 dark:border-rose-800 shadow-sm">삭제</button>
            </div>
         </div>
      </div>
    </div>
  </div>

  <!-- 🌟 자동 사라짐 토스트 (Toast) 알림 UI 🌟 -->
  <div class="fixed bottom-6 right-6 z-[999999] flex flex-col gap-2.5 pointer-events-none">
    <transition-group name="toast">
      <div v-for="toast in toasts" :key="toast.id"
           class="px-4 py-3.5 rounded-xl shadow-[0_10px_40px_-10px_rgba(0,0,0,0.5)] font-bold text-[13px] sm:text-sm text-white flex items-center gap-3 pointer-events-auto border border-white/10"
           :class="toast.type === 'error' ? 'bg-neutral-800/95' : 'bg-neutral-800/95'">
         <span class="text-xl leading-none" v-if="toast.type === 'error'">🚫</span>
         <span class="text-xl leading-none" v-else>✅</span>
         <span class="tracking-tight drop-shadow-md pr-1">{{ toast.msg }}</span>
      </div>
    </transition-group>
  </div>

  <!-- 🌟 글로벌 스킬 툴팁 (화면 밖 잘림 완벽 방지) 🌟 -->
  <div v-if="tooltipState.show" 
       class="fixed z-[99999] pointer-events-none drop-shadow-2xl transition-all duration-75"
       :style="{ 
         top: (tooltipState.y - 8) + 'px', 
         left: tooltipState.x + 'px',
         transform: tooltipState.transform
       }">
      <div class="bg-neutral-900 dark:bg-white text-neutral-100 dark:text-neutral-900 text-[11px] font-medium px-3 py-2.5 rounded-xl shadow-2xl text-left leading-relaxed whitespace-pre-wrap border border-neutral-700 dark:border-neutral-200 tracking-tight w-max max-w-[240px]">
        {{ getNormalSkillDescription(tooltipState.skill) }}
      </div>
      <div class="absolute bottom-0 w-3 h-3 bg-neutral-900 dark:bg-white rotate-45 border-r border-b border-neutral-700 dark:border-neutral-200"
           :style="{ left: tooltipState.arrowLeft, transform: 'translate(-50%, 50%)' }"></div>
  </div>
<!-- 🌟 OCR 분석 진행 오버레이 모달 -->
  <div v-if="isOcrProcessing" class="fixed inset-0 z-[999999] flex items-center justify-center bg-black/75 backdrop-blur-sm p-4">
    <div class="bg-neutral-900 border border-neutral-700 p-6 rounded-2xl shadow-2xl flex flex-col items-center max-w-xs w-full text-center">
      <div class="animate-spin rounded-full border-4 border-neutral-700 border-t-amber-400 h-10 w-10 mb-4"></div>
      <h3 class="text-sm font-black text-white mb-1">인게임 라인업 스캔 중</h3>
      <p class="text-xs font-bold text-amber-400">{{ ocrProgressText }}</p>
      <span class="text-[10px] text-neutral-400 mt-3">기기 사양에 따라 5~10초 정도 소요될 수 있습니다.</span>
    </div>
  </div>
<!-- 🌟 FC 온라인 스타일: 동일 선수 시즌 교체 모달 🌟 -->
  <div v-if="showCardSwapModal && swapTargetSlot" class="fixed inset-0 z-[99999] flex items-center justify-center bg-black/70 backdrop-blur-sm p-4">
    <div class="bg-white dark:bg-neutral-900 w-full max-w-lg rounded-2xl shadow-2xl overflow-hidden flex flex-col max-h-[85vh] border border-neutral-200 dark:border-neutral-700">
      
      <!-- 모달 헤더 -->
      <div class="flex justify-between items-center p-4 border-b border-neutral-200 dark:border-neutral-800 bg-neutral-50 dark:bg-black">
         <div class="flex items-center gap-2">
            <RefreshCw class="w-4 h-4 text-indigo-600 dark:text-indigo-400" />
            <h2 class="text-base font-black tracking-tight text-neutral-800 dark:text-white">
               [{{ lineup[swapTargetSlot]?.name }}] 시즌/등급 교체
            </h2>
         </div>
         <button @click="showCardSwapModal = false" class="text-neutral-400 hover:text-white text-2xl font-bold leading-none">&times;</button>
      </div>

      <!-- 모달 바디: 카드 목록 -->
      <div class="p-4 overflow-y-auto custom-scrollbar flex flex-col gap-2.5 bg-neutral-100 dark:bg-neutral-900">
         <div class="text-[11px] font-bold text-neutral-500 mb-1">
            원하는 시즌 카드를 클릭하면 즉시 해당 슬롯({{ swapTargetSlot }})에 장착됩니다.
         </div>

         <div v-for="cand in swapCandidates" :key="cand.id || (cand.name + cand.grade + cand.year)"
              @click="applyCardSwap(cand)"
              class="flex items-center justify-between p-2.5 rounded-xl border bg-white dark:bg-neutral-800 cursor-pointer transition-all hover:border-indigo-500 hover:shadow-md"
              :class="isSameCard(lineup[swapTargetSlot]!, cand) ? 'border-indigo-500 bg-indigo-50/50 dark:bg-indigo-900/20 ring-1 ring-indigo-400' : 'border-neutral-200 dark:border-neutral-700'">
            
            <div class="flex items-center gap-3">
               <div class="w-12 h-12 rounded-lg border border-neutral-200 dark:border-neutral-700 bg-neutral-50 dark:bg-neutral-700/50 flex items-center justify-center shrink-0 overflow-hidden">
                  <img v-if="getGradeImage(cand.grade)" :src="getGradeImage(cand.grade)" class="w-10 object-contain" @error="hideImage" />
                  <span v-else class="text-[9px] font-black text-neutral-400">{{ cand.grade }}</span>
               </div>
               
               <div class="flex flex-col">
                  <div class="flex items-center gap-2">
                     <span class="font-black text-sm text-neutral-900 dark:text-white">{{ cand.name }}</span>
                     <span class="text-[10px] font-bold px-1.5 py-0.5 rounded bg-neutral-100 dark:bg-neutral-700 text-neutral-600 dark:text-neutral-300 border border-neutral-200 dark:border-neutral-600">
                        {{ cand.grade }}
                     </span>
                     <span v-if="isSameCard(lineup[swapTargetSlot]!, cand)" class="text-[10px] font-black text-indigo-600 bg-indigo-100 dark:bg-indigo-900/50 px-1.5 py-0.5 rounded">
                        현재 장착 중
                     </span>
                  </div>
                  <div class="text-xs text-neutral-500 font-medium mt-0.5 flex items-center gap-1.5">
                     <span>{{ findTeamName(Array.isArray(cand.team) ? cand.team[0] : cand.team) }}</span>
                     <span>·</span>
                     <span class="font-bold text-neutral-700 dark:text-neutral-300">
                        {{ cand.year ? `'${String(cand.year).replace(/[\[\]]/g, '').slice(-2)}년` : '연도 없음' }}
                     </span>
                     <span>·</span>
                     <span>포지션: {{ getPlayerPositions(cand).join(', ') }}</span>
                  </div>
               </div>
            </div>

            <button class="px-3.5 py-1.5 bg-indigo-600 hover:bg-indigo-500 text-white rounded-lg text-xs font-black transition-colors shadow-sm shrink-0">
               선택
            </button>
         </div>
      </div>
    </div>
  </div> <!-- 👈 1. 카드 교체 모달이 여기서 정상적으로 닫힙니다 -->

  <!-- ======================================================= -->
  <!-- 🔍 2. 독립된 OCR 크롭 검사기 (화면 최상단 z-index 하단 서랍장) -->
  <!-- ======================================================= -->
  <div v-if="ocrDebugList.length > 0" class="fixed inset-x-0 bottom-0 z-[999999] max-h-[60vh] bg-neutral-950/95 backdrop-blur-md border-t-2 border-emerald-500 shadow-2xl p-4 overflow-y-auto text-white">
    <div class="max-w-7xl mx-auto">
      <div class="flex items-center justify-between mb-3 pb-2 border-b border-neutral-800">
        <div class="flex items-center gap-2">
          <span class="w-2.5 h-2.5 rounded-full bg-emerald-500 animate-pulse"></span>
          <h3 class="font-bold text-sm sm:text-base text-emerald-400">🔍 OCR 크롭 검사기 (9개 슬롯 실제 캡처 및 인식 결과)</h3>
          <span class="text-xs text-neutral-400 hidden sm:inline">사진 속 글자가 위아래로 잘리지 않았는지 확인하세요</span>
        </div>
        <button @click="ocrDebugList = []" class="px-2.5 py-1 text-xs bg-neutral-800 hover:bg-neutral-700 text-neutral-300 rounded border border-neutral-600 transition-colors font-bold">✕ 닫기</button>
      </div>
      
      <div class="grid grid-cols-3 sm:grid-cols-5 md:grid-cols-9 gap-2">
        <div 
          v-for="item in ocrDebugList" 
          :key="item.slot"
          class="bg-neutral-900 p-2 rounded-xl border text-center flex flex-col items-center shadow-lg"
          :class="item.matchedCard ? 'border-emerald-500/50' : 'border-rose-500/60'"
        >
          <div class="w-full flex items-center justify-between mb-1 px-1">
            <span class="font-black text-xs px-1.5 py-0.2 rounded" :class="item.matchedCard ? 'bg-emerald-600 text-white' : 'bg-rose-600 text-white'">
              {{ item.slot }}
            </span>
            <span class="text-[10px] font-bold" :class="item.matchedCard ? 'text-emerald-400' : 'text-rose-400'">
              {{ item.matchedCard ? '성공' : '실패' }}
            </span>
          </div>

          <!-- 실제 캡처된 크롭 이미지 -->
          <div class="w-full h-24 bg-black rounded-lg border border-neutral-800 flex items-center justify-center overflow-hidden mb-1 p-0.5">
            <img :src="item.imgUrl" class="w-full h-full object-contain" />
          </div>
          
          <!-- 확정된 카드 명 -->
          <div class="text-[11px] font-bold truncate w-full text-center" :class="item.matchedCard ? 'text-emerald-300' : 'text-rose-400'">
            {{ item.matchedCard || '미인식' }}
          </div>
          
          <!-- OCR이 읽은 날것의 텍스트 -->
          <div class="text-[9px] text-neutral-400 truncate w-full mt-1 bg-neutral-950 px-1 py-0.5 rounded border border-neutral-800" :title="item.rawText">
            "{{ item.rawText || '텍스트 없음' }}"
          </div>
        </div>
      </div>
    </div>
  </div>
</template>

<style scoped>
.toast-enter-active,
.toast-leave-active {
  transition: all 0.4s cubic-bezier(0.2, 0.8, 0.2, 1);
}
.toast-enter-from {
  opacity: 0;
  transform: translateX(30px) scale(0.95);
}
.toast-leave-to {
  opacity: 0;
  transform: translateY(-20px) scale(0.95);
}
.custom-scrollbar::-webkit-scrollbar { width: 4px; }
.custom-scrollbar::-webkit-scrollbar-thumb { background-color: #cbd5e1; border-radius: 4px; }
::-webkit-scrollbar { display: none; }
</style>
