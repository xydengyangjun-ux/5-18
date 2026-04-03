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
type Page = 'game' | 'analysis' | 'assessment' | 'certificate' | 'extension';

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
const isAnalysisComplete = ref(false);

// State Preservation
const gameSavedState = ref<any>(null);
const analysisSavedState = ref<any>(null);

// AI Tutor Q1 & Q2 State
const showGameTutorQ1 = ref(false);
const gameTutorQ1Answered = ref(false);
const q1SelectedText = ref('');
const q1Countdown = ref(5);
let q1Timer: any = null;

const showGameTutorQ2 = ref(false);
const gameTutorQ2Answered = ref(false);
const q2SelectedText = ref('');
const q2Countdown = ref(5);
let q2Timer: any = null;
const showGoToAnalysisBtn = ref(false);

const checkQ1 = () => {
  if (q1SelectedText.value === '最大数') {
    gameTutorQ1Answered.value = true;
    q1Countdown.value = 5;
    q1Timer = setInterval(() => {
      q1Countdown.value--;
      if (q1Countdown.value <= 0) {
        clearInterval(q1Timer);
        showGameTutorQ1.value = false;
      }
    }, 1000);
  } else {
    q1SelectedText.value = '不对哦，再想想';
    setTimeout(() => {
      if (q1SelectedText.value === '不对哦，再想想') {
        q1SelectedText.value = '';
      }
    }, 1500);
  }
};

const checkQ2 = () => {
  if (q2SelectedText.value === '减少一次') {
    gameTutorQ2Answered.value = true;
    q2Countdown.value = 5;
    q2Timer = setInterval(() => {
      q2Countdown.value--;
      if (q2Countdown.value <= 0) {
        clearInterval(q2Timer);
        showGoToAnalysisBtn.value = true;
      }
    }, 1000);
  } else {
    q2SelectedText.value = '不对哦，再想想';
    setTimeout(() => {
      if (q2SelectedText.value === '不对哦，再想想') {
        q2SelectedText.value = '';
      }
    }, 1500);
  }
};

const extensionSavedState = ref<any>(null);

const saveCurrentState = () => {
  const state = {
    bubbles: JSON.parse(JSON.stringify(bubbles.value)),
    currentIndex: currentIndex.value,
    passCount: passCount.value,
    isGameOver: isGameOver.value,
    passSuccess: passSuccess.value,
    showGameTutorQ1: showGameTutorQ1.value,
    gameTutorQ1Answered: gameTutorQ1Answered.value,
    q1SelectedText: q1SelectedText.value,
    showGameTutorQ2: showGameTutorQ2.value,
    gameTutorQ2Answered: gameTutorQ2Answered.value,
    q2SelectedText: q2SelectedText.value,
    showGoToAnalysisBtn: showGoToAnalysisBtn.value,
    analysisStep: analysisStep.value,
    analysisPass: analysisPass.value,
    showLogDialog: showLogDialog.value,
    showStepDesc: showStepDesc.value,
    logInput: logInput.value,
    analysisLogHistory: [...analysisLogHistory.value],
    analysisError: analysisError.value,
    swappedIndices: [...swappedIndices.value],
    logErrorCount: logErrorCount.value,
    logErrorMessage: logErrorMessage.value,
    sortedHistory: JSON.parse(JSON.stringify(sortedHistory.value)),
    isAnalysisComplete: isAnalysisComplete.value,
  };
  if (currentPage.value === 'game') gameSavedState.value = state;
  if (currentPage.value === 'analysis') analysisSavedState.value = state;
  
  if (currentPage.value === 'extension') {
    extensionSavedState.value = {
      extensionData: [...extensionData.value],
      extensionCorrectRounds: JSON.parse(JSON.stringify(extensionCorrectRounds.value)),
      extensionUserInputs: JSON.parse(JSON.stringify(extensionUserInputs.value)),
      extensionCurrentRound: extensionCurrentRound.value,
      extensionCompleted: extensionCompleted.value,
      extensionBlanks: { ...extensionBlanks.value },
      extensionBlanksResult: extensionBlanksResult.value
    };
  }
};

const restoreState = (page: Page) => {
  if (page === 'extension') {
    const state = extensionSavedState.value;
    if (state) {
      extensionData.value = state.extensionData;
      extensionCorrectRounds.value = state.extensionCorrectRounds;
      extensionUserInputs.value = state.extensionUserInputs;
      extensionCurrentRound.value = state.extensionCurrentRound;
      extensionCompleted.value = state.extensionCompleted;
      extensionBlanks.value = state.extensionBlanks;
      extensionBlanksResult.value = state.extensionBlanksResult;
      return true;
    }
    return false;
  }

  const state = page === 'game' ? gameSavedState.value : analysisSavedState.value;
  if (state) {
    bubbles.value = state.bubbles;
    currentIndex.value = state.currentIndex;
    passCount.value = state.passCount;
    isGameOver.value = state.isGameOver;
    passSuccess.value = state.passSuccess;
    showGameTutorQ1.value = state.showGameTutorQ1 || false;
    gameTutorQ1Answered.value = state.gameTutorQ1Answered || false;
    q1SelectedText.value = state.q1SelectedText || '';
    showGameTutorQ2.value = state.showGameTutorQ2 || false;
    gameTutorQ2Answered.value = state.gameTutorQ2Answered || false;
    q2SelectedText.value = state.q2SelectedText || '';
    showGoToAnalysisBtn.value = state.showGoToAnalysisBtn || false;
    analysisStep.value = state.analysisStep;
    analysisPass.value = state.analysisPass;
    showLogDialog.value = state.showLogDialog;
    showStepDesc.value = state.showStepDesc;
    logInput.value = state.logInput;
    analysisLogHistory.value = state.analysisLogHistory;
    analysisError.value = state.analysisError;
    swappedIndices.value = state.swappedIndices;
    logErrorCount.value = state.logErrorCount;
    logErrorMessage.value = state.logErrorMessage;
    sortedHistory.value = state.sortedHistory;
    isAnalysisComplete.value = state.isAnalysisComplete;
    return true;
  }
  return false;
};

const switchPage = (page: Page) => {
  if (currentPage.value === page) return;
  saveCurrentState();
  currentPage.value = page;
  
  if (page === 'game') {
    if (!restoreState('game')) {
      initBubbles(10);
    }
  } else if (page === 'analysis') {
    if (!restoreState('analysis')) {
      startAnalysis();
    }
  } else if (page === 'extension') {
    if (!restoreState('extension')) {
      initExtension();
    }
  }
};

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

let q1ShowTimer: any = null;
let q2ShowTimer: any = null;

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
  
  showGameTutorQ1.value = false;
  gameTutorQ1Answered.value = false;
  q1SelectedText.value = '';
  if (q1Timer) clearInterval(q1Timer);
  if (q1ShowTimer) clearTimeout(q1ShowTimer);
  
  showGameTutorQ2.value = false;
  gameTutorQ2Answered.value = false;
  q2SelectedText.value = '';
  showGoToAnalysisBtn.value = false;
  if (q2Timer) clearInterval(q2Timer);
  if (q2ShowTimer) clearTimeout(q2ShowTimer);
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

      if (passCount.value === 1) {
        q1ShowTimer = setTimeout(() => { showGameTutorQ1.value = true; }, 1500);
      }

      if (passCount.value >= bubbles.value.length - 1) {
        bubbles.value[0].isLocked = true;
        isGameOver.value = true;
        q2ShowTimer = setTimeout(() => { showGameTutorQ2.value = true; }, 1500);
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
  initBubbles(8); // Analysis with 8 bubbles
  analysisStep.value = 0;
  analysisPass.value = 0;
  analysisLogHistory.value = [];
  currentRoundComparisons.value = 0;
  currentRoundSwaps.value = 0;
  swappedIndices.value = [];
  showAnalysisHint.value = false;
  aiChatType.value = 'step';
  isAnalysisComplete.value = false;
  
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
    
    if (analysisPass.value >= bubbles.value.length - 2) {
      bubbles.value[0].isLocked = true;
      aiChatType.value = 'final';
      showStepDesc.value = true;
      initChat();
    } else {
      analysisPass.value++;
      currentIndex.value = 0;
      logInput.value = '';
      comparisonsInput.value = '';
      swapsInput.value = '';
      currentRoundComparisons.value = 0;
      currentRoundSwaps.value = 0;
      swappedIndices.value = [];
      showAnalysisHint.value = false;
    }
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
  if (aiChatType.value === 'final') {
    showStepDesc.value = false;
    isAnalysisComplete.value = true;
    showHistoryDialog.value = true;
  }
};

// Page 3: Assessment
const assessmentChatMessages = ref<ChatMessage[]>([
  {
    role: 'assistant',
    content: '你好！我是你的AI助教小智🤖。你已经完成了冒泡排序的所有步骤，现在请结合刚才的体验，用自然语言归纳一下冒泡排序的算法步骤吧！'
  }
]);
const assessmentChatInput = ref('');
const isAssessmentAiTyping = ref(false);
const finalChallengePassed = ref(false);

const sendAssessmentMessage = async () => {
  if (!assessmentChatInput.value.trim() || isAssessmentAiTyping.value) return;
  
  const userMsg = assessmentChatInput.value.trim();
  assessmentChatMessages.value.push({ role: 'user', content: userMsg });
  assessmentChatInput.value = '';
  isAssessmentAiTyping.value = true;

  await nextTick();
  const container = document.getElementById('assessment-chat-container');
  if (container) container.scrollTop = container.scrollHeight;

  try {
    const systemPromptFinal = `你叫小智，是小学五年级信息技术课的AI助教。学生刚刚完成了冒泡排序的所有步骤。
你向学生提出了一个任务：请结合刚才的体验，用自己的话归纳冒泡排序的算法步骤。
标准答案应包含以下两步的核心意思：
第 1 步: 比较相邻的两个数，如果第一个比第二个大，就交换位置。对每一对相邻数进行同样的操作，从开始两个数到最后两个数。操作后，排在最后面的数就是最大数。
第 2 步: 除已排序的数，重复第 1 步的操作，对其余数进行比较与交换，直到没有任何一对数需要交换位置。

你的任务是评价学生的回答。
严格遵守以下规则：
1. 如果学生回答无关内容，严厉批评并引导回问题。
2. 针对学生的回答，指出正确的部分，补充不完整的部分。引导他们说出完整的两步。
3. 如果学生已经完整、正确地归纳了冒泡排序的步骤，你必须在回复的最后加上 [PASS] 这个标记。
4. 语气亲切，符合五年级设定。
5. 语言简练，不超过150字。`;

    const messages = [
      { role: 'system', content: systemPromptFinal },
      ...assessmentChatMessages.value.map(m => ({ role: m.role, content: m.content }))
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
    let reply = data.choices[0].message.content;
    
    if (reply.includes('[PASS]')) {
      finalChallengePassed.value = true;
      reply = reply.replace('[PASS]', '').trim();
    }

    assessmentChatMessages.value.push({ role: 'assistant', content: reply });
  } catch (error) {
    console.error('AI Chat Error:', error);
    assessmentChatMessages.value.push({ role: 'assistant', content: '网络连接出现问题，请检查网络后重试。' });
  } finally {
    isAssessmentAiTyping.value = false;
    await nextTick();
    const container = document.getElementById('assessment-chat-container');
    if (container) container.scrollTop = container.scrollHeight;
  }
};

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
  
  if (allMcqCorrect && finalChallengePassed.value) {
    assessmentResult.value = 'success';
    setTimeout(() => { currentPage.value = 'certificate'; }, 1500);
  } else {
    assessmentResult.value = 'fail';
    setTimeout(() => { assessmentResult.value = 'idle'; }, 2000);
  }
};

// Extension Task State
const extensionData = ref<number[]>([]);
const extensionCorrectRounds = ref<number[][]>([]);
const extensionUserInputs = ref<string[][]>([]);
const extensionCurrentRound = ref(0);
const extensionCompleted = ref(false);

const extensionBlanks = ref({
  b1: '',
  b2: '',
  b3: '',
  b4: '',
  b5: '',
  b6: ''
});
const extensionBlanksResult = ref<'idle' | 'success' | 'fail'>('idle');
const extensionBlanksErrors = ref<string[]>([]);

const extensionErrorDetails = ref({
  show: false,
  round: 0,
  wrongIndices: [] as number[],
  processSteps: [] as { sequence: number[], swapped: number[], comparing: number[] }[],
  canClose: false
});

const initExtension = () => {
  extensionData.value = Array.from({ length: 6 }, () => Math.floor(Math.random() * 120) + 80);
  
  let currentArr = [...extensionData.value];
  extensionCorrectRounds.value = [];
  for (let i = 0; i < 5; i++) {
    for (let j = 0; j < 5 - i; j++) {
      if (currentArr[j] > currentArr[j + 1]) {
        let temp = currentArr[j];
        currentArr[j] = currentArr[j + 1];
        currentArr[j + 1] = temp;
      }
    }
    extensionCorrectRounds.value.push([...currentArr]);
  }
  
  extensionUserInputs.value = Array.from({ length: 5 }, () => Array(6).fill(''));
  extensionCurrentRound.value = 0;
  extensionCompleted.value = false;
  
  extensionBlanks.value = { b1: '', b2: '', b3: '', b4: '', b5: '', b6: '' };
  extensionBlanksResult.value = 'idle';
  extensionErrorDetails.value.show = false;
};

const checkExtensionRound = (roundIndex: number) => {
  const userArr = extensionUserInputs.value[roundIndex].map(v => parseInt(v) || 0);
  const correctArr = extensionCorrectRounds.value[roundIndex];
  
  const wrongIndices: number[] = [];
  userArr.forEach((v, i) => {
    if (v !== correctArr[i]) wrongIndices.push(i);
  });
  
  if (wrongIndices.length === 0) {
    if (roundIndex < 4) {
      extensionCurrentRound.value++;
      // Auto-fill previously sorted for the next round
      for (let i = 5 - roundIndex; i < 6; i++) {
        extensionUserInputs.value[roundIndex + 1][i] = extensionUserInputs.value[roundIndex][i];
      }
    } else {
      extensionCompleted.value = true;
    }
  } else {
    // Generate process steps for this round
    const startArr = roundIndex === 0 ? [...extensionData.value] : [...extensionCorrectRounds.value[roundIndex - 1]];
    const processSteps = [];
    let currentArr = [...startArr];
    processSteps.push({ sequence: [...currentArr], swapped: [], comparing: [] });

    for (let j = 0; j < 5 - roundIndex; j++) {
      const comparing = [j, j + 1];
      if (currentArr[j] > currentArr[j + 1]) {
        let temp = currentArr[j];
        currentArr[j] = currentArr[j + 1];
        currentArr[j + 1] = temp;
        processSteps.push({ sequence: [...currentArr], swapped: [...comparing], comparing: [...comparing] });
      } else {
        processSteps.push({ sequence: [...currentArr], swapped: [], comparing: [...comparing] });
      }
    }

    extensionErrorDetails.value = {
      show: true,
      round: roundIndex + 1,
      wrongIndices,
      processSteps,
      canClose: false
    };

    setTimeout(() => {
      extensionErrorDetails.value.canClose = true;
    }, 3000);
  }
};

const checkExtensionBlanks = () => {
  const correct = {
    b1: '相邻',
    b2: '大',
    b3: '交换',
    b4: '最大数',
    b5: '已排序',
    b6: '交换'
  };
  
  extensionBlanksErrors.value = [];
  if (extensionBlanks.value.b1 !== correct.b1) extensionBlanksErrors.value.push('b1');
  if (extensionBlanks.value.b2 !== correct.b2) extensionBlanksErrors.value.push('b2');
  if (extensionBlanks.value.b3 !== correct.b3) extensionBlanksErrors.value.push('b3');
  if (extensionBlanks.value.b4 !== correct.b4) extensionBlanksErrors.value.push('b4');
  if (extensionBlanks.value.b5 !== correct.b5) extensionBlanksErrors.value.push('b5');
  if (extensionBlanks.value.b6 !== correct.b6) extensionBlanksErrors.value.push('b6');
    
  if (extensionBlanksErrors.value.length === 0) {
    extensionBlanksResult.value = 'success';
  } else {
    extensionBlanksResult.value = 'fail';
    setTimeout(() => { extensionBlanksResult.value = 'idle'; }, 5000);
  }
};

watch(extensionBlanks, () => {
  if (extensionBlanksResult.value === 'fail') {
    extensionBlanksResult.value = 'idle';
    extensionBlanksErrors.value = [];
  }
}, { deep: true });

// Lifecycle
onMounted(() => {
  initBubbles(10);
  initExtension();
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
            @click="switchPage('game')"
            :class="cn('px-3 py-1 rounded-full border transition-all hover:bg-slate-800', currentPage === 'game' ? 'bg-cyan-500/20 border-cyan-500/50 text-cyan-300' : 'border-slate-800')"
          >
            1. 玩中学
          </button>
          <ChevronRight class="w-4 h-4" />
          <button 
            @click="switchPage('analysis')"
            :class="cn('px-3 py-1 rounded-full border transition-all hover:bg-slate-800', currentPage === 'analysis' ? 'bg-cyan-500/20 border-cyan-500/50 text-cyan-300' : 'border-slate-800')"
          >
            2. 做中学
          </button>
          <ChevronRight class="w-4 h-4" />
          <button 
            @click="switchPage('extension')"
            :class="cn('px-3 py-1 rounded-full border transition-all', 
              currentPage === 'extension' ? 'bg-cyan-500/20 border-cyan-500/50 text-cyan-300' : 'border-slate-800 hover:bg-slate-800'
            )"
          >
            3. 拓展任务
          </button>
          <ChevronRight class="w-4 h-4" />
          <button 
            @click="extensionBlanksResult === 'success' ? switchPage('assessment') : null"
            :class="cn('px-3 py-1 rounded-full border transition-all', 
              currentPage === 'assessment' ? 'bg-cyan-500/20 border-cyan-500/50 text-cyan-300' : 'border-slate-800',
              extensionBlanksResult !== 'success' ? 'opacity-50 cursor-not-allowed text-slate-500' : 'hover:bg-slate-800'
            )"
            :title="extensionBlanksResult !== 'success' ? '请先完成拓展任务' : ''"
          >
            4. 归纳提炼
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
            @click="initBubbles(10)"
            class="px-8 py-5 bg-slate-800 hover:bg-slate-700 border border-slate-700 rounded-2xl font-bold text-slate-300 transition-all active:scale-95 flex items-center gap-3"
          >
            <RotateCcw class="w-6 h-6" />
            重新开始
          </button>
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

        <!-- AI Tutor Q1 Modal -->
        <div v-if="showGameTutorQ1" class="fixed inset-0 z-[100] flex items-center justify-center p-6 bg-slate-950/90 backdrop-blur-md">
          <div class="bg-slate-900 border border-cyan-500/30 p-10 rounded-3xl max-w-2xl w-full space-y-8 shadow-2xl relative">
            <div class="flex items-center gap-4 mb-6">
              <div class="w-12 h-12 bg-cyan-500/20 rounded-full flex items-center justify-center border border-cyan-500/50 shrink-0">
                <Bot class="w-7 h-7 text-cyan-400" />
              </div>
              <div>
                <h3 class="text-2xl font-bold text-white">AI 导师小智</h3>
                <p class="text-cyan-400">观察与思考</p>
              </div>
            </div>
            
            <div class="text-xl text-slate-200 leading-relaxed flex flex-wrap items-center gap-2">
              <span>每一轮“冒泡”后，都会固定当前未排序数中</span>
              <span class="inline-block min-w-[100px] h-10 border-b-2 border-cyan-500 text-center text-cyan-400 font-bold px-4">
                {{ q1SelectedText }}
              </span>
              <span>的位置。</span>
            </div>
            
            <div class="flex gap-4 mt-6" v-if="!gameTutorQ1Answered">
              <button @click="q1SelectedText = '最小数'; checkQ1()" class="px-6 py-3 rounded-xl border border-slate-700 hover:bg-slate-800 text-slate-300 transition-colors">最小数</button>
              <button @click="q1SelectedText = '最大数'; checkQ1()" class="px-6 py-3 rounded-xl border border-slate-700 hover:bg-slate-800 text-slate-300 transition-colors">最大数</button>
              <button @click="q1SelectedText = '中间数'; checkQ1()" class="px-6 py-3 rounded-xl border border-slate-700 hover:bg-slate-800 text-slate-300 transition-colors">中间数</button>
            </div>
            
            <div v-else class="p-6 bg-green-500/20 border border-green-500/50 rounded-xl text-green-400 flex flex-col items-center gap-3">
              <CheckCircle2 class="w-10 h-10" />
              <p class="text-lg font-bold">回答正确！</p>
              <p class="text-sm">每一轮冒泡都会把当前最大的数固定在最右侧。窗口将在 {{ q1Countdown }} 秒后关闭，请继续排序...</p>
            </div>
          </div>
        </div>

        <!-- AI Tutor Q2 Modal -->
        <div v-if="showGameTutorQ2" class="fixed inset-0 z-[100] flex items-center justify-center p-6 bg-slate-950/90 backdrop-blur-md">
          <div class="bg-slate-900 border border-cyan-500/30 p-10 rounded-3xl max-w-2xl w-full space-y-8 shadow-2xl relative">
            <div class="flex items-center gap-4 mb-6">
              <div class="w-12 h-12 bg-cyan-500/20 rounded-full flex items-center justify-center border border-cyan-500/50 shrink-0">
                <Bot class="w-7 h-7 text-cyan-400" />
              </div>
              <div>
                <h3 class="text-2xl font-bold text-white">AI 导师小智</h3>
                <p class="text-cyan-400">总结与发现</p>
              </div>
            </div>
            
            <div class="text-xl text-slate-200 leading-relaxed flex flex-wrap items-center gap-2">
              <span>第一轮已经将最大数固定在最右侧，比较的次数可以</span>
              <span class="inline-block min-w-[120px] h-10 border-b-2 border-cyan-500 text-center text-cyan-400 font-bold px-4">
                {{ q2SelectedText }}
              </span>
              <span>，即已经排好位置的最大数，不必参与后续的比较。</span>
            </div>
            
            <div class="flex gap-4 mt-6" v-if="!gameTutorQ2Answered">
              <button @click="q2SelectedText = '增加一次'; checkQ2()" class="px-6 py-3 rounded-xl border border-slate-700 hover:bg-slate-800 text-slate-300 transition-colors">增加一次</button>
              <button @click="q2SelectedText = '保持不变'; checkQ2()" class="px-6 py-3 rounded-xl border border-slate-700 hover:bg-slate-800 text-slate-300 transition-colors">保持不变</button>
              <button @click="q2SelectedText = '减少一次'; checkQ2()" class="px-6 py-3 rounded-xl border border-slate-700 hover:bg-slate-800 text-slate-300 transition-colors">减少一次</button>
            </div>
            
            <div v-else class="p-6 bg-green-500/20 border border-green-500/50 rounded-xl text-green-400 flex flex-col items-center gap-3">
              <CheckCircle2 class="w-10 h-10" />
              <p class="text-lg font-bold">回答正确！</p>
              <p class="text-sm">这既是对每轮找到最大数意义的回应，又引出了后几轮需要处理的关键问题。{{ showGoToAnalysisBtn ? '' : `（${q2Countdown} 秒后进入下一关）` }}</p>
            </div>

            <button 
              v-if="showGoToAnalysisBtn"
              @click="startAnalysis"
              class="w-full mt-4 py-4 bg-cyan-600 hover:bg-cyan-500 text-white rounded-xl font-bold transition-all flex items-center justify-center gap-2 animate-in fade-in slide-in-from-bottom-4"
            >
              进入做中学：慢动作逻辑追踪
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
          <div class="flex justify-center gap-8 text-sm items-center">
            <div class="flex items-center gap-2 bg-slate-800/50 px-4 py-2 rounded-full border border-slate-700">
              <span class="text-slate-400">本轮比较次数:</span>
              <span class="text-cyan-400 font-bold text-lg">{{ currentRoundComparisons }}</span>
            </div>
            <div class="flex items-center gap-2 bg-slate-800/50 px-4 py-2 rounded-full border border-slate-700">
              <span class="text-slate-400">本轮交换次数:</span>
              <span class="text-purple-400 font-bold text-lg">{{ currentRoundSwaps }}</span>
            </div>
            <button 
              @click="startAnalysis"
              class="flex items-center gap-2 bg-slate-800 hover:bg-slate-700 px-4 py-2 rounded-full border border-slate-700 text-slate-300 transition-colors"
            >
              <RotateCcw class="w-4 h-4" />
              重新开始
            </button>
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
                {{ aiChatType === 'step' ? '结束对话并进入下一轮' : '结束对话并查看排序记录' }}
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
                <div class="font-mono text-lg flex flex-wrap gap-2 mt-2">
                  <span 
                    v-for="(num, idx) in record.sequence.split(', ')" 
                    :key="idx"
                    :class="cn(
                      'px-2 py-1 rounded flex items-center justify-center min-w-[2.5rem]',
                      record.round > 0 && idx >= (record.round === bubbles.length - 1 ? 0 : bubbles.length - record.round)
                        ? 'border border-red-500 text-red-400 bg-red-500/10'
                        : 'border border-transparent text-slate-200 bg-slate-900/50'
                    )"
                  >
                    {{ num }}
                  </span>
                </div>
              </div>
            </div>
            <div v-if="isAnalysisComplete" class="mt-6 flex justify-center">
              <button 
                @click="switchPage('extension'); showHistoryDialog = false"
                class="w-full py-4 bg-gradient-to-r from-cyan-600 to-blue-600 text-white rounded-xl font-bold transition-all flex items-center justify-center gap-2"
              >
                进入拓展任务
                <ChevronRight class="w-5 h-5" />
              </button>
            </div>
          </div>
        </div>
      </section>

      <!-- Page 4: Assessment Mode -->
      <section v-if="currentPage === 'assessment'" class="flex-1 flex gap-6 p-6">
        <!-- Left 70%: Questions -->
        <div class="w-[70%] space-y-8 pb-12">
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

          <!-- Submit -->
          <div class="flex flex-col items-center gap-4">
            <button 
              @click="checkAssessment"
              :disabled="!finalChallengePassed"
              class="px-12 py-5 bg-gradient-to-r from-cyan-600 to-blue-600 hover:from-cyan-500 hover:to-blue-500 disabled:opacity-50 disabled:cursor-not-allowed text-white rounded-2xl font-bold text-xl shadow-2xl transition-all active:scale-95 flex items-center gap-3"
            >
              提交认证申请
              <ChevronRight class="w-6 h-6" />
            </button>
            <p v-if="!finalChallengePassed" class="text-sm text-amber-400">请先在右侧与小智完成“最后的挑战”</p>
            <p v-if="assessmentResult === 'fail'" class="text-red-400 font-medium animate-pulse">选择题有错误，请仔细检查！</p>
            <p v-if="assessmentResult === 'success'" class="text-green-400 font-bold flex items-center gap-2">
              <CheckCircle2 class="w-5 h-5" />
              认证成功！正在生成证书...
            </p>
          </div>
        </div>

        <!-- Right 30%: AI Chat -->
        <div class="w-[30%] bg-slate-900/50 border border-slate-800 rounded-3xl flex flex-col overflow-hidden sticky top-6 h-[calc(100vh-48px)] self-start">
          <!-- Chat Header -->
          <div class="p-4 bg-slate-800/80 border-b border-slate-700/50 flex items-center gap-3">
            <div class="w-10 h-10 bg-cyan-500/20 rounded-full flex items-center justify-center border border-cyan-500/50 shrink-0">
              <Bot class="w-6 h-6 text-cyan-400" />
            </div>
            <div>
              <h3 class="text-lg font-bold text-white">最后的挑战</h3>
              <p class="text-xs text-slate-400">用自然语言归纳冒泡排序</p>
            </div>
            <div v-if="finalChallengePassed" class="ml-auto flex items-center gap-1 text-green-400 bg-green-500/10 px-2 py-1 rounded-full text-xs font-bold border border-green-500/30">
              <CheckCircle2 class="w-3 h-3" />
              已通过
            </div>
          </div>
          
          <!-- Chat Messages -->
          <div id="assessment-chat-container" class="flex-1 overflow-y-auto p-4 space-y-4 custom-scrollbar">
            <div 
              v-for="(msg, idx) in assessmentChatMessages" 
              :key="idx"
              :class="cn('flex gap-3 max-w-[90%]', msg.role === 'user' ? 'ml-auto flex-row-reverse' : '')"
            >
              <div v-if="msg.role === 'assistant'" class="w-8 h-8 bg-cyan-500/20 border border-cyan-500/50 rounded-full flex items-center justify-center shrink-0 mt-1">
                <Bot class="w-5 h-5 text-cyan-400" />
              </div>
              <div v-else class="w-8 h-8 bg-blue-500/20 border border-blue-500/50 rounded-full flex items-center justify-center shrink-0 mt-1">
                <User class="w-5 h-5 text-blue-400" />
              </div>
              <div :class="cn('p-3 rounded-2xl whitespace-pre-wrap text-sm leading-relaxed', msg.role === 'user' ? 'bg-blue-600 text-white rounded-tr-sm' : 'bg-slate-800 border border-slate-700 text-slate-200 rounded-tl-sm')">
                {{ msg.content }}
              </div>
            </div>
            <div v-if="isAssessmentAiTyping" class="flex gap-3 max-w-[90%]">
              <div class="w-8 h-8 bg-cyan-500/20 border border-cyan-500/50 rounded-full flex items-center justify-center shrink-0 mt-1">
                <Bot class="w-5 h-5 text-cyan-400" />
              </div>
              <div class="p-3 rounded-2xl bg-slate-800 border border-slate-700 text-slate-400 rounded-tl-sm flex items-center gap-2">
                <div class="w-2 h-2 bg-cyan-400 rounded-full animate-bounce"></div>
                <div class="w-2 h-2 bg-cyan-400 rounded-full animate-bounce" style="animation-delay: 0.2s"></div>
                <div class="w-2 h-2 bg-cyan-400 rounded-full animate-bounce" style="animation-delay: 0.4s"></div>
              </div>
            </div>
          </div>
          
          <!-- Chat Input -->
          <div class="p-4 bg-slate-800/50 border-t border-slate-700/50">
            <form @submit.prevent="sendAssessmentMessage" class="flex gap-2">
              <input 
                v-model="assessmentChatInput"
                type="text" 
                placeholder="告诉小智你的答案..."
                class="flex-1 bg-slate-900 border border-slate-700 rounded-xl px-3 py-2 text-white text-sm focus:outline-none focus:border-cyan-500 focus:ring-1 focus:ring-cyan-500 transition-all"
                :disabled="isAssessmentAiTyping || finalChallengePassed"
              />
              <button 
                type="submit"
                :disabled="!assessmentChatInput.trim() || isAssessmentAiTyping || finalChallengePassed"
                class="px-4 py-2 bg-cyan-600 hover:bg-cyan-500 disabled:opacity-50 disabled:cursor-not-allowed text-white rounded-xl font-bold transition-all flex items-center justify-center"
              >
                <Send class="w-4 h-4" />
              </button>
            </form>
          </div>
        </div>
      </section>

      <!-- Page 5: Certificate -->
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

      <!-- Page 3: Extension Task -->
      <section v-if="currentPage === 'extension'" class="flex-1 flex flex-col gap-8 relative pb-12">
        <div class="text-center space-y-2">
          <h2 class="text-3xl font-bold text-white">拓展任务：跳绳数据比一比</h2>
          <p class="text-slate-400">随机抽取了6位同学的1分钟跳绳成绩，请你用冒泡排序帮他们排个序吧！</p>
        </div>

        <div class="bg-slate-900/50 border border-slate-800 rounded-3xl p-8 shadow-xl">
          <div class="flex items-center gap-4 mb-8">
            <h3 class="text-xl font-bold text-cyan-400">初始数据：</h3>
            <div class="flex gap-3">
              <div v-for="(num, idx) in extensionData" :key="'init-'+idx" class="w-14 h-14 bg-slate-800 border border-slate-700 rounded-xl flex items-center justify-center text-xl font-bold text-slate-200">
                {{ num }}
              </div>
            </div>
            <button @click="initExtension" class="ml-auto px-4 py-2 bg-slate-800 hover:bg-slate-700 border border-slate-700 rounded-xl text-sm text-slate-300 transition-colors flex items-center gap-2">
              <RotateCcw class="w-4 h-4" />
              重新生成数据
            </button>
          </div>

          <div class="space-y-6">
            <div v-for="round in 5" :key="'ext-round-'+round" class="relative">
              <div :class="cn('p-6 rounded-2xl border transition-all', extensionCurrentRound >= round - 1 ? 'bg-slate-800/40 border-slate-700' : 'bg-slate-900/20 border-slate-800/50 opacity-50 pointer-events-none')">
                <div class="flex items-center gap-6">
                  <div class="w-24 text-lg font-bold text-slate-300">第 {{ round }} 轮</div>
                  
                  <div class="flex gap-3 items-center">
                    <!-- Unsorted Box -->
                    <div class="flex gap-3 p-2 rounded-xl border-2 border-dashed border-slate-600 bg-slate-900/30 relative">
                      <span class="absolute -top-3 left-3 bg-slate-900 px-2 text-xs text-slate-400">未排序区</span>
                      <input 
                        v-for="i in (6 - round)" 
                        :key="'unsorted-'+i"
                        v-model="extensionUserInputs[round - 1][i - 1]"
                        type="text"
                        class="w-12 h-12 bg-slate-950 border border-slate-700 rounded-lg text-center text-lg font-bold text-cyan-300 focus:outline-none focus:border-cyan-500 focus:ring-1 focus:ring-cyan-500"
                        :disabled="extensionCurrentRound !== round - 1"
                      />
                    </div>
                    
                    <!-- Newly Sorted Box (Red) -->
                    <div class="relative">
                      <span class="absolute -top-5 left-1/2 -translate-x-1/2 text-xs text-red-400 whitespace-nowrap">本轮归位</span>
                      <input 
                        v-model="extensionUserInputs[round - 1][6 - round]"
                        type="text"
                        class="w-12 h-12 bg-red-950/30 border-2 border-red-500 rounded-lg text-center text-lg font-bold text-red-400 focus:outline-none focus:border-red-400 focus:ring-1 focus:ring-red-400 shadow-[0_0_10px_rgba(239,68,68,0.2)]"
                        :disabled="extensionCurrentRound !== round - 1"
                      />
                    </div>

                    <!-- Previously Sorted Boxes -->
                    <div v-if="round > 1" class="flex gap-3 p-2 rounded-xl border-2 border-slate-700 bg-slate-800/50 relative">
                      <span class="absolute -top-3 left-3 bg-slate-800 px-2 text-xs text-slate-400">已排序区</span>
                      <input 
                        v-for="i in (round - 1)" 
                        :key="'sorted-'+i"
                        v-model="extensionUserInputs[round - 1][6 - round + i]"
                        type="text"
                        class="w-12 h-12 bg-slate-900 border border-slate-700 rounded-lg text-center text-lg font-bold text-slate-500 cursor-not-allowed"
                        disabled
                      />
                    </div>
                  </div>

                  <div class="ml-auto flex items-center gap-4">
                    <button 
                      v-if="extensionCurrentRound === round - 1"
                      @click="checkExtensionRound(round - 1)"
                      class="px-6 py-2 bg-cyan-600 hover:bg-cyan-500 text-white rounded-xl font-bold transition-all"
                    >
                      验证
                    </button>
                    <div v-else-if="extensionCurrentRound > round - 1" class="flex items-center gap-2 text-green-400 font-bold">
                      <CheckCircle2 class="w-5 h-5" />
                      正确
                    </div>
                  </div>
                </div>
              </div>
            </div>
          </div>
        </div>

        <!-- Extension Error Modal -->
        <div v-if="extensionErrorDetails.show" class="fixed inset-0 z-[100] flex items-center justify-center p-6 bg-slate-950/90 backdrop-blur-md">
          <div class="bg-slate-900 border border-red-500/50 p-8 rounded-3xl max-w-3xl w-full shadow-2xl flex flex-col gap-6 animate-in zoom-in-95">
            <div class="flex items-center gap-4 text-red-400">
              <AlertCircle class="w-8 h-8" />
              <h3 class="text-2xl font-bold">填写有误</h3>
            </div>

            <div class="bg-slate-950 rounded-2xl p-6 border border-slate-800 space-y-4">
              <p class="text-slate-300">
                你在第 <span class="text-cyan-400 font-bold">{{ extensionErrorDetails.round }}</span> 轮中，
                第 <span class="text-red-400 font-bold">{{ extensionErrorDetails.wrongIndices.map(i => i + 1).join(', ') }}</span> 个框填写错误。
              </p>
              <p class="text-slate-400 text-sm">下面是本轮正确的冒泡交换过程：</p>

              <div class="space-y-3 mt-4 max-h-[40vh] overflow-y-auto custom-scrollbar pr-2">
                <div v-for="(step, idx) in extensionErrorDetails.processSteps" :key="'step-'+idx" class="flex items-center gap-4">
                  <span class="text-slate-500 text-sm w-16">{{ idx === 0 ? '初始' : `比较 ${idx}` }}</span>
                  <div class="flex gap-2">
                    <span
                      v-for="(num, nIdx) in step.sequence"
                      :key="'num-'+nIdx"
                      :class="cn(
                        'w-10 h-10 flex items-center justify-center rounded-lg font-bold text-sm transition-all',
                        step.swapped.includes(nIdx) ? 'bg-purple-500/20 text-purple-300 border border-purple-500/50' : 
                        (step.comparing.includes(nIdx) ? 'bg-cyan-500/10 text-cyan-300 border border-cyan-500/30' : 'bg-slate-800 text-slate-300 border border-slate-700')
                      )"
                    >
                      {{ num }}
                    </span>
                  </div>
                  <span v-if="step.swapped.length > 0" class="text-purple-400 text-xs ml-2">发生交换</span>
                  <span v-else-if="idx > 0" class="text-slate-500 text-xs ml-2">无需交换</span>
                </div>
              </div>
            </div>

            <div class="flex justify-end mt-4">
              <button
                @click="extensionErrorDetails.show = false"
                :disabled="!extensionErrorDetails.canClose"
                :class="cn(
                  'px-8 py-3 rounded-xl font-bold transition-all flex items-center gap-2',
                  extensionErrorDetails.canClose
                    ? 'bg-cyan-600 hover:bg-cyan-500 text-white shadow-lg active:scale-95'
                    : 'bg-slate-800 text-slate-500 cursor-not-allowed'
                )"
              >
                继续填写 {{ !extensionErrorDetails.canClose ? '(3s)' : '' }}
              </button>
            </div>
          </div>
        </div>

        <!-- Fill in the blanks -->
        <div v-if="extensionCompleted" class="bg-slate-900/50 border border-cyan-500/30 rounded-3xl p-8 shadow-xl animate-in fade-in slide-in-from-bottom-8">
          <div class="flex items-center gap-4 mb-6">
            <div class="w-12 h-12 bg-cyan-500/20 rounded-full flex items-center justify-center border border-cyan-500/50 shrink-0">
              <Bot class="w-7 h-7 text-cyan-400" />
            </div>
            <div>
              <h3 class="text-2xl font-bold text-white">终极总结</h3>
              <p class="text-cyan-400">用自然语言描述冒泡排序算法，你来帮我挖空</p>
            </div>
          </div>

          <div class="space-y-6 text-lg text-slate-300 leading-loose bg-slate-950/50 p-6 rounded-2xl border border-slate-800">
            <p>
              <span class="font-bold text-cyan-400">第 1 步：</span>比较
              <input v-model="extensionBlanks.b1" type="text" :class="cn('w-20 bg-slate-800 border-b-2 text-center focus:outline-none focus:bg-slate-700 px-2 mx-1 transition-colors', extensionBlanksErrors.includes('b1') ? 'border-red-500 text-red-400' : 'border-cyan-500 text-cyan-300')" placeholder="???" />
              的两个数，如果第一个比第二个
              <input v-model="extensionBlanks.b2" type="text" :class="cn('w-16 bg-slate-800 border-b-2 text-center focus:outline-none focus:bg-slate-700 px-2 mx-1 transition-colors', extensionBlanksErrors.includes('b2') ? 'border-red-500 text-red-400' : 'border-cyan-500 text-cyan-300')" placeholder="???" />
              ，就
              <input v-model="extensionBlanks.b3" type="text" :class="cn('w-20 bg-slate-800 border-b-2 text-center focus:outline-none focus:bg-slate-700 px-2 mx-1 transition-colors', extensionBlanksErrors.includes('b3') ? 'border-red-500 text-red-400' : 'border-cyan-500 text-cyan-300')" placeholder="???" />
              位置。对每一对相邻数进行同样的操作，从开始两个数到最后两个数。操作后，排在最后面的数就是
              <input v-model="extensionBlanks.b4" type="text" :class="cn('w-24 bg-slate-800 border-b-2 text-center focus:outline-none focus:bg-slate-700 px-2 mx-1 transition-colors', extensionBlanksErrors.includes('b4') ? 'border-red-500 text-red-400' : 'border-cyan-500 text-cyan-300')" placeholder="???" />
              。
            </p>
            <p>
              <span class="font-bold text-cyan-400">第 2 步：</span>除
              <input v-model="extensionBlanks.b5" type="text" :class="cn('w-24 bg-slate-800 border-b-2 text-center focus:outline-none focus:bg-slate-700 px-2 mx-1 transition-colors', extensionBlanksErrors.includes('b5') ? 'border-red-500 text-red-400' : 'border-cyan-500 text-cyan-300')" placeholder="???" />
              的数，重复第 1 步的操作，对其余数进行比较与交换，直到没有任何一对数需要
              <input v-model="extensionBlanks.b6" type="text" :class="cn('w-20 bg-slate-800 border-b-2 text-center focus:outline-none focus:bg-slate-700 px-2 mx-1 transition-colors', extensionBlanksErrors.includes('b6') ? 'border-red-500 text-red-400' : 'border-cyan-500 text-cyan-300')" placeholder="???" />
              位置。
            </p>
          </div>

          <div class="mt-8 flex flex-col items-center gap-4">
            <button 
              @click="checkExtensionBlanks"
              class="px-10 py-4 bg-gradient-to-r from-cyan-600 to-blue-600 hover:from-cyan-500 hover:to-blue-500 text-white rounded-xl font-bold text-lg shadow-lg transition-all active:scale-95 flex items-center gap-2"
            >
              提交总结
              <Send class="w-5 h-5" />
            </button>
            <div v-if="extensionBlanksResult === 'fail'" class="text-red-400 font-medium flex flex-col items-center gap-2 animate-in zoom-in">
              <p class="flex items-center gap-2"><AlertCircle class="w-5 h-5" /> 标红的空填得不对哦！</p>
              <p class="text-sm text-slate-400">提示：相邻、大、交换、最大数、已排序、交换</p>
            </div>
            <div v-if="extensionBlanksResult === 'success'" class="text-green-400 font-bold flex flex-col items-center gap-2 animate-in zoom-in">
              <div class="flex items-center gap-2 text-xl">
                <CheckCircle2 class="w-6 h-6" />
                太棒了！你已经完全掌握了冒泡排序！
              </div>
              <button 
                @click="switchPage('assessment')"
                class="mt-4 px-8 py-3 bg-cyan-600 hover:bg-cyan-500 text-white rounded-xl font-bold shadow-lg transition-all active:scale-95 flex items-center gap-2"
              >
                进入归纳提炼
                <ChevronRight class="w-5 h-5" />
              </button>
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
