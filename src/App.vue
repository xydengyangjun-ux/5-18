<script setup lang="ts">
import { ref, computed, onMounted, watch, nextTick } from 'vue';
import { 
  Trophy, 
  ArrowRight, 
  ArrowLeftRight, 
  Play, 
  CheckCircle2, 
  AlertCircle,
  Award,
  ChevronRight,
  RotateCcw,
  Search,
  Rocket,
  Zap,
  ShieldAlert,
  Bot,
  Send,
  User
} from 'lucide-vue-next';
import { clsx, type ClassValue } from 'clsx';
import { twMerge } from 'tailwind-merge';

// --- Utilities ---
function cn(...inputs: ClassValue[]) {
  return twMerge(clsx(inputs));
}

// --- Types ---
type Page = 'game' | 'analysis' | 'assessment' | 'certificate';

interface Bubble {
  id: number;
  value: number;
  isLocked: boolean;
  isComparing: boolean;
}

// --- State ---
const currentPage = ref<Page>('game');
const bubbles = ref<Bubble[]>([]);
const currentIndex = ref(0);
const passCount = ref(0);
const isGameOver = ref(false);
const showRules = ref(false); // Changed to false for direct entry
const passSuccess = ref(false);
const isSwapping = ref(false);

// Analysis Mode State
const analysisStep = ref(0);
const analysisPass = ref(0);
const showLogDialog = ref(false);
const showStepDesc = ref(false);
const logInput = ref('');
const analysisLogHistory = ref<string[]>([]);
const analysisError = ref(false);
const swappedIndices = ref<number[]>([]);
const logErrorCount = ref(0);
const logErrorMessage = ref('');

// AI Chat State
interface ChatMessage {
  role: 'system' | 'user' | 'assistant';
  content: string;
}
const chatMessages = ref<ChatMessage[]>([]);
const isAiTyping = ref(false);
const chatInput = ref('');

const quickReplies = [
  '比较了相邻的两个数',
  '左边比右边大，所以交换了',
  '左边不大于右边，保持原位',
  '最大的数冒泡到了最右侧',
  '本轮没有发生任何交换',
  '比较次数比上一轮少了一次'
];

interface RoundRecord {
  round: number;
  sequence: string;
  comparisons: number;
  swaps: number;
}
const sortedHistory = ref<RoundRecord[]>([]);
const showHistoryDialog = ref(false);

// Assessment Mode State
const answers = ref<Record<string, string>>({
  q1: '', q2: '', q3: '', q4: '', q5: '',
  q6: '', q7: '', q8: '', q9: '', q10: '',
  subjective: ''
});
const assessmentResult = ref<'idle' | 'success' | 'fail'>('idle');

// --- Logic ---

const initBubbles = (count: number = 10) => {
  const values = Array.from({ length: count }, () => Math.floor(Math.random() * 90) + 10);
  bubbles.value = values.map((v, i) => ({
    id: i,
    value: v,
    isLocked: false,
    isComparing: false
  }));
  currentIndex.value = 0;
  passCount.value = 0;
  isGameOver.value = false;
  passSuccess.value = false;
};

// Page 1: Game Mode
const gameError = ref(false);

const gameHint = computed(() => {
  if (isGameOver.value) return '游戏结束！你已经掌握了冒泡排序！';
  if (passSuccess.value) return '太棒了！这一轮最大的星核已经成功“冒泡”到最右侧！';
  
  const limit = bubbles.value.length - 1 - passCount.value;
  if (currentIndex.value >= limit) return '本轮比较结束，准备进入下一轮。';

  const left = bubbles.value[currentIndex.value].value;
  const right = bubbles.value[currentIndex.value + 1].value;
  
  if (isSwapping.value) return '正在交换位置...';
  
  if (left > right) {
    return `相邻比较：左边（${left}）大于右边（${right}），请点击“交换位置”。`;
  } else {
    return `相邻比较：左边（${left}）不大于右边（${right}），无需交换，请点击“向右移动”。`;
  }
});

const handleSwap = async () => {
  const i = currentIndex.value;
  const limit = bubbles.value.length - 1 - passCount.value;
  if (i >= limit) return;

  const left = bubbles.value[i].value;
  const right = bubbles.value[i + 1].value;

  if (left > right) {
    isSwapping.value = true;
    // Small delay to let the "swapping" state trigger animations
    await new Promise(resolve => setTimeout(resolve, 50));
    
    const temp = bubbles.value[i];
    bubbles.value[i] = bubbles.value[i + 1];
    bubbles.value[i + 1] = temp;
    
    setTimeout(() => { isSwapping.value = false; }, 500);
    gameError.value = false;
  } else {
    gameError.value = true;
    setTimeout(() => { gameError.value = false; }, 500);
  }
};

const handleNext = () => {
  const limit = bubbles.value.length - 1 - passCount.value;
  if (currentIndex.value < limit - 1) {
    const left = bubbles.value[currentIndex.value].value;
    const right = bubbles.value[currentIndex.value + 1].value;
    
    // Enforce logic: must swap if left > right
    if (left > right) {
      gameError.value = true;
      setTimeout(() => { gameError.value = false; }, 500);
      return;
    }
    
    currentIndex.value++;
  } else {
    // End of pass check
    const currentPassMax = Math.max(...bubbles.value.slice(0, limit + 1).map(b => b.value));
    const lastValue = bubbles.value[limit].value;
    
    if (lastValue === currentPassMax) {
      bubbles.value[limit].isLocked = true;
      passCount.value++;
      currentIndex.value = 0;
      passSuccess.value = true;
      setTimeout(() => { passSuccess.value = false; }, 1500);

      if (passCount.value >= bubbles.value.length - 1) {
        bubbles.value[0].isLocked = true;
        isGameOver.value = true;
      }
    } else {
      alert('注意！这一轮最大的泡泡还没有到达最右侧哦，请继续调整！');
    }
  }
};

// Page 2: Analysis Mode
const currentRoundComparisons = ref(0);
const currentRoundSwaps = ref(0);
const comparisonsInput = ref('');
const swapsInput = ref('');
const showAnalysisHint = ref(false);
const aiChatType = ref<'step' | 'final'>('step');

const analysisHintText = computed(() => {
  const left = bubbles.value[currentIndex.value].value;
  const right = bubbles.value[currentIndex.value + 1].value;
  if (left > right) {
    return `提示：左边（${left}）大于右边（${right}），需要交换。`;
  } else {
    return `提示：左边（${left}）不大于右边（${right}），保持原位。`;
  }
});

const startAnalysis = () => {
  initBubbles(10); // Analysis with 10 bubbles
  analysisStep.value = 0;
  analysisPass.value = 0;
  analysisLogHistory.value = [];
  currentRoundComparisons.value = 0;
  currentRoundSwaps.value = 0;
  swappedIndices.value = [];
  showAnalysisHint.value = false;
  aiChatType.value = 'step';
  currentPage.value = 'analysis';
  
  sortedHistory.value = [{
    round: 0,
    sequence: bubbles.value.map(b => b.value).join(', '),
    comparisons: 0,
    swaps: 0
  }];
};

const checkAnalysisLogic = async (shouldSwap: boolean) => {
  const i = currentIndex.value;
  const left = bubbles.value[i].value;
  const right = bubbles.value[i + 1].value;
  const correct = left > right ? shouldSwap : !shouldSwap;

  if (correct) {
    currentRoundComparisons.value++;
    showAnalysisHint.value = false;
    if (shouldSwap) {
      currentRoundSwaps.value++;
      isSwapping.value = true;
      swappedIndices.value.push(i);
      await new Promise(resolve => setTimeout(resolve, 50));
      const temp = bubbles.value[i];
      bubbles.value[i] = bubbles.value[i + 1];
      bubbles.value[i + 1] = temp;
      setTimeout(() => { isSwapping.value = false; }, 500);
    }
    
    const limit = bubbles.value.length - 1 - analysisPass.value;
    if (currentIndex.value < limit - 1) {
      currentIndex.value++;
    } else {
      bubbles.value[limit].isLocked = true;
      showLogDialog.value = true;
    }
    analysisError.value = false;
  } else {
    analysisError.value = true;
    setTimeout(() => { analysisError.value = false; }, 500);
  }
};

const submitLog = () => {
  const expected = bubbles.value.map(b => b.value).join(',');
  const cleanedInput = logInput.value.replace(/\s/g, '').replace(/，/g, ',');
  
  const isSequenceCorrect = cleanedInput === expected;
  const isComparisonsCorrect = parseInt(comparisonsInput.value) === currentRoundComparisons.value;
  const isSwapsCorrect = parseInt(swapsInput.value) === currentRoundSwaps.value;

  if (isSequenceCorrect && isComparisonsCorrect && isSwapsCorrect) {
    sortedHistory.value.push({
      round: analysisPass.value + 1,
      sequence: bubbles.value.map(b => b.value).join(', '),
      comparisons: currentRoundComparisons.value,
      swaps: currentRoundSwaps.value
    });

    showLogDialog.value = false;
    logErrorCount.value = 0;
    logErrorMessage.value = '';
    aiChatType.value = 'step';
    showStepDesc.value = true; // Ask for description after log
  } else {
    logErrorCount.value++;
    
    if (logErrorCount.value >= 3) {
      logErrorMessage.value = `小智提示：星核顺序应该是 ${expected}，比较了 ${currentRoundComparisons.value} 次，交换了 ${currentRoundSwaps.value} 次。我已经帮你填好了，你可以直接提交！`;
      logInput.value = expected;
      comparisonsInput.value = currentRoundComparisons.value.toString();
      swapsInput.value = currentRoundSwaps.value.toString();
    } else {
      let errors = [];
      if (!isSequenceCorrect) errors.push('星核顺序');
      if (!isComparisonsCorrect) errors.push('比较次数');
      if (!isSwapsCorrect) errors.push('交换次数');
      logErrorMessage.value = `记录不正确，你的【${errors.join('、')}】填错了，请仔细核对！`;
    }
  }
};

const initChat = () => {
  if (aiChatType.value === 'step') {
    chatMessages.value = [
      { 
        role: 'assistant', 
        content: `你好！我是你的AI助教小智🤖。你刚刚完成了一轮星核排位（冒泡排序），能用自己的话描述一下，你是怎么把最高能量的星核送到右边的吗？\n\n（提示：试着用“相邻”、“比较”、“交换”等词语哦）` 
      }
    ];
  } else {
    chatMessages.value = [
      { 
        role: 'assistant', 
        content: `恭喜你完成了所有的排队任务！现在小智有两个问题想考考你：\n1. 冒泡排序每一轮分别固定了什么数？有什么特点？\n2. 从第二轮开始，排序时可以简化哪一步，要比较几次？` 
      }
    ];
  }
};

watch(showStepDesc, (newVal) => {
  if (newVal) {
    initChat();
  }
});

const sendMessageToAi = async () => {
  if (!chatInput.value.trim() || isAiTyping.value) return;
  
  const userMsg = chatInput.value.trim();
  chatMessages.value.push({ role: 'user', content: userMsg });
  chatInput.value = '';
  isAiTyping.value = true;

  await nextTick();
  const container = document.getElementById('chat-container');
  if (container) container.scrollTop = container.scrollHeight;

  try {
    const systemPromptStep = `你叫小智，是小学五年级信息技术课的AI助教。当前课程是《冒泡排序》。
教学重难点：
1. 相邻两个元素进行比较；
2. 如果左边大于右边则交换位置；
3. 一轮比较后，最大的数会像泡泡一样移动到最右侧。

你的任务是评价学生对排序步骤的描述。
严格遵守以下规则：
1. 如果学生回答与冒泡排序、比较、交换、数字等无关的内容（例如聊游戏、闲聊、乱码等），你必须严厉批评，并强制引导他们回到本节课的知识点。
2. 如果学生回答正确或部分正确，给予表扬，并引导他们说出完整的重难点（相邻比较、左大右小交换、最大数归位）。
3. 语气要符合对五年级小学生的设定，既有亲和力，在学生跑题时又有老师的严厉。
4. 语言简练，每次回复不超过100字，多用启发式提问。`;

    const systemPromptFinal = `你叫小智，是小学五年级信息技术课的AI助教。学生刚刚完成了冒泡排序的所有步骤。
你向学生提出了一个任务：请结合刚才的体验，用自己的话归纳冒泡排序的算法步骤。
标准答案应包含以下两步的核心意思：
第 1 步: 比较相邻的两个数，如果第一个比第二个大，就交换位置。对每一对相邻数进行同样的操作，从开始两个数到最后两个数。操作后，排在最后面的数就是最大数。
第 2 步: 除已排序的数，重复第 1 步的操作，对其余数进行比较与交换，直到没有任何一对数需要交换位置。

你的任务是评价学生的回答。
严格遵守以下规则：
1. 如果学生回答无关内容，严厉批评并引导回问题。
2. 针对学生的回答，指出正确的部分，补充不完整的部分。引导他们说出完整的两步。
3. 语气亲切，符合五年级设定。
4. 语言简练，不超过150字。`;

    const systemPrompt = aiChatType.value === 'final' ? systemPromptFinal : systemPromptStep;

    const messages = [
      { role: 'system', content: systemPrompt },
      ...chatMessages.value.map(m => ({ role: m.role, content: m.content }))
    ];

    const response = await fetch('https://api.deepseek.com/chat/completions', {
      method: 'POST',
      headers: {
        'Content-Type': 'application/json',
        'Authorization': 'Bearer sk-eb65e011c69a4e1cb667eecdfce990a8'
      },
      body: JSON.stringify({
        model: 'deepseek-chat',
        messages: messages,
        temperature: 0.7
      })
    });

    const data = await response.json();
    if (data.choices && data.choices[0]) {
      chatMessages.value.push({ role: 'assistant', content: data.choices[0].message.content });
    } else {
      chatMessages.value.push({ role: 'assistant', content: '抱歉，星际通信受到干扰，请再说一次。' });
    }
  } catch (error) {
    console.error('AI Chat Error:', error);
    chatMessages.value.push({ role: 'assistant', content: '网络连接出现问题，请检查网络后重试。' });
  } finally {
    isAiTyping.value = false;
    await nextTick();
    const container = document.getElementById('chat-container');
    if (container) container.scrollTop = container.scrollHeight;
  }
};

const finishAnalysisPass = () => {
  if (aiChatType.value === 'step') {
    analysisPass.value++;
    currentIndex.value = 0;
    showStepDesc.value = false;
    logInput.value = '';
    comparisonsInput.value = '';
    swapsInput.value = '';
    currentRoundComparisons.value = 0;
    currentRoundSwaps.value = 0;
    swappedIndices.value = [];
    showAnalysisHint.value = false;
    logErrorCount.value = 0;
    logErrorMessage.value = '';
    
    if (analysisPass.value >= bubbles.value.length - 1) {
      bubbles.value[0].isLocked = true;
      aiChatType.value = 'final';
      showStepDesc.value = true;
      initChat();
    }
  } else {
    showStepDesc.value = false;
    currentPage.value = 'assessment';
  }
};

// Page 3: Assessment
const assessmentQuestions = [
  { id: 'q1', title: '1. 冒泡排序的基本规则是比较什么位置的两个数？', options: [{id: 'any', t: '任意两个数'}, {id: 'adjacent', t: '相邻的两个数'}, {id: 'ends', t: '第一个和最后一个数'}] },
  { id: 'q2', title: '2. 在从小到大的冒泡排序中，什么时候需要交换两个数的位置？', options: [{id: 'greater', t: '左边的数大于右边的数'}, {id: 'less', t: '左边的数小于右边的数'}, {id: 'equal', t: '两个数相等时'}] },
  { id: 'q3', title: '3. 第一轮冒泡排序结束后，哪个数会被固定在最右侧？', options: [{id: 'min', t: '最小的数'}, {id: 'max', t: '最大的数'}, {id: 'random', t: '随机的一个数'}] },
  { id: 'q4', title: '4. 从第二轮开始，排序时可以简化哪一步？', options: [{id: 'stop', t: '不再需要比较'}, {id: 'reduce', t: '比较次数可以减少一次'}, {id: 'restart', t: '重新打乱顺序'}] },
  { id: 'q5', title: '5. 冒泡排序什么时候说明已经全部排好序了？', options: [{id: 'ten', t: '比较了10次之后'}, {id: 'right', t: '最大的数到了最右边'}, {id: 'noswap', t: '整轮比较中没有任何一对数需要交换'}] },
  { id: 'q6', title: '6. 【学习活动2】算法在计算机解决问题时起到了什么作用？', options: [{id: 'steps', t: '提供了明确的求解步骤'}, {id: 'ui', t: '提供了好看的界面'}, {id: 'storage', t: '提供了存储空间'}] },
  { id: 'q7', title: '7. 【学习活动2】什么是实现自动化和智能化的基础？', options: [{id: 'hardware', t: '鼠标和键盘'}, {id: 'algorithm', t: '算法'}, {id: 'display', t: '显示器'}] },
  { id: 'q8', title: '8. 【学习活动2】针对同一个问题，不同的算法会产生什么影响？', options: [{id: 'none', t: '没有任何影响'}, {id: 'efficiency', t: '可能产生不同的解决方案和执行效率'}, {id: 'error', t: '计算机会报错'}] },
  { id: 'q9', title: '9. 【学习活动2】选择最优的算法可以带来什么好处？', options: [{id: 'increase', t: '增加计算量'}, {id: 'optimize', t: '减少计算量、降低存储需求'}, {id: 'heavy', t: '让电脑变重'}] },
  { id: 'q10', title: '10. 【学习活动2】程序设计的主要依据和解决实际问题的策略是什么？', options: [{id: 'basis', t: '算法'}, {id: 'game', t: '游戏规则'}, {id: 'device', t: '硬件设备'}] }
];

const checkAssessment = () => {
  const correctAnswers: Record<string, string> = {
    q1: 'adjacent', q2: 'greater', q3: 'max', q4: 'reduce', q5: 'noswap',
    q6: 'steps', q7: 'algorithm', q8: 'efficiency', q9: 'optimize', q10: 'basis'
  };
  
  const allMcqCorrect = Object.keys(correctAnswers).every(k => answers.value[k] === correctAnswers[k]);
  
  const keywords = ['相邻', '比较', '交换', '最后'];
  const subjectiveCorrect = keywords.every(k => answers.value.subjective.includes(k));
  
  if (allMcqCorrect && subjectiveCorrect) {
    assessmentResult.value = 'success';
    setTimeout(() => { currentPage.value = 'certificate'; }, 1500);
  } else {
    assessmentResult.value = 'fail';
    setTimeout(() => { assessmentResult.value = 'idle'; }, 2000);
  }
};

// Lifecycle
onMounted(() => {
  initBubbles(10);
});

</script>

<template>
  <div class="min-h-screen bg-[#020617] text-slate-100 font-sans selection:bg-cyan-500/30 overflow-x-hidden overflow-y-auto relative">
    <!-- Background Effects -->
    <div class="fixed inset-0 bg-[radial-gradient(circle_at_50%_-20%,#083344,transparent)] pointer-events-none"></div>
    <div class="fixed inset-0 bg-[url('https://www.transparenttextures.com/patterns/carbon-fibre.png')] opacity-10 pointer-events-none"></div>
    
    <!-- Floating Particles -->
    <div class="fixed inset-0 pointer-events-none overflow-hidden z-0">
      <div v-for="n in 20" :key="n" 
        class="absolute rounded-full bg-cyan-500/20 blur-xl animate-pulse"
        :style="{
          width: Math.random() * 100 + 50 + 'px',
          height: Math.random() * 100 + 50 + 'px',
          left: Math.random() * 100 + '%',
          top: Math.random() * 100 + '%',
          animationDelay: Math.random() * 5 + 's',
          animationDuration: Math.random() * 10 + 10 + 's'
        }"
      ></div>
    </div>

    <!-- Main Content -->
    <main class="relative z-10 max-w-5xl mx-auto px-6 py-12 min-h-screen flex flex-col">
      
      <!-- Header -->
      <header class="flex justify-between items-center mb-12">
        <div class="flex items-center gap-3">
          <div class="p-2 bg-cyan-500/20 rounded-lg border border-cyan-500/30">
            <Rocket class="w-6 h-6 text-cyan-400" />
          </div>
          <h1 class="text-2xl font-bold tracking-tight bg-clip-text text-transparent bg-gradient-to-r from-cyan-400 to-blue-500 uppercase">
            星际能量舱：星核排位系统
          </h1>
        </div>
        <nav class="flex items-center gap-2 text-sm font-medium text-slate-400">
          <button 
            @click="currentPage = 'game'; initBubbles(10)"
            :class="cn('px-3 py-1 rounded-full border transition-all hover:bg-slate-800', currentPage === 'game' ? 'bg-cyan-500/20 border-cyan-500/50 text-cyan-300' : 'border-slate-800')"
          >
            1. 玩中学
          </button>
          <ChevronRight class="w-4 h-4" />
          <button 
            @click="startAnalysis"
            :class="cn('px-3 py-1 rounded-full border transition-all hover:bg-slate-800', currentPage === 'analysis' ? 'bg-cyan-500/20 border-cyan-500/50 text-cyan-300' : 'border-slate-800')"
          >
            2. 做中学
          </button>
          <ChevronRight class="w-4 h-4" />
          <button 
            @click="currentPage = 'assessment'"
            :class="cn('px-3 py-1 rounded-full border transition-all hover:bg-slate-800', currentPage === 'assessment' ? 'bg-cyan-500/20 border-cyan-500/50 text-cyan-300' : 'border-slate-800')"
          >
            3. 归纳提炼
          </button>
        </nav>
      </header>

      <!-- Page 1: Game Mode -->
      <section v-if="currentPage === 'game'" class="flex-1 flex flex-col items-center justify-center gap-10">
        <div class="text-center space-y-2">
          <h2 class="text-3xl font-bold text-white">星核排位赛</h2>
          <p class="text-slate-400">目标：遵循“左大右小则交换”原则，将最高能量星核送到最右侧。</p>
        </div>

        <!-- Bubble Track Container -->
        <div class="relative p-8 bg-slate-900/40 rounded-3xl border border-slate-800/50 backdrop-blur-sm shadow-2xl">
          <!-- The Magnetic Rail Base -->
          <div class="absolute bottom-6 left-8 right-8 h-1 bg-cyan-500/20 rounded-full blur-[1px]"></div>
          <div class="absolute bottom-6 left-8 right-8 h-[1px] bg-cyan-400/50 rounded-full"></div>
          
          <div class="relative flex items-center gap-6 h-24 px-4">
            <!-- Scanner Box (Tractor Beam) - Now "passes through" -->
            <div 
              class="absolute h-[140%] border-[3px] border-cyan-400 bg-cyan-400/10 rounded-2xl transition-all duration-300 ease-out shadow-[0_0_40px_rgba(34,211,238,0.4)] z-20 pointer-events-none"
              :style="{ 
                width: '136px', 
                left: (currentIndex * 72 + 8) + 'px',
                top: '-20%'
              }"
              :class="gameError && 'border-red-500 bg-red-500/20 animate-[shake_0.4s_ease-in-out]'"
            >
              <!-- Inner Beam Effect -->
              <div class="absolute inset-0 bg-gradient-to-b from-cyan-400/20 to-transparent opacity-30"></div>
              
              <div class="absolute -top-12 left-1/2 -translate-x-1/2 flex flex-col items-center">
                <Zap class="w-8 h-8 text-cyan-400 animate-pulse" :class="gameError && 'text-red-500'" />
                <span v-if="gameError" class="text-red-500 text-xs font-bold whitespace-nowrap mt-1 uppercase tracking-tighter">能量冲突！逻辑错误</span>
              </div>
            </div>

            <TransitionGroup name="bubble-list">
              <div 
                v-for="(bubble, index) in bubbles" 
                :key="bubble.id"
                class="relative z-10 shrink-0"
              >
                <div :class="cn(
                  'w-12 h-12 rounded-full flex items-center justify-center text-lg font-bold transition-all duration-500',
                  bubble.isLocked ? 'bg-gradient-to-br from-amber-400 to-orange-600 shadow-[0_0_15px_rgba(251,191,36,0.4)] border border-amber-200' : 'bg-gradient-to-br from-cyan-400/20 to-blue-600/40 border border-cyan-400/30 shadow-lg backdrop-blur-md',
                  (index === currentIndex || index === currentIndex + 1) && !bubble.isLocked && 'scale-125 ring-2 ring-cyan-400 shadow-[0_0_25px_rgba(34,211,238,0.5)]',
                  isSwapping && (index === currentIndex || index === currentIndex + 1) && 'animate-swapping'
                )">
                  <span :class="bubble.isLocked ? 'text-white' : 'text-cyan-50'">{{ bubble.value }}</span>
                  <Award v-if="bubble.isLocked" class="absolute -top-1 -right-1 w-4 h-4 text-amber-300 drop-shadow-md" />
                </div>
              </div>
            </TransitionGroup>
          </div>
        </div>

        <!-- Game Hint -->
        <div class="bg-cyan-950/50 border border-cyan-500/30 rounded-xl px-6 py-4 flex items-center gap-4 max-w-2xl shadow-[0_0_20px_rgba(34,211,238,0.1)]">
          <div class="w-10 h-10 rounded-full bg-cyan-500/20 flex items-center justify-center shrink-0 border border-cyan-500/50">
            <Bot class="w-5 h-5 text-cyan-400" />
          </div>
          <p class="text-cyan-100 font-medium text-lg leading-relaxed">
            {{ gameHint }}
          </p>
        </div>

        <!-- Controls -->
        <div class="flex gap-6 mt-4">
          <button 
            @click="handleSwap"
            :disabled="isGameOver"
            class="group px-10 py-5 bg-gradient-to-r from-cyan-600 to-blue-700 rounded-2xl font-bold text-white shadow-xl hover:shadow-cyan-500/30 hover:-translate-y-1 transition-all active:scale-95 disabled:opacity-50 disabled:pointer-events-none flex items-center gap-3"
          >
            <ArrowLeftRight class="w-6 h-6 group-hover:rotate-180 transition-transform duration-500" />
            交换位置
          </button>
          <button 
            @click="handleNext"
            :disabled="isGameOver"
            class="group px-10 py-5 bg-slate-800 border border-slate-700 rounded-2xl font-bold text-slate-200 hover:bg-slate-700 transition-all active:scale-95 disabled:opacity-50 disabled:pointer-events-none flex items-center gap-3"
          >
            向右移动
            <ArrowRight class="w-6 h-6 group-hover:translate-x-1 transition-transform" />
          </button>
        </div>

        <!-- Rules Modal -->
        <div v-if="showRules" class="fixed inset-0 z-[100] flex items-center justify-center p-6 bg-slate-950/90 backdrop-blur-md">
          <div class="bg-slate-900 border border-cyan-500/30 p-10 rounded-3xl max-w-2xl w-full space-y-8 shadow-2xl">
            <div class="text-center space-y-2">
              <h3 class="text-3xl font-bold text-white">游戏规则</h3>
              <p class="text-cyan-400 font-mono">冒泡排序：让大的数字像泡泡一样浮上去</p>
            </div>
            <ul class="space-y-4 text-slate-300 text-lg">
              <li class="flex items-start gap-3">
                <div class="w-6 h-6 rounded-full bg-cyan-500/20 flex items-center justify-center text-cyan-400 text-sm font-bold shrink-0 mt-1">1</div>
                <span>扫描仪每次框住<strong>相邻</strong>的两个泡泡。</span>
              </li>
              <li class="flex items-start gap-3">
                <div class="w-6 h-6 rounded-full bg-cyan-500/20 flex items-center justify-center text-cyan-400 text-sm font-bold shrink-0 mt-1">2</div>
                <span>如果<strong>左边比右边大</strong>，点击【交换位置】。</span>
              </li>
              <li class="flex items-start gap-3">
                <div class="w-6 h-6 rounded-full bg-cyan-500/20 flex items-center justify-center text-cyan-400 text-sm font-bold shrink-0 mt-1">3</div>
                <span>点击【向右移动】检查下一对，直到将<strong>本轮最大数</strong>送到最右侧。</span>
              </li>
            </ul>
            <button @click="showRules = false" class="w-full py-5 bg-cyan-600 hover:bg-cyan-500 text-white rounded-2xl font-bold text-xl transition-all shadow-lg">
              我准备好了，开始挑战！
            </button>
          </div>
        </div>

        <!-- Success Dialog -->
        <div v-if="isGameOver" class="fixed inset-0 z-50 flex items-center justify-center p-6 bg-slate-950/80 backdrop-blur-sm">
          <div class="bg-slate-900 border border-slate-800 p-8 rounded-3xl max-w-md w-full text-center space-y-6 shadow-2xl">
            <div class="w-20 h-20 bg-yellow-500/20 rounded-full flex items-center justify-center mx-auto border border-yellow-500/50">
              <Trophy class="w-10 h-10 text-yellow-400" />
            </div>
            <div class="space-y-2">
              <h3 class="text-2xl font-bold text-white">恭喜过关！</h3>
              <p class="text-slate-400">你发现了吗？最大的泡泡总是最先浮出水面！进入下一关，我们将拆解这个魔法。</p>
            </div>
            <button 
              @click="startAnalysis"
              class="w-full py-4 bg-cyan-600 hover:bg-cyan-500 text-white rounded-xl font-bold transition-all flex items-center justify-center gap-2"
            >
              进入：慢动作逻辑追踪
              <ChevronRight class="w-5 h-5" />
            </button>
          </div>
        </div>
      </section>

      <!-- Page 2: Analysis Mode -->
      <section v-if="currentPage === 'analysis'" class="flex-1 flex flex-col gap-8 relative pb-8">
        <!-- AI Tutor Header -->
        <div class="flex flex-col gap-4">
          <div class="bg-slate-900/50 border border-cyan-500/20 rounded-2xl p-6 flex items-start gap-4 backdrop-blur-md">
            <div class="w-12 h-12 bg-cyan-500/20 rounded-full flex items-center justify-center border border-cyan-500/30 shrink-0">
              <div class="w-6 h-6 bg-cyan-400 rounded-full animate-pulse"></div>
            </div>
            <div class="space-y-1">
              <p class="text-cyan-400 text-xs font-bold uppercase tracking-wider">AI 导师引导中</p>
              <p class="text-lg text-slate-200 font-medium">
                第 <span class="text-cyan-400">{{ analysisPass + 1 }}</span> 轮比较：请观察索引 {{ currentIndex }} 和 {{ currentIndex + 1 }} 的星核，它们需要交换吗？
              </p>
            </div>
          </div>
          
          <!-- Stats Bar -->
          <div class="flex justify-center gap-8 text-sm">
            <div class="flex items-center gap-2 bg-slate-800/50 px-4 py-2 rounded-full border border-slate-700">
              <span class="text-slate-400">本轮比较次数:</span>
              <span class="text-cyan-400 font-bold text-lg">{{ currentRoundComparisons }}</span>
            </div>
            <div class="flex items-center gap-2 bg-slate-800/50 px-4 py-2 rounded-full border border-slate-700">
              <span class="text-slate-400">本轮交换次数:</span>
              <span class="text-purple-400 font-bold text-lg">{{ currentRoundSwaps }}</span>
            </div>
          </div>
        </div>

        <div class="flex-1 flex flex-col items-center justify-center gap-12">
          <!-- Bubble Container -->
          <div 
            :class="cn(
              'relative flex items-center gap-6 h-32 px-12 transition-transform',
              analysisError && 'animate-[shake_0.5s_ease-in-out]'
            )"
          >
            <!-- Scanner Box (Tractor Beam) - Now "passes through" -->
            <div 
              class="absolute h-[140%] border-[3px] border-cyan-400 bg-cyan-400/10 rounded-2xl transition-all duration-300 ease-out shadow-[0_0_40px_rgba(34,211,238,0.4)] z-20 pointer-events-none flex justify-center"
              :style="{ 
                width: '136px', 
                left: (currentIndex * 72 + 40) + 'px',
                top: '-20%'
              }"
              :class="analysisError && 'border-red-500 bg-red-500/20 animate-[shake_0.4s_ease-in-out]'"
            >
              <!-- Hint Tooltip -->
              <div v-if="showAnalysisHint" class="absolute -top-16 bg-cyan-900/95 border border-cyan-400 text-cyan-50 px-4 py-2 rounded-xl text-sm font-medium whitespace-nowrap shadow-[0_0_20px_rgba(34,211,238,0.3)] animate-in fade-in zoom-in-95">
                {{ analysisHintText }}
                <div class="absolute -bottom-2 left-1/2 -translate-x-1/2 border-4 border-transparent border-t-cyan-400"></div>
              </div>

              <!-- Inner Beam Effect -->
              <div class="absolute inset-0 bg-gradient-to-b from-cyan-400/20 to-transparent opacity-30"></div>
              
              <div class="absolute -top-12 left-1/2 -translate-x-1/2">
                <Search class="w-8 h-8 text-cyan-400 animate-pulse" :class="analysisError && 'text-red-500'" />
              </div>
            </div>

            <!-- Swap Markers between bubbles -->
            <div class="absolute inset-0 pointer-events-none z-0">
              <div v-for="idx in swappedIndices" :key="'swap-'+idx"
                   class="absolute top-1/2 -translate-y-1/2 -translate-x-1/2 text-purple-400 bg-slate-900/80 rounded-full p-1 border border-purple-500/30 shadow-[0_0_10px_rgba(168,85,247,0.4)] animate-in zoom-in"
                   :style="{ left: (108 + idx * 72) + 'px' }">
                <ArrowLeftRight class="w-4 h-4" />
              </div>
            </div>

            <TransitionGroup name="bubble-list">
              <div 
                v-for="(bubble, index) in bubbles" 
                :key="bubble.id"
                class="relative z-10 shrink-0"
              >
                <div :class="cn(
                  'w-12 h-12 rounded-full flex items-center justify-center text-lg font-bold transition-all duration-500',
                  bubble.isLocked ? 'bg-slate-800 border border-slate-700 opacity-50' : 
                  'bg-gradient-to-br from-cyan-400/20 to-blue-600/40 border border-cyan-400/30 shadow-lg backdrop-blur-md',
                  (index === currentIndex || index === currentIndex + 1) && !bubble.isLocked && 'scale-125 ring-2 ring-cyan-400 shadow-[0_0_25px_rgba(34,211,238,0.5)]',
                  isSwapping && (index === currentIndex || index === currentIndex + 1) && 'animate-swapping'
                )">
                  <span :class="bubble.isLocked ? 'text-slate-500' : 'text-cyan-50'">{{ bubble.value }}</span>
                  <CheckCircle2 v-if="bubble.isLocked" class="absolute -top-1 -right-1 w-4 h-4 text-cyan-500" />
                </div>
              </div>
            </TransitionGroup>
          </div>

          <!-- Logic Buttons -->
          <div v-if="!showLogDialog" class="flex flex-col items-center gap-4">
            <div class="flex gap-6">
              <button 
                @click="checkAnalysisLogic(true)"
                class="px-10 py-5 bg-cyan-600 hover:bg-cyan-500 text-white rounded-2xl font-bold shadow-xl transition-all active:scale-95 flex flex-col items-center gap-1"
              >
                <ArrowLeftRight class="w-6 h-6" />
                <span>需要交换 (左 > 右)</span>
              </button>
              <button 
                @click="checkAnalysisLogic(false)"
                class="px-10 py-5 bg-slate-800 border border-slate-700 hover:bg-slate-700 text-slate-200 rounded-2xl font-bold shadow-xl transition-all active:scale-95 flex flex-col items-center gap-1"
              >
                <RotateCcw class="w-6 h-6" />
                <span>保持原位 (左 ≤ 右)</span>
              </button>
            </div>
            
            <button @click="showAnalysisHint = true" class="text-cyan-400 hover:text-cyan-300 text-sm flex items-center gap-1 transition-colors">
              <AlertCircle class="w-4 h-4" /> 需要提示吗？
            </button>
          </div>

          <!-- Log Dialog (Replaces Logic Buttons when active) -->
          <div v-if="showLogDialog" class="w-full max-w-3xl mx-auto z-50 animate-in zoom-in-95">
            <div class="bg-slate-900 border-2 border-cyan-500/50 p-6 rounded-3xl space-y-4 shadow-2xl backdrop-blur-xl">
              <div class="flex justify-between items-center">
                <h3 class="text-xl font-bold text-white flex items-center gap-2">
                  <Award class="text-cyan-400 w-5 h-5" />
                  第 {{ analysisPass + 1 }} 轮结束：请记录状态
                </h3>
                <p class="text-xs text-slate-400">请观察上方泡泡的数值并按顺序输入</p>
              </div>
              
              <div class="grid grid-cols-2 gap-4">
                <div class="col-span-2 space-y-1">
                  <label class="text-xs text-slate-400">本轮最终星核排列顺序（用逗号隔开）：</label>
                  <input 
                    v-model="logInput"
                    type="text"
                    placeholder="例如: 10,20,30..."
                    class="w-full bg-slate-950 border border-slate-700 rounded-xl px-4 py-3 text-lg font-mono text-cyan-400 focus:outline-none focus:ring-2 focus:ring-cyan-500/50"
                  />
                </div>
                <div class="space-y-1">
                  <label class="text-xs text-slate-400">本轮比较次数：</label>
                  <input 
                    v-model="comparisonsInput"
                    type="number"
                    placeholder="输入次数"
                    class="w-full bg-slate-950 border border-slate-700 rounded-xl px-4 py-3 text-lg font-mono text-cyan-400 focus:outline-none focus:ring-2 focus:ring-cyan-500/50"
                  />
                </div>
                <div class="space-y-1">
                  <label class="text-xs text-slate-400">本轮交换次数：</label>
                  <input 
                    v-model="swapsInput"
                    type="number"
                    placeholder="输入次数"
                    class="w-full bg-slate-950 border border-slate-700 rounded-xl px-4 py-3 text-lg font-mono text-cyan-400 focus:outline-none focus:ring-2 focus:ring-cyan-500/50"
                    @keyup.enter="submitLog"
                  />
                </div>
              </div>
              <div v-if="logErrorMessage" class="p-3 bg-red-500/20 border border-red-500/50 rounded-xl text-red-400 text-sm">
                {{ logErrorMessage }}
              </div>
              <div class="flex justify-end pt-2 gap-4">
                <button 
                  @click="showHistoryDialog = true"
                  class="px-8 py-3 bg-slate-800 border border-slate-700 hover:bg-slate-700 text-slate-200 rounded-xl font-bold transition-all"
                >
                  已排序轮次记录
                </button>
                <button 
                  @click="submitLog"
                  class="px-8 py-3 bg-cyan-600 hover:bg-cyan-500 text-white rounded-xl font-bold transition-all"
                >
                  确认记录
                </button>
              </div>
            </div>
          </div>
        </div>

        <!-- Step Description Dialog -->
        <div v-if="showStepDesc" class="fixed inset-0 z-50 flex items-center justify-center p-6 bg-slate-950/80 backdrop-blur-sm">
          <div class="bg-slate-900 border border-cyan-500/30 rounded-3xl max-w-2xl w-full h-[600px] flex flex-col shadow-[0_0_50px_rgba(34,211,238,0.15)] overflow-hidden">
            <!-- Chat Header -->
            <div class="px-6 py-4 bg-slate-800/50 border-b border-slate-700/50 flex items-center justify-between">
              <div class="flex items-center gap-3">
                <div class="w-10 h-10 bg-cyan-500/20 rounded-full flex items-center justify-center border border-cyan-500/50">
                  <Bot class="w-6 h-6 text-cyan-400" />
                </div>
                <div>
                  <h3 class="text-lg font-bold text-white">AI 助教 小智</h3>
                  <p class="text-xs text-cyan-400">在线指导中...</p>
                </div>
              </div>
              <button @click="finishAnalysisPass" class="px-4 py-2 bg-cyan-600 hover:bg-cyan-500 text-white text-sm rounded-lg font-bold transition-all">
                {{ aiChatType === 'step' ? '结束对话并进入下一轮' : '结束对话并进入归纳提炼' }}
              </button>
            </div>

            <!-- Chat Messages Area -->
            <div class="flex-1 overflow-y-auto p-6 space-y-6 scroll-smooth" id="chat-container">
              <div 
                v-for="(msg, idx) in chatMessages.filter(m => m.role !== 'system')" 
                :key="idx"
                :class="cn('flex gap-4 max-w-[85%]', msg.role === 'user' ? 'ml-auto flex-row-reverse' : '')"
              >
                <div :class="cn('w-8 h-8 rounded-full flex items-center justify-center shrink-0 mt-1', msg.role === 'user' ? 'bg-blue-500/20 border border-blue-500/50' : 'bg-cyan-500/20 border border-cyan-500/50')">
                  <User v-if="msg.role === 'user'" class="w-5 h-5 text-blue-400" />
                  <Bot v-else class="w-5 h-5 text-cyan-400" />
                </div>
                <div :class="cn('p-4 rounded-2xl whitespace-pre-wrap text-sm leading-relaxed', msg.role === 'user' ? 'bg-blue-600 text-white rounded-tr-sm' : 'bg-slate-800 border border-slate-700 text-slate-200 rounded-tl-sm')">
                  {{ msg.content }}
                </div>
              </div>
              <div v-if="isAiTyping" class="flex gap-4 max-w-[85%]">
                <div class="w-8 h-8 bg-cyan-500/20 border border-cyan-500/50 rounded-full flex items-center justify-center shrink-0 mt-1">
                  <Bot class="w-5 h-5 text-cyan-400" />
                </div>
                <div class="p-4 rounded-2xl bg-slate-800 border border-slate-700 text-slate-400 rounded-tl-sm flex items-center gap-2">
                  <div class="w-2 h-2 bg-cyan-400 rounded-full animate-bounce"></div>
                  <div class="w-2 h-2 bg-cyan-400 rounded-full animate-bounce" style="animation-delay: 0.2s"></div>
                  <div class="w-2 h-2 bg-cyan-400 rounded-full animate-bounce" style="animation-delay: 0.4s"></div>
                </div>
              </div>
            </div>

            <!-- Chat Input Area -->
            <div class="p-4 bg-slate-800/50 border-t border-slate-700/50 flex flex-col gap-3">
              <!-- Quick Replies -->
              <div class="flex flex-wrap gap-2">
                <button 
                  v-for="reply in quickReplies" 
                  :key="reply"
                  @click="chatInput += (chatInput ? '，' : '') + reply"
                  class="px-3 py-1.5 bg-slate-800 border border-slate-700 hover:border-cyan-500/50 text-slate-300 hover:text-cyan-300 text-xs rounded-lg transition-colors text-left"
                >
                  {{ reply }}
                </button>
              </div>
              <form @submit.prevent="sendMessageToAi" class="flex gap-3">
                <input 
                  v-model="chatInput"
                  type="text" 
                  placeholder="告诉小智你是怎么排序的..."
                  class="flex-1 bg-slate-900 border border-slate-700 rounded-xl px-4 py-3 text-white focus:outline-none focus:border-cyan-500 focus:ring-1 focus:ring-cyan-500 transition-all"
                  :disabled="isAiTyping"
                />
                <button 
                  type="submit"
                  :disabled="!chatInput.trim() || isAiTyping"
                  class="px-6 py-3 bg-cyan-600 hover:bg-cyan-500 disabled:opacity-50 disabled:cursor-not-allowed text-white rounded-xl font-bold transition-all flex items-center gap-2"
                >
                  发送
                  <Send class="w-4 h-4" />
                </button>
              </form>
            </div>
          </div>
        </div>

        <!-- Final Transition -->
        <div v-if="analysisPass >= bubbles.length - 1" class="fixed inset-0 z-50 flex items-center justify-center p-6 bg-slate-950/80 backdrop-blur-sm">
          <div class="bg-slate-900 border border-slate-800 p-8 rounded-3xl max-w-md w-full text-center space-y-6 shadow-2xl">
            <div class="w-20 h-20 bg-cyan-500/20 rounded-full flex items-center justify-center mx-auto border border-cyan-500/50">
              <CheckCircle2 class="w-10 h-10 text-cyan-400" />
            </div>
            <div class="space-y-2">
              <h3 class="text-2xl font-bold text-white">逻辑追踪完成！</h3>
              <p class="text-slate-400">你已经掌握了冒泡排序的严谨步骤。现在，准备好接受“算法工程师”的认证了吗？</p>
            </div>
            <button 
              @click="currentPage = 'assessment'"
              class="w-full py-4 bg-gradient-to-r from-cyan-600 to-blue-600 text-white rounded-xl font-bold transition-all flex items-center justify-center gap-2"
            >
              进入考核
              <ChevronRight class="w-5 h-5" />
            </button>
          </div>
        </div>

        <!-- History Dialog -->
        <div v-if="showHistoryDialog" class="fixed inset-0 z-[100] flex items-center justify-center p-6 bg-slate-950/80 backdrop-blur-sm">
          <div class="bg-slate-900 border border-cyan-500/30 p-8 rounded-3xl max-w-2xl w-full max-h-[80vh] flex flex-col shadow-2xl">
            <div class="flex justify-between items-center mb-6">
              <h3 class="text-2xl font-bold text-white flex items-center gap-2">
                <RotateCcw class="w-6 h-6 text-cyan-400" />
                已排序轮次记录
              </h3>
              <button @click="showHistoryDialog = false" class="text-slate-400 hover:text-white transition-colors">
                关闭
              </button>
            </div>
            <div class="flex-1 overflow-y-auto space-y-4 pr-2 custom-scrollbar">
              <div v-for="record in sortedHistory" :key="record.round" class="bg-slate-800/50 border border-slate-700 rounded-xl p-4">
                <div class="flex justify-between items-center mb-2">
                  <span class="text-cyan-400 font-bold">第 {{ record.round }} 轮</span>
                  <div class="text-sm text-slate-400 flex gap-4">
                    <span>比较: <span class="text-cyan-300">{{ record.comparisons }}</span> 次</span>
                    <span>交换: <span class="text-purple-300">{{ record.swaps }}</span> 次</span>
                  </div>
                </div>
                <div class="font-mono text-lg text-slate-200 tracking-widest break-all">
                  {{ record.sequence }}
                </div>
              </div>
            </div>
          </div>
        </div>
      </section>

      <!-- Page 3: Assessment Mode -->
      <section v-if="currentPage === 'assessment'" class="flex-1 overflow-y-auto pr-2 custom-scrollbar">
        <div class="max-w-3xl mx-auto space-y-12 pb-12">
          <div class="text-center space-y-2">
            <h2 class="text-3xl font-bold text-white">算法工程师认证</h2>
            <p class="text-slate-400">总结你的发现，完成最后的挑战</p>
          </div>

          <!-- Objective Questions -->
          <div class="space-y-8">
            <div v-for="q in assessmentQuestions" :key="q.id" class="bg-slate-900/50 border border-slate-800 p-6 rounded-2xl space-y-4">
              <p class="text-lg font-medium text-slate-200">{{ q.title }}</p>
              <div class="grid grid-cols-1 gap-3">
                <label v-for="opt in q.options" :key="opt.id"
                  class="flex items-center gap-3 p-4 rounded-xl border border-slate-800 cursor-pointer hover:bg-slate-800/50 transition-all"
                  :class="answers[q.id] === opt.id && 'border-cyan-500/50 bg-cyan-500/5 text-cyan-400'"
                >
                  <input type="radio" v-model="answers[q.id]" :value="opt.id" class="hidden" />
                  <div class="w-5 h-5 rounded-full border-2 border-slate-700 flex items-center justify-center" :class="answers[q.id] === opt.id && 'border-cyan-500'">
                    <div v-if="answers[q.id] === opt.id" class="w-2.5 h-2.5 rounded-full bg-cyan-500"></div>
                  </div>
                  <span>{{ opt.t }}</span>
                </label>
              </div>
            </div>
          </div>

          <!-- Subjective Question -->
          <div class="bg-slate-900/50 border border-slate-800 p-8 rounded-3xl space-y-6">
            <div class="flex items-start gap-4">
              <div class="w-12 h-12 bg-cyan-500/20 rounded-full flex items-center justify-center border border-cyan-500/30 shrink-0">
                <Play class="w-6 h-6 text-cyan-400" />
              </div>
              <div class="space-y-1">
                <p class="text-xl font-bold text-white">最后的挑战：学习活动2</p>
                <p class="text-slate-400">请结合刚才的“做中学”体验，用自然语言归纳冒泡排序的算法步骤，并说明每一轮比较次数的变化规律。</p>
              </div>
            </div>
            
            <textarea 
              v-model="answers.subjective"
              rows="5"
              placeholder="例如：1. 将相邻的两个数进行比较，如果左边比右边大就交换，直到最大的数排到最后。2. 第二轮比较时，不需要再和最后一个数比较，比较次数减少1次..."
              class="w-full bg-slate-950 border border-slate-800 rounded-2xl p-6 text-slate-200 focus:outline-none focus:ring-2 focus:ring-cyan-500/50 transition-all resize-none"
            ></textarea>
          </div>

          <!-- Submit -->
          <div class="flex flex-col items-center gap-4">
            <button 
              @click="checkAssessment"
              class="px-12 py-5 bg-gradient-to-r from-cyan-600 to-blue-600 hover:from-cyan-500 hover:to-blue-500 text-white rounded-2xl font-bold text-xl shadow-2xl transition-all active:scale-95 flex items-center gap-3"
            >
              提交认证申请
              <ChevronRight class="w-6 h-6" />
            </button>
            <p v-if="assessmentResult === 'fail'" class="text-red-400 font-medium animate-pulse">认证未通过，请检查选项或完善描述！</p>
            <p v-if="assessmentResult === 'success'" class="text-green-400 font-bold flex items-center gap-2">
              <CheckCircle2 class="w-5 h-5" />
              认证成功！正在生成证书...
            </p>
          </div>
        </div>
      </section>

      <!-- Page 4: Certificate -->
      <section v-if="currentPage === 'certificate'" class="flex-1 flex items-center justify-center p-6">
        <div class="relative max-w-2xl w-full aspect-[1.414/1] bg-slate-900 border-8 border-double border-cyan-500/30 p-12 overflow-hidden shadow-[0_0_100px_rgba(6,182,212,0.1)]">
          <!-- Decorative Corners -->
          <div v-for="pos in ['top-0 left-0', 'top-0 right-0', 'bottom-0 left-0', 'bottom-0 right-0']" :key="pos" 
            :class="cn('absolute w-16 h-16 border-cyan-500 border-4', pos.includes('top') ? 'border-b-0' : 'border-t-0', pos.includes('left') ? 'border-r-0' : 'border-l-0')"></div>
          
          <!-- Watermark -->
          <Trophy class="absolute top-1/2 left-1/2 -translate-x-1/2 -translate-y-1/2 w-64 h-64 text-cyan-500/5" />

          <div class="relative z-10 h-full flex flex-col items-center justify-between text-center">
            <div class="space-y-2">
              <h2 class="text-4xl font-black tracking-widest text-white uppercase">算法工程师认证证书</h2>
              <div class="h-1 w-32 bg-gradient-to-r from-transparent via-cyan-500 to-transparent mx-auto"></div>
            </div>

            <div class="space-y-6">
              <p class="text-slate-400 italic">兹证明</p>
              <p class="text-5xl font-bold text-cyan-400 underline underline-offset-8 decoration-cyan-500/30">优秀的小学五年级同学</p>
              <p class="text-xl text-slate-300 leading-relaxed max-w-md">
                在《校园跳绳榜：冒泡排序泡泡舱》挑战中，成功掌握了冒泡排序的核心逻辑，具备了将动作经验转化为算法思维的能力。
              </p>
            </div>

            <div class="w-full flex justify-between items-end">
              <div class="text-left space-y-1">
                <p class="text-xs text-slate-500 uppercase tracking-tighter">认证编号</p>
                <p class="font-mono text-cyan-500/70">ALG-2026-{{ Math.floor(Math.random()*100000) }}</p>
              </div>
              <div class="flex flex-col items-center gap-2">
                <div class="w-20 h-20 rounded-full border-4 border-cyan-500/20 flex items-center justify-center">
                  <Award class="w-10 h-10 text-cyan-500" />
                </div>
                <p class="text-sm font-bold text-slate-400">深海算法实验室</p>
              </div>
            </div>
          </div>
        </div>
      </section>

    </main>

    <!-- Footer Info -->
    <footer class="fixed bottom-6 right-6 z-20 text-slate-500 text-xs flex items-center gap-4">
      <div class="flex items-center gap-1">
        <div class="w-2 h-2 rounded-full bg-green-500"></div>
        系统运行正常
      </div>
      <span>v1.0.4 - Deep Sea Tech</span>
    </footer>
  </div>
</template>

<style>
@keyframes shake {
  0%, 100% { transform: translateX(0); }
  25% { transform: translateX(-10px); }
  75% { transform: translateX(10px); }
}

@keyframes swapping {
  0%, 100% { transform: scale(1.25); }
  50% { transform: scale(1.4) rotate(10deg); filter: brightness(1.5); }
}

.animate-swapping {
  animation: swapping 0.5s ease-in-out;
}

.bubble-list-move,
.bubble-list-enter-active,
.bubble-list-leave-active {
  transition: all 0.6s cubic-bezier(0.34, 1.56, 0.64, 1);
}

.bubble-list-enter-from,
.bubble-list-leave-to {
  opacity: 0;
  transform: scale(0.5);
}

.custom-scrollbar::-webkit-scrollbar {
  width: 6px;
}
.custom-scrollbar::-webkit-scrollbar-track {
  background: transparent;
}
.custom-scrollbar::-webkit-scrollbar-thumb {
  background: #1e293b;
  border-radius: 10px;
}
.custom-scrollbar::-webkit-scrollbar-thumb:hover {
  background: #334155;
}
</style>
