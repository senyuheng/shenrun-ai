<template>
  <view class="page">
    <view class="topbar">
      <view class="brand">AI公考Copilot</view>
      <view class="setting" @click="goSettings">设置</view>
    </view>

    <scroll-view scroll-y class="body">
      <view v-if="activeTab === 'home'" class="home-grid">
        <view
          v-for="card in cards"
          :key="card.key"
          class="home-card"
          @click="openFeature(card.key)"
        >
          <view class="home-card-title">{{ card.name }}</view>
          <view class="home-card-desc">{{ card.desc }}</view>
        </view>
      </view>

      <view v-if="activeTab === 'policyList'" class="policy-list">
        <button class="btn-secondary" @click="activeTab = 'home'; activeNav = 'home'">返回首页</button>
        <button class="btn-primary" @click="policyUpdateOpen = !policyUpdateOpen">更新今日时政</button>
        <view v-if="policyUpdateOpen" class="panel inline-panel">
          <view class="label">粘贴当天新闻 / 政策材料</view>
          <textarea v-model="policyUpdateText" class="textarea" placeholder="把新闻、政策原文或材料要点粘贴到这里..." maxlength="-1" />
          <button class="btn-primary" @click="submitPolicyUpdate" :disabled="loading || !policyUpdateText.trim()">
            {{ loading && loadingType === 'policyUpdate' ? '生成中...' : 'AI生成时政考点' }}
          </button>
        </view>
        <view
          v-for="item in allPolicyNews"
          :key="item.id"
          class="policy-card"
          @click="openPolicyDetail(item)"
        >
          <view class="policy-title">{{ item.title }}</view>
          <view class="policy-meta">
            <text>{{ item.date }}</text>
            <text class="policy-tag">{{ item.tag }}</text>
          </view>
          <view class="policy-summary">{{ item.summary }}</view>
          <view class="policy-more">查看详情</view>
        </view>
      </view>

      <view v-if="activeTab === 'policyDetail'" class="panel">
        <button class="btn-secondary" @click="activeTab = 'policyList'; activeNav = 'news'">返回列表</button>
        <view class="placeholder-title">{{ selectedPolicy.title }}</view>
        <view class="label">背景概括</view>
        <view class="detail-text">{{ selectedPolicy.background }}</view>
        <view class="label">政策意义</view>
        <view class="detail-text">{{ selectedPolicy.significance }}</view>
        <view class="label">申论可用角度</view>
        <view class="detail-text">{{ selectedPolicy.essayAngle }}</view>
        <view class="label">面试可能问法</view>
        <view class="detail-text">{{ selectedPolicy.interviewQuestion }}</view>
        <view class="label">可背诵金句</view>
        <view class="detail-text">{{ selectedPolicy.quote }}</view>
        <view class="label">100字速记版</view>
        <view class="detail-text">{{ selectedPolicy.memory }}</view>
        <button class="btn-secondary" @click="favoritePolicy(selectedPolicy)">收藏</button>
        <button class="btn-primary" @click="openPodcastFromPolicy(selectedPolicy)">生成双人播客</button>
        <button class="btn-secondary" @click="openInterviewFromPolicy(selectedPolicy)">去面试模拟</button>
        <button class="btn-secondary" @click="openEssayFromPolicy(selectedPolicy)">用作申论素材</button>
      </view>

      <view v-if="activeTab === 'mine'" class="my-page">
        <view class="my-section">
          <view class="my-title">我的批改记录</view>
          <view v-if="!essayRecords.length" class="my-empty">暂无申论批改记录。</view>
          <view v-for="item in essayRecords" :key="item.id" class="policy-card">
            <view class="policy-title">{{ item.topic || '申论批改' }}</view>
            <view class="policy-meta">
              <text>{{ formatDate(item.ts) }}</text>
              <text class="policy-tag">总分 {{ item.score || '—' }}</text>
            </view>
            <button class="btn-secondary" @click="openEssayRecord(item)">查看详情</button>
          </view>
        </view>
        <view class="my-section">
          <view class="my-title">我的面试记录</view>
          <view v-if="!interviewRecords.length" class="my-empty">暂无面试记录。</view>
          <view v-for="item in interviewRecords" :key="item.id" class="policy-card">
            <view class="policy-title">{{ item.question }}</view>
            <view class="policy-meta">
              <text>{{ formatDate(item.ts) }}</text>
              <text class="policy-tag">{{ item.jobType }}</text>
            </view>
            <view class="policy-summary">{{ item.reviewText }}</view>
          </view>
        </view>
        <view class="my-section">
          <view class="my-title">我的收藏时政</view>
          <view v-if="!favoritePolicies.length" class="my-empty">暂无收藏时政。</view>
          <view v-for="item in favoritePolicies" :key="item.id" class="policy-card" @click="openPolicyDetail(item)">
            <view class="policy-title">{{ item.title }}</view>
            <view class="policy-meta">
              <text>{{ item.date }}</text>
              <text class="policy-tag">{{ item.tag }}</text>
            </view>
            <view class="policy-summary">{{ item.summary }}</view>
          </view>
        </view>
      </view>

      <view v-if="activeTab === 'essayRecord'" class="panel">
        <button class="btn-secondary" @click="openNav('mine')">返回我的</button>
        <view class="placeholder-title">{{ selectedRecord.topic || '申论批改详情' }}</view>
        <view class="label">总分</view>
        <view class="detail-text">{{ selectedRecord.score || '—' }}</view>
        <view class="label">用户答案</view>
        <view class="detail-text">{{ selectedRecord.answer }}</view>
        <view class="label">AI批改结果</view>
        <view class="detail-text">{{ selectedRecord.reviewText }}</view>
        <view class="label">AI改写结果</view>
        <view class="detail-text">{{ selectedRecord.rewriteText || '暂无改写结果' }}</view>
      </view>

      <view v-if="activeTab === 'essay'" class="panel">
        <button class="btn-secondary" @click="activeTab = 'home'">返回首页</button>
        <view class="label">题目</view>
        <view class="upload-row">
          <button class="btn-secondary compact" @click="chooseEssayFile('topic')" :disabled="loading">上传题目文件</button>
          <button class="btn-secondary compact" @click="chooseEssayImage('topic')" :disabled="loading">拍照识别题目</button>
        </view>
        <textarea v-model="essayTopic" class="textarea small" placeholder="请输入申论题目或作答要求..." maxlength="-1" />
        <view class="label">材料</view>
        <view class="upload-row">
          <button class="btn-secondary compact" @click="chooseEssayFile('material')" :disabled="loading">上传材料文件</button>
          <button class="btn-secondary compact" @click="chooseEssayImage('material')" :disabled="loading">拍照识别材料</button>
        </view>
        <textarea v-model="essayMaterial" class="textarea" placeholder="请输入或粘贴题目材料..." maxlength="-1" />
        <view class="label">申论作答</view>
        <button class="btn-secondary" @click="chooseEssayImage('answer')" :disabled="loading">
          {{ loading && loadingType === 'ocr' ? '识别中...' : '上传手写答案照片识别' }}
        </button>
        <textarea v-model="essay" class="textarea tall" placeholder="在此输入你的申论作答..." maxlength="-1" />
        <button class="btn-primary" @click="submitEssay" :disabled="loading || !essayTopic.trim() || !essayMaterial.trim() || !essay.trim()">
          {{ loading && loadingType === 'essay' ? '批改中...' : '提交批改' }}
        </button>
      </view>

      <view v-if="activeTab === 'interview'" class="panel">
        <button class="btn-secondary" @click="activeTab = 'home'; activeNav = 'home'">返回首页</button>
        <view class="label">岗位类型</view>
        <picker :range="jobTypes" :value="jobIndex" @change="jobIndex = Number($event.detail.value)">
          <view class="picker">{{ jobTypes[jobIndex] }}</view>
        </picker>
        <button class="btn-primary" @click="createQuestion" :disabled="loading">
          {{ loading && loadingType === 'question' ? '生成中...' : '生成题目' }}
        </button>
        <view class="label">面试题</view>
        <view class="question">{{ interviewQuestion || '请先选择岗位并生成一道结构化面试题' }}</view>
        <view class="label">你的回答</view>
        <textarea v-model="interviewAnswer" class="textarea" placeholder="在此输入你的回答..." maxlength="-1" />
        <button class="btn-primary" @click="submitInterview" :disabled="loading || !interviewQuestion || !interviewAnswer.trim()">
          {{ loading && loadingType === 'interview' ? '点评中...' : 'AI点评' }}
        </button>
      </view>

      <view v-if="activeTab === 'podcast'" class="panel">
        <button class="btn-secondary" @click="activeTab = 'home'; activeNav = 'home'">返回首页</button>
        <view class="label">选择时政</view>
        <picker :range="allPolicyNews" range-key="title" :value="podcastPolicyIndex" @change="changePodcastPolicy">
          <view class="picker">{{ activePodcastPolicy.title }}</view>
        </picker>
        <view class="detail-text">
          {{ activePodcastPolicy.title }}
          {{ activePodcastPolicy.summary }}
          政策意义：{{ activePodcastPolicy.significance }}
        </view>
        <button class="btn-primary" @click="submitPodcast" :disabled="loading">
          {{ loading && loadingType === 'podcast' ? '生成中...' : '生成播客脚本' }}
        </button>
      </view>

      <view v-if="activeTab === 'placeholder'" class="panel">
        <button class="btn-secondary" @click="activeTab = 'home'">返回首页</button>
        <view class="placeholder-title">{{ placeholderTitle }}</view>
        <view class="placeholder-body">{{ placeholderBody }}</view>
      </view>

      <view v-if="activeTab !== 'home' && resultText" class="result">
        <view class="label">AI输出</view>
        <text class="result-text">{{ resultText }}</text>
      </view>
    </scroll-view>

    <view class="bottom-nav">
      <view class="bottom-nav-item" :class="{ active: activeNav === 'home' }" @click="openNav('home')">首页</view>
      <view class="bottom-nav-item" :class="{ active: activeNav === 'news' }" @click="openNav('news')">时政</view>
      <view class="bottom-nav-item" :class="{ active: activeNav === 'practice' }" @click="openNav('practice')">练习</view>
      <view class="bottom-nav-item" :class="{ active: activeNav === 'mine' }" @click="openNav('mine')">我的</view>
    </view>
  </view>
</template>

<script>
export default {
  data() {
    return {
      activeTab: 'home',
      activeNav: 'home',
      cards: [
        { key: 'news', name: '今日时政', desc: '每日热点、政策背景和公考使用场景' },
        { key: 'podcast', name: '双人播客', desc: '双人对话脚本，适合碎片化复习' },
        { key: 'essay', name: '申论批改', desc: '输入答案，获取AI批改' },
        { key: 'interview', name: '面试模拟', desc: '选择岗位生成题目，输入回答后获得AI点评' }
      ],
      placeholderTitle: '',
      placeholderBody: '',
      selectedPolicy: {},
      selectedRecord: {},
      essayRecords: [],
      interviewRecords: [],
      favoritePolicies: [],
      generatedPolicies: [],
      policyUpdateOpen: false,
      policyUpdateText: '',
      currentPolicyId: '',
      podcastSourcePolicy: null,
      interviewSourcePolicy: null,
      essaySourcePolicy: null,
      podcastPolicyIndex: 0,
      policyNews: [
        {
          id: 'new-quality-productivity',
          title: '新质生产力',
          date: '2026-06-11',
          tag: '高质量发展',
          summary: '新质生产力强调以科技创新推动产业创新，代表先进生产力的发展方向。它要求以数字技术、绿色技术和高端制造培育新动能，推动产业链升级、发展方式转型和公共服务提质，是高质量发展的重要抓手，也需要政府优化政策供给和应用场景。',
          background: '新质生产力是在新一轮科技革命和产业变革背景下提出的重要发展理念，核心是通过原创性、颠覆性科技创新，催生新产业、新模式、新动能。',
          significance: '新质生产力关系现代化产业体系建设和高质量发展全局，有助于突破传统增长路径依赖，提升产业竞争力、资源配置效率和发展可持续性。',
          essayAngle: '可用于高质量发展、科技创新、产业升级、人才强国等主题。申论写作中可从创新驱动、数字赋能、绿色转型和制度供给四个角度展开。',
          interviewQuestion: '请结合实际谈谈，基层政府应如何服务新质生产力发展？',
          quote: '以科技创新塑造发展新动能，以产业升级拓展发展新空间。',
          memory: '新质生产力的核心是创新，关键在产业，落点在高质量发展。答题时可围绕科技创新、人才支撑、产业升级、绿色转型展开，突出政府服务、制度保障和场景应用。'
        },
        {
          id: 'modern-grassroots-governance',
          title: '基层治理现代化',
          date: '2026-06-10',
          tag: '基层治理',
          summary: '基层治理现代化强调党建引领、多元参与、数字赋能和精细服务。它通过网格化管理、数据协同、群众参与和资源下沉，提升基层发现问题、解决问题、服务群众的能力，是国家治理现代化的重要基础，也能增强群众获得感和政策落地效率。',
          background: '基层是政策落实和服务群众的最后一公里。随着社会结构和群众需求变化，基层治理需要从粗放管理转向精准服务、协同治理和智慧治理。',
          significance: '基层治理现代化能够把制度优势转化为治理效能，提升公共服务响应速度，化解矛盾在基层，推动群众诉求有人管、问题闭环能落实。',
          essayAngle: '可用于基层治理、群众路线、数字政府、公共服务等主题。可围绕共建共治共享、网格化治理、数据协同和民生服务精准化展开。',
          interviewQuestion: '如果你所在街道要提升基层治理效能，你会从哪些方面推进？',
          quote: '基层治理的温度，来自服务群众的精度；治理体系的效能，体现在解决问题的速度。',
          memory: '基层治理现代化要抓住党建引领、多元共治、数字赋能、服务下沉四个关键词。申论可写体系建设，面试可谈问题发现、资源整合和闭环落实。'
        },
        {
          id: 'silver-economy',
          title: '银发经济',
          date: '2026-06-09',
          tag: '民生经济',
          summary: '银发经济围绕老年群体多层次需求，发展养老服务、健康管理、适老化产品和智慧养老。它既是积极应对人口老龄化的重要举措，也是扩大内需、改善民生、优化公共服务和培育新产业的结合点，推动养老从保障型向品质型升级。',
          background: '人口老龄化带来养老、医疗、照护和精神文化等多方面需求，也催生服务消费和产业升级空间。发展银发经济需要政府引导、市场供给和社会参与协同发力。',
          significance: '银发经济既回应老年群体对高质量生活的期待，也能带动养老服务、医疗健康、智能产品等产业发展，实现民生改善和扩大内需的统一。',
          essayAngle: '可用于民生保障、养老服务、扩大内需、社会治理等主题。可从供给优化、普惠服务、产业融合和适老化改造角度展开。',
          interviewQuestion: '面对人口老龄化趋势，政府应如何推动银发经济健康发展？',
          quote: '让老年人共享发展成果，让银发经济成为民生温度与发展活力的结合点。',
          memory: '银发经济一头连着民生福祉，一头连着消费潜力。答题可从养老服务供给、适老化改造、智慧养老、监管规范和普惠可及五方面展开。'
        }
      ],
      jobTypes: ['国税', '海关', '街道办', '选调生', '综合岗'],
      jobIndex: 0,
      essayTopic: '',
      essayMaterial: '',
      essay: '',
      interviewQuestion: '',
      interviewAnswer: '',
      newsText: '',
      podcastText: '',
      resultText: '',
      loading: false,
      loadingType: '',
      recording: false,
      recordingStatus: '未开始录音',
      recordingDuration: '00:00',
      recordAudioUrl: '',
      mediaRecorder: null,
      recordStream: null,
      recordChunks: [],
      recordTimer: null,
      recordStartedAt: 0
    }
  },
  mounted() {
    this.loadLocalRecords()
  },
  computed: {
    allPolicyNews() {
      const ids = new Set(this.generatedPolicies.map(item => item.id))
      return [...this.generatedPolicies, ...this.policyNews.filter(item => !ids.has(item.id))]
    },
    activePodcastPolicy() {
      return this.podcastSourcePolicy || this.allPolicyNews[this.podcastPolicyIndex] || {}
    }
  },
  methods: {
    goSettings() {
      uni.navigateTo({ url: '/pages/settings/settings' })
    },
    openFeature(key) {
      if (key === 'essay') {
        this.activeTab = 'essay'
        this.activeNav = 'practice'
        this.essaySourcePolicy = null
        return
      }
      if (key === 'interview') {
        this.activeTab = 'interview'
        this.activeNav = 'practice'
        this.interviewSourcePolicy = null
        this.resultText = ''
        return
      }
      if (key === 'podcast') {
        this.activeTab = 'podcast'
        this.activeNav = 'practice'
        this.podcastSourcePolicy = this.allPolicyNews[this.podcastPolicyIndex]
        this.resultText = ''
        return
      }
      if (key === 'news') {
        this.activeTab = 'policyList'
        this.activeNav = 'news'
        this.resultText = ''
        return
      }
      const item = ['双人播客', '请选择一条时政生成播客脚本。']
      this.placeholderTitle = item[0]
      this.placeholderBody = item[1]
      this.activeTab = 'placeholder'
      this.activeNav = key === 'news' ? 'news' : 'home'
      this.resultText = ''
    },
    openNav(key) {
      this.activeNav = key
      if (key === 'home') {
        this.activeTab = 'home'
        this.resultText = ''
        return
      }
      if (key === 'news') return this.openFeature('news')
      if (key === 'practice') {
        this.placeholderTitle = '练习'
        this.placeholderBody = '当前可进入申论批改、面试模拟和双人播客练习。'
        this.activeTab = 'placeholder'
        this.resultText = ''
        return
      }
      this.loadLocalRecords()
      this.activeTab = 'mine'
      this.resultText = ''
    },
    loadLocalRecords() {
      const data = uni.getStorageSync('copilot_local_records') || {}
      this.essayRecords = data.essays || []
      this.interviewRecords = data.interviews || []
      this.favoritePolicies = data.favoritePolicies || []
      this.generatedPolicies = data.generatedPolicies || []
    },
    saveLocalRecords() {
      uni.setStorageSync('copilot_local_records', {
        essays: this.essayRecords,
        interviews: this.interviewRecords,
        favoritePolicies: this.favoritePolicies,
        generatedPolicies: this.generatedPolicies
      })
    },
    saveEssayRecord(record) {
      this.essayRecords.unshift(record)
      if (this.essayRecords.length > 50) this.essayRecords.length = 50
      this.saveLocalRecords()
    },
    updateEssayRecordRewrite(id, rewriteText) {
      const record = this.essayRecords.find(item => item.id === id)
      if (record) {
        record.rewriteText = rewriteText
        this.saveLocalRecords()
      }
    },
    saveInterviewRecord(record) {
      this.interviewRecords.unshift(record)
      if (this.interviewRecords.length > 50) this.interviewRecords.length = 50
      this.saveLocalRecords()
    },
    favoritePolicy(item) {
      const favorite = { ...item, ts: Date.now() }
      const index = this.favoritePolicies.findIndex(policy => policy.id === item.id)
      if (index >= 0) this.favoritePolicies.splice(index, 1, favorite)
      else this.favoritePolicies.unshift(favorite)
      if (this.favoritePolicies.length > 30) this.favoritePolicies.length = 30
      this.saveLocalRecords()
      uni.showToast({ title: '已收藏', icon: 'none' })
    },
    openEssayRecord(item) {
      this.selectedRecord = item
      this.activeTab = 'essayRecord'
      this.activeNav = 'mine'
      this.resultText = ''
    },
    formatDate(ts) {
      if (!ts) return ''
      const d = new Date(ts)
      return `${d.getMonth() + 1}/${d.getDate()}`
    },
    getScoreFromReview(text) {
      const match = String(text || '').match(/SCORE:\s*(\d{1,3})/i)
      return match ? Math.min(100, Number(match[1])) : ''
    },
    extractReviewSection(text, name) {
      const re = new RegExp(`【${name}】\\s*([\\s\\S]*?)(?=【[^】]+】|$)`)
      return String(text || '').match(re)?.[1]?.trim() || ''
    },
    changePodcastPolicy(e) {
      this.podcastPolicyIndex = Number(e.detail.value)
      this.podcastSourcePolicy = this.allPolicyNews[this.podcastPolicyIndex]
      this.currentPolicyId = this.podcastSourcePolicy?.id || ''
      this.resultText = ''
    },
    formatPolicyContext(item) {
      if (!item) return ''
      return `【标题】\n${item.title}\n\n【日期】\n${item.date}\n\n【标签】\n${item.tag}\n\n【摘要】\n${item.summary}\n\n【背景概括】\n${item.background}\n\n【政策意义】\n${item.significance}\n\n【申论可用角度】\n${item.essayAngle}\n\n【面试可能问法】\n${item.interviewQuestion}\n\n【可背诵金句】\n${item.quote}\n\n【100字速记版】\n${item.memory}`
    },
    openPolicyDetail(item) {
      this.selectedPolicy = item
      this.currentPolicyId = item.id
      this.activeTab = 'policyDetail'
      this.activeNav = 'news'
      this.resultText = ''
    },
    openPodcastFromPolicy(item) {
      const index = this.allPolicyNews.findIndex(p => p.id === item.id)
      this.podcastPolicyIndex = index >= 0 ? index : 0
      this.currentPolicyId = item.id
      this.podcastSourcePolicy = item
      this.activeTab = 'podcast'
      this.activeNav = 'practice'
      this.resultText = ''
    },
    openInterviewFromPolicy(item) {
      this.activeTab = 'interview'
      this.activeNav = 'practice'
      this.currentPolicyId = item.id
      this.interviewSourcePolicy = item
      this.interviewQuestion = item.interviewQuestion
      this.interviewAnswer = ''
      this.resultText = ''
    },
    openEssayFromPolicy(item) {
      this.activeTab = 'essay'
      this.activeNav = 'practice'
      this.currentPolicyId = item.id
      this.essaySourcePolicy = item
      this.essayTopic = `请结合“${item.title}”相关材料，自选角度写一段申论分析。`
      this.essayMaterial = `【${item.title}】\n${item.summary}\n\n背景概括：${item.background}\n\n政策意义：${item.significance}\n\n申论可用角度：${item.essayAngle}\n\n可背诵金句：${item.quote}\n\n100字速记版：${item.memory}`
      this.resultText = ''
    },
    getApiKey() {
      const apiKey = uni.getStorageSync('deepseek_key')
      if (!apiKey) {
        uni.showModal({
          title: '需要API Key',
          content: '请先在设置中填入 DeepSeek API Key',
          showCancel: false
        })
        return ''
      }
      return apiKey
    },
    async apiPost(messages) {
      const apiKey = this.getApiKey()
      if (!apiKey) throw new Error('缺少 API Key')
      const res = await uni.request({
        url: 'https://api.deepseek.com/chat/completions',
        method: 'POST',
        header: {
          'Content-Type': 'application/json',
          'Authorization': `Bearer ${apiKey}`
        },
        data: {
          model: 'deepseek-chat',
          messages
        }
      })
      if (res.statusCode < 200 || res.statusCode >= 300) {
        throw new Error(res.data?.error?.message || `请求失败（HTTP ${res.statusCode}）`)
      }
      const content = res.data?.choices?.[0]?.message?.content
      if (!content) throw new Error('AI 返回了空响应，请重试')
      return this.cleanMarkdown(content)
    },
    cleanMarkdown(text) {
      return String(text)
        .replace(/#{1,6}\s*/g, '')
        .replace(/\*{1,3}([^*]+)\*{1,3}/g, '$1')
        .replace(/^[*-]\s+/gm, '')
        .replace(/`/g, '')
        .replace(/\n{3,}/g, '\n\n')
        .trim()
    },
    setEssayField(kind, text) {
      if (kind === 'topic') this.essayTopic = text
      if (kind === 'material') this.essayMaterial = text
      if (kind === 'answer') this.essay = text
    },
    getEssayFieldName(kind) {
      return { topic: '题目', material: '材料', answer: '答案' }[kind] || '内容'
    },
    async parseLocalEssayFile(file, kind) {
      if (!file) throw new Error('未选择文件')
      const name = file.name || file.path || ''
      const ext = name.split('.').pop()?.toLowerCase() || ''
      const isImage = file.type?.startsWith('image/') || ['jpg', 'jpeg', 'png', 'webp', 'bmp'].includes(ext)
      if (isImage) {
        if (typeof window === 'undefined' || !window.Tesseract) {
          throw new Error('当前端暂未加载图片识别能力，请手动输入或在 H5 页面重试')
        }
        const result = await window.Tesseract.recognize(file, 'chi_sim+eng')
        const text = result?.data?.text?.trim() || ''
        if (!text) throw new Error('未识别出文字，请换一张更清晰的照片或手动输入')
        return text
      }
      if (ext === 'txt' || file.type?.startsWith('text/')) {
        return await new Promise((resolve, reject) => {
          const reader = new FileReader()
          reader.onload = e => resolve(String(e.target.result || '').trim())
          reader.onerror = () => reject(new Error('TXT 文件读取失败'))
          reader.readAsText(file, 'UTF-8')
        })
      }
      throw new Error('当前页面暂只支持 TXT 和图片识别，PDF/DOCX 请在 H5 页面上传或手动粘贴内容')
    },
    async chooseEssayFile(kind) {
      if (typeof uni.chooseFile !== 'function') {
        uni.showToast({ title: '当前端不支持文件选择，请手动粘贴内容', icon: 'none' })
        return
      }
      const label = this.getEssayFieldName(kind)
      try {
        const chooseRes = await uni.chooseFile({
          count: 1,
          extension: ['.txt', '.pdf', '.docx', '.jpg', '.jpeg', '.png', '.webp', '.bmp']
        })
        const file = chooseRes.tempFiles?.[0]?.file || chooseRes.tempFiles?.[0]
        this.loading = true
        this.loadingType = kind === 'answer' ? 'ocr' : 'file'
        this.resultText = `正在读取${label}文件，请稍候...`
        const text = await this.parseLocalEssayFile(file, kind)
        this.setEssayField(kind, text)
        this.resultText = `${label}文件已读取：\n${text}\n\n请先校对内容，再提交批改。`
      } catch (e) {
        this.resultText = `${label}文件上传失败：${e.errMsg || e.message || '请选择更清晰或受支持的文件'}`
      } finally {
        this.loading = false
        this.loadingType = ''
      }
    },
    async chooseEssayImage(kind) {
      const label = this.getEssayFieldName(kind)
      try {
        const chooseRes = await uni.chooseImage({
          count: 1,
          sizeType: ['compressed'],
          sourceType: ['album', 'camera']
        })
        const file = chooseRes.tempFiles?.[0]?.file
        if (!file || typeof window === 'undefined' || !window.Tesseract) {
          uni.showModal({
            title: '暂不支持自动识别',
            content: '当前端暂未加载图片识别能力，请先在 H5 页面使用照片识别，或手动输入识别后的文字。',
            showCancel: false
          })
          return
        }
        this.loading = true
        this.loadingType = 'ocr'
        this.resultText = `正在识别${label}照片，请稍候...`
        const result = await window.Tesseract.recognize(file, 'chi_sim+eng')
        const text = result?.data?.text?.trim() || ''
        if (!text) throw new Error('未识别出文字，请换一张更清晰的照片或手动输入')
        this.setEssayField(kind, text)
        this.resultText = `${label}照片识别结果：\n${text}\n\n识别可能存在错字、漏字或段落错位。请先校对，再点击提交批改。`
      } catch (e) {
        this.resultText = `${label}照片识别失败：${e.errMsg || e.message || '请换一张更清晰的照片'}`
      } finally {
        this.loading = false
        this.loadingType = ''
      }
    },
    reviewEssay(text) {
      return this.apiPost([
        {
          role: 'system',
          content: `你是专业申论评分导师。请严格按以下格式输出：
SCORE: [0-100 整数]
【审题立意】
【要点覆盖】
【逻辑结构】
【语言表达】
【扣分原因】
【修改建议】
【高分改写版】
不得使用 # * - 等 Markdown 符号。`
        },
        { role: 'user', content: text }
      ])
    },
    generateInterviewQuestion(jobType, policyContext = '') {
      return this.apiPost([
        { role: 'system', content: '你是公务员结构化面试命题老师。请根据岗位类型和给定时政材料生成一道结构化面试题，题目要贴近岗位职责、基层治理真实场景和时政主题。只输出题目，不输出解析。不得使用 Markdown 符号。' },
        { role: 'user', content: `岗位类型：${jobType}\n\n时政材料：${policyContext || '无指定时政，请按岗位职责生成。'}` }
      ])
    },
    reviewInterviewAnswer(jobType, question, answer) {
      return this.apiPost([
        {
          role: 'system',
          content: `你是公务员面试考官。请按以下格式点评：
【内容完整度】
【逻辑结构】
【表达流畅度】
【机关表达】
【改进建议】
【参考答案】
不得使用 # * - 等 Markdown 符号。`
        },
        { role: 'user', content: `岗位类型：${jobType}\n面试题：${question}\n考生回答：${answer}` }
      ])
    },
    async startRecording() {
      if (typeof window === 'undefined' || !window.navigator?.mediaDevices?.getUserMedia || !window.MediaRecorder) {
        uni.showToast({ title: '当前端不支持浏览器录音', icon: 'none' })
        return
      }
      try {
        this.clearRecording(false)
        this.recordStream = await window.navigator.mediaDevices.getUserMedia({ audio: true })
        this.recordChunks = []
        this.mediaRecorder = new MediaRecorder(this.recordStream)
        this.mediaRecorder.ondataavailable = e => {
          if (e.data?.size) this.recordChunks.push(e.data)
        }
        this.mediaRecorder.onstop = this.finishRecording
        this.mediaRecorder.start()
        this.recording = true
        this.recordingStatus = '正在录音'
        this.recordStartedAt = Date.now()
        this.recordTimer = setInterval(this.updateRecordingDuration, 500)
        this.updateRecordingDuration()
      } catch (e) {
        this.stopRecordStream()
        uni.showToast({ title: e.name === 'NotAllowedError' ? '请允许使用麦克风' : '无法访问麦克风', icon: 'none' })
      }
    },
    stopRecording() {
      if (this.mediaRecorder?.state === 'recording') {
        this.mediaRecorder.stop()
        this.recording = false
        this.recordingStatus = '正在生成录音回放'
      }
    },
    finishRecording() {
      clearInterval(this.recordTimer)
      this.recordTimer = null
      this.stopRecordStream()
      if (!this.recordChunks.length) {
        this.recordingStatus = '未获取到有效录音，请重试'
        return
      }
      const blob = new Blob(this.recordChunks, { type: this.recordChunks[0]?.type || 'audio/webm' })
      if (this.recordAudioUrl) URL.revokeObjectURL(this.recordAudioUrl)
      this.recordAudioUrl = URL.createObjectURL(blob)
      this.recordingStatus = '录音已完成，可回放'
      this.resultText = '录音已完成。当前不做自动转写，请手动粘贴转写文本到回答框后再点击 AI点评。'
    },
    clearRecording(resetStatus = true) {
      if (this.mediaRecorder?.state === 'recording') this.mediaRecorder.stop()
      clearInterval(this.recordTimer)
      this.recordTimer = null
      this.recording = false
      this.recordChunks = []
      this.stopRecordStream()
      this.mediaRecorder = null
      if (this.recordAudioUrl) URL.revokeObjectURL(this.recordAudioUrl)
      this.recordAudioUrl = ''
      if (resetStatus) {
        this.recordingDuration = '00:00'
        this.recordingStatus = '未开始录音'
      }
    },
    updateRecordingDuration() {
      if (!this.recordStartedAt) return
      const sec = Math.floor((Date.now() - this.recordStartedAt) / 1000)
      const mm = String(Math.floor(sec / 60)).padStart(2, '0')
      const ss = String(sec % 60).padStart(2, '0')
      this.recordingDuration = `${mm}:${ss}`
    },
    stopRecordStream() {
      this.recordStream?.getTracks().forEach(track => track.stop())
      this.recordStream = null
    },
    summarizePolicyNews(text) {
      return this.apiPost([
        {
          role: 'system',
          content: `你是时政精读老师。请按以下格式输出：
【事件概括】
【政策背景】
【申论可用角度】
【面试可能问法】
【可背诵金句】
【100字速记版】
不得使用 # * - 等 Markdown 符号。`
        },
        { role: 'user', content: text }
      ])
    },
    generatePolicyCard(text) {
      return this.apiPost([
        {
          role: 'system',
          content: `你是公务员考试时政教研老师。请把用户粘贴的新闻或政策材料提炼成一条今日时政学习卡片。
请严格按以下格式输出：
【标题】
【标签】
【100字摘要】
【政策背景】
【申论可用角度】
【面试可能问法】
【可背诵金句】
【100字速记版】
不得使用 # * - 等 Markdown 符号，不得输出寒暄，不得编造具体时间、地点或数据。`
        },
        { role: 'user', content: text }
      ])
    },
    parsePolicyCard(raw, sourceText = '') {
      const clean = this.cleanMarkdown(raw)
      const section = name => {
        const re = new RegExp(`【${name}】\\s*([\\s\\S]*?)(?=【[^】]+】|$)`)
        return clean.match(re)?.[1]?.trim() || ''
      }
      const today = new Date().toISOString().slice(0, 10)
      const background = section('政策背景') || section('背景概括')
      return {
        id: `policy-${Date.now().toString(36)}`,
        ts: Date.now(),
        date: today,
        title: section('标题') || sourceText.slice(0, 18) || '今日时政',
        tag: section('标签') || '今日更新',
        summary: section('100字摘要') || clean.slice(0, 120),
        background,
        significance: background,
        essayAngle: section('申论可用角度'),
        interviewQuestion: section('面试可能问法'),
        quote: section('可背诵金句'),
        memory: section('100字速记版'),
        sourceText,
        raw: clean
      }
    },
    submitPolicyUpdate() {
      const text = this.policyUpdateText.trim()
      if (!text) return
      this.loading = true
      this.loadingType = 'policyUpdate'
      this.resultText = ''
      this.generatePolicyCard(text)
        .then(result => {
          const card = this.parsePolicyCard(result, text)
          this.generatedPolicies = this.generatedPolicies.filter(item => item.id !== card.id)
          this.generatedPolicies.unshift(card)
          if (this.generatedPolicies.length > 50) this.generatedPolicies.length = 50
          this.saveLocalRecords()
          this.policyUpdateText = ''
          this.policyUpdateOpen = false
          uni.showToast({ title: '已生成时政卡片', icon: 'none' })
        })
        .catch(e => {
          this.resultText = `生成时政失败：${e.errMsg || e.message || '请检查网络或 API Key'}`
        })
        .finally(() => {
          this.loading = false
          this.loadingType = ''
        })
    },
    generatePodcastScript(text) {
      return this.apiPost([
        {
          role: 'system',
          content: `你是中文公考学习播客脚本策划。请生成约3分钟中文双人播客脚本。
必须多轮对话，明确标注：
主持人A：
主持人B：
主持人A：
主持人B：
内容必须覆盖：热点背景、为什么重要、申论怎么用、面试怎么答、最后一句速记总结。
口语化但不松散，适合直接录制，不要生成真实音频。
不得使用 # * - 等 Markdown 符号。`
        },
        { role: 'user', content: text }
      ])
    },
    async run(type, task) {
      this.loading = true
      this.loadingType = type
      this.resultText = ''
      try {
        this.resultText = await task()
      } catch (e) {
        this.resultText = `请求失败：${e.errMsg || e.message || '请检查网络或 API Key'}`
      } finally {
        this.loading = false
        this.loadingType = ''
      }
    },
    submitEssay() {
      const topic = this.essayTopic.trim()
      const material = this.essayMaterial.trim()
      const answer = this.essay.trim()
      if (!topic || !material || !answer) return
      const text = `【题目】\n${topic}\n\n【材料】\n${material}\n\n【考生答案】\n${answer}`
      this.loading = true
      this.loadingType = 'essay'
      this.resultText = ''
      this.reviewEssay(text)
        .then(result => {
          this.resultText = result
          this.saveEssayRecord({
            id: Date.now().toString(36),
            ts: Date.now(),
            topic,
            material,
            answer,
            reviewText: result,
            rewriteText: this.extractReviewSection(result, '高分改写版'),
            score: this.getScoreFromReview(result)
          })
        })
        .catch(e => {
          this.resultText = `请求失败：${e.errMsg || e.message || '请检查网络或 API Key'}`
        })
        .finally(() => {
          this.loading = false
          this.loadingType = ''
        })
    },
    async createQuestion() {
      const jobType = this.jobTypes[this.jobIndex]
      this.loading = true
      this.loadingType = 'question'
      this.resultText = ''
      try {
        const policyContext = this.interviewSourcePolicy ? this.formatPolicyContext(this.interviewSourcePolicy) : ''
        this.interviewQuestion = await this.generateInterviewQuestion(jobType, policyContext)
      } catch (e) {
        this.resultText = `生成题目失败：${e.errMsg || e.message || '请检查网络或 API Key'}`
      } finally {
        this.loading = false
        this.loadingType = ''
      }
    },
    submitInterview() {
      const answer = this.interviewAnswer.trim()
      if (!this.interviewQuestion || !answer) return
      const jobType = this.jobTypes[this.jobIndex]
      this.loading = true
      this.loadingType = 'interview'
      this.resultText = ''
      this.reviewInterviewAnswer(jobType, this.interviewQuestion, answer)
        .then(result => {
          this.resultText = result
          this.saveInterviewRecord({
            id: Date.now().toString(36),
            ts: Date.now(),
            jobType,
            question: this.interviewQuestion,
            answer,
            reviewText: result
          })
        })
        .catch(e => {
          this.resultText = `请求失败：${e.errMsg || e.message || '请检查网络或 API Key'}`
        })
        .finally(() => {
          this.loading = false
          this.loadingType = ''
        })
    },
    submitNews() {
      const text = this.newsText.trim()
      if (!text) return
      this.run('news', () => this.summarizePolicyNews(text))
    },
    submitPodcast() {
      const item = this.activePodcastPolicy
      const text = this.formatPolicyContext(item)
      this.run('podcast', () => this.generatePodcastScript(text))
    }
  }
}
</script>

<style scoped>
.page { min-height: 100vh; background: #f7f8fa; color: #1f2937; }
.topbar {
  display: flex; justify-content: space-between; align-items: center;
  padding: 24rpx 28rpx; background: #fff; border-bottom: 1rpx solid #e5e7eb;
}
.brand { font-size: 32rpx; font-weight: 700; color: #111827; }
.setting { font-size: 26rpx; color: #2563eb; }
.tabs { display: flex; background: #fff; border-bottom: 1rpx solid #e5e7eb; }
.tab {
  flex: 1; text-align: center; padding: 22rpx 4rpx; font-size: 25rpx;
  color: #6b7280; border-bottom: 4rpx solid transparent;
}
.tab.active { color: #2563eb; border-bottom-color: #2563eb; font-weight: 700; }
.body { height: calc(100vh - 152rpx); padding-bottom: 120rpx; box-sizing: border-box; }
.home-grid { padding: 24rpx; display: flex; flex-direction: column; gap: 20rpx; }
.home-card {
  background: #fff; border: 1rpx solid #dbeafe; border-radius: 24rpx;
  padding: 30rpx; box-shadow: 0 8rpx 28rpx rgba(37,99,235,0.08);
}
.home-card-title { font-size: 34rpx; font-weight: 800; color: #1f2937; margin-bottom: 12rpx; }
.home-card-desc { font-size: 26rpx; color: #6b7280; line-height: 1.6; }
.panel, .result {
  margin: 20rpx; padding: 24rpx; background: #fff; border-radius: 14rpx;
  border: 1rpx solid #edf0f3;
}
.inline-panel { margin: 0 0 20rpx; border-color: #dbeafe; }
.placeholder-title { font-size: 36rpx; font-weight: 800; color: #1d4ed8; margin-bottom: 18rpx; }
.placeholder-body { font-size: 28rpx; line-height: 1.8; color: #4b5563; }
.policy-list { padding: 24rpx; }
.policy-card {
  background: #fff; border: 1rpx solid #dbeafe; border-radius: 24rpx;
  padding: 28rpx; margin-bottom: 20rpx;
  box-shadow: 0 8rpx 28rpx rgba(37,99,235,0.08);
}
.policy-title { font-size: 32rpx; font-weight: 800; color: #1f2937; margin-bottom: 12rpx; }
.policy-meta { display: flex; gap: 12rpx; align-items: center; margin-bottom: 12rpx; font-size: 24rpx; color: #6b7280; }
.policy-tag { color: #2563eb; background: #eff6ff; border-radius: 999rpx; padding: 4rpx 14rpx; font-weight: 700; }
.policy-summary { font-size: 26rpx; line-height: 1.7; color: #4b5563; }
.policy-more { margin-top: 16rpx; font-size: 26rpx; color: #2563eb; font-weight: 700; }
.my-page { padding: 24rpx; }
.my-section { margin-bottom: 30rpx; }
.my-title { font-size: 30rpx; font-weight: 800; color: #1f2937; margin-bottom: 18rpx; }
.my-empty {
  background: #fff; border: 1rpx solid #e5e7eb; border-radius: 16rpx;
  padding: 24rpx; color: #9ca3af; font-size: 26rpx; line-height: 1.7;
}
.detail-text {
  background: #fff; border: 1rpx solid #e5e7eb; border-radius: 14rpx;
  padding: 22rpx; font-size: 28rpx; line-height: 1.8; color: #374151;
  margin-bottom: 22rpx;
}
.label { font-size: 24rpx; font-weight: 700; color: #9ca3af; margin-bottom: 14rpx; }
.textarea {
  width: 100%; min-height: 220rpx; box-sizing: border-box; padding: 20rpx;
  border: 1rpx solid #e5e7eb; border-radius: 12rpx; font-size: 28rpx;
  line-height: 1.8; margin-bottom: 20rpx; background: #fff;
}
.textarea.small { min-height: 150rpx; }
.textarea.tall { min-height: 360rpx; }
.upload-row { display: flex; gap: 16rpx; margin-bottom: 18rpx; }
.picker, .question {
  padding: 20rpx; border: 1rpx solid #e5e7eb; border-radius: 12rpx;
  font-size: 28rpx; line-height: 1.7; margin-bottom: 20rpx; background: #f9fafb;
}
.question { color: #1e3a8a; background: #eff6ff; border-color: #dbeafe; }
.btn-primary {
  background: #2563eb; color: #fff; border-radius: 12rpx; padding: 20rpx;
  font-size: 30rpx; border: none; margin-bottom: 20rpx;
}
.btn-primary[disabled] { opacity: 0.45; }
.btn-secondary {
  background: #eff6ff; color: #2563eb; border-radius: 12rpx; padding: 18rpx;
  font-size: 28rpx; border: 1rpx solid #bfdbfe; margin-bottom: 20rpx;
}
.btn-secondary[disabled] { opacity: 0.45; }
.btn-secondary.compact { flex: 1; margin-bottom: 0; }
.recorder {
  background: #f9fafb; border: 1rpx solid #e5e7eb; border-radius: 12rpx;
  padding: 20rpx; margin-bottom: 20rpx;
}
.recorder-row { display: flex; gap: 16rpx; margin-bottom: 14rpx; }
.recorder-meta { font-size: 24rpx; color: #6b7280; margin-bottom: 12rpx; }
.hint { font-size: 24rpx; color: #9ca3af; line-height: 1.6; }
.audio { width: 100%; margin-bottom: 12rpx; }
.empty { text-align: center; color: #9ca3af; font-size: 28rpx; padding: 30rpx 0; }
.result-text {
  display: block; white-space: pre-wrap; font-size: 28rpx; line-height: 1.85;
  color: #374151;
}
.bottom-nav {
  position: fixed; left: 0; right: 0; bottom: 0; height: 104rpx;
  background: #fff; border-top: 1rpx solid #e5e7eb;
  display: flex; box-shadow: 0 -8rpx 28rpx rgba(15,23,42,0.08);
}
.bottom-nav-item {
  flex: 1; display: flex; align-items: center; justify-content: center;
  font-size: 24rpx; font-weight: 700; color: #6b7280;
}
.bottom-nav-item.active { color: #2563eb; }
</style>
