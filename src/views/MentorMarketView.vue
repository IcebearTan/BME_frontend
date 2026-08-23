<template>
  <div :class="['mentor-market', { 'theme-dark': isDarkMode }]">
    <main class="shell">
      <DewCard class="market-hero" glass tinted accent="info" no-hover>
        <div class="hero-constellation" aria-hidden="true">
          <span class="hero-orbit hero-orbit--one"></span>
          <span class="hero-orbit hero-orbit--two"></span>
          <span class="hero-node hero-node--medical">医学</span>
          <span class="hero-node hero-node--engineering">工程</span>
          <span class="hero-node hero-node--spark">✦</span>
          <span class="hero-bridge"></span>
          <span class="hero-constellation__label">医工融合<br><b>创新成长</b></span>
        </div>
        <div class="hero-topline">
          <DewButton class="hero-back" size="sm" :active="true" @click="router.push('/home')">← 返回首页</DewButton>
          <DewBadge :type="activity?.is_open ? 'success' : 'warning'">
            {{ activity?.is_open ? '双选进行中' : '等待开放' }}
          </DewBadge>
        </div>

        <div class="hero-content">
          <div>
            <p class="eyebrow">BME GROWTH COMMUNITY</p>
            <h1>导生双选</h1>
            <p class="hero-description">找到适合你的导生，开启你的医工探索之旅。</p>
          </div>
          <div class="hero-rules" aria-label="双选规则">
            <span>3–8 人小组</span>
            <span>实时名额</span>
            <span>开放期间可更换</span>
          </div>
        </div>
      </DewCard>

      <DewCard v-if="loading" class="notice" no-hover>
        <p class="eyebrow">正在连接活动</p>
        <h2>正在读取导师双选信息…</h2>
      </DewCard>

      <DewCard v-else-if="errorMessage" class="notice notice--error" tinted accent="warning" no-hover>
        <p class="eyebrow">暂时无法进入</p>
        <h2>活动信息加载失败</h2>
        <p>{{ errorMessage }}</p>
      </DewCard>

      <template v-else-if="activity">
        <DewCard v-if="!activity.is_open && !canEditProfile" class="notice timing-card" tinted accent="warning" no-hover>
          <DewBadge type="warning">双选尚未开放</DewBadge>
          <h2>{{ startText }}</h2>
          <p>开抢前不会展示导生名单；到达设定时间后，本页面会自动更新。</p>
        </DewCard>

        <section v-if="canEditProfile" class="mentor-workspace page-enter">
          <DewCard class="profile-editor" variant="elevated" divided>
            <template #header>
              <div class="section-heading">
                <div>
                  <p class="section-kicker">MENTOR PROFILE</p>
                  <h2>我的展示卡</h2>
                  <p>先准备你的海报与介绍，发布后同学便能在活动开放时看到你。</p>
                </div>
                <DewBadge :type="profile.poster_url ? 'success' : 'neutral'">
                  {{ profile.poster_url ? '海报已上传' : '待上传海报' }}
                </DewBadge>
              </div>
            </template>

            <label class="poster-upload">
              <img v-if="profile.poster_url" :src="posterSrc(profile.poster_url)" alt="我的导生海报" />
              <span v-else class="poster-upload__placeholder">
                <b>上传你的自我介绍海报</b>
                <small>建议使用 4:3 图片 · JPG / PNG / WEBP · 最大 10MB</small>
              </span>
              <span class="poster-upload__action">{{ profile.poster_url ? '更换海报' : '选择图片' }}</span>
              <input type="file" accept="image/png,image/jpeg,image/webp" @change="uploadPoster" />
            </label>

            <el-form class="profile-form" label-position="top">
              <el-form-item label="年级">
                <el-select v-model="profile.grade" placeholder="选择入学年份" style="width: 100%">
                  <el-option v-for="year in gradeOptions" :key="year" :label="`${year}级`" :value="`${year}级`" />
                </el-select>
              </el-form-item>
              <el-form-item label="组别 / 兴趣方向">
                <el-select v-model="profile.group_name" placeholder="选择你想带领的细分方向" style="width: 100%">
                  <el-option-group v-for="category in directionOptions" :key="category.label" :label="category.label">
                    <el-option v-for="option in category.options" :key="option" :label="option" :value="option" />
                  </el-option-group>
                </el-select>
              </el-form-item>
              <el-form-item label="一句自我介绍">
                <el-input v-model="profile.introduction" type="textarea" :rows="3" maxlength="300" show-word-limit placeholder="想和同学们一起做什么？" />
              </el-form-item>
              <div class="capacity-rule">
                <div>
                  <span>小组名额</span>
                  <strong>设定你愿意陪伴的同学人数</strong>
                  <small>可在 3–8 人之间选择，已有人加入后不能低于当前人数。</small>
                </div>
                <el-select v-model="profile.capacity" class="capacity-select" aria-label="可带人数">
                  <el-option v-for="count in capacityOptions" :key="count" :label="`${count} 人`" :value="count" />
                </el-select>
              </div>
              <DewButton block :active="!savingProfile" :disabled="savingProfile" @click="saveProfile">
                {{ savingProfile ? '保存中…' : '保存并发布展示卡' }}
              </DewButton>
            </el-form>
          </DewCard>

          <DewCard class="student-board" variant="elevated" divided>
            <template #header>
              <div class="section-heading">
                <div>
                  <p class="section-kicker">MY GROUP</p>
                  <h2>选择我的同学</h2>
                </div>
                <DewBadge :type="students.length >= profile.capacity ? 'neutral' : 'success'">{{ students.length }} / {{ profile.capacity || 4 }} 人</DewBadge>
              </div>
            </template>

            <div v-if="students.length" class="student-list">
              <article v-for="student in students" :key="student.student_id">
                <span class="student-avatar">{{ student.name?.slice(0, 1) || '?' }}</span>
                <div>
                  <strong>{{ student.name }}</strong>
                  <p>{{ student.message || '还没有留下悄悄话。' }}</p>
                </div>
              </article>
            </div>
            <div v-else class="empty-state">
              <span class="empty-orbit">✦</span>
              <h3>小组正在等待第一位成员</h3>
              <p>上传并发布海报后，等待活动开放吧。</p>
            </div>
          </DewCard>
        </section>

        <template v-else-if="activity.is_open">
          <section v-if="myChoice" class="my-match page-enter" aria-label="我的导生">
            <div class="section-title-row">
              <div>
                <p class="section-kicker">MY MENTOR</p>
                <h2>我的导生</h2>
              </div>
              <DewTag type="success" round>开放期间可更换</DewTag>
            </div>
            <DewCard class="my-match-card" variant="elevated">
              <div class="my-match-poster">
                <img v-if="currentMentor?.poster_url" :src="posterSrc(currentMentor.poster_url)" :alt="`${myChoice.mentor_name} 的海报`" />
                <span v-else>{{ myChoice.mentor_name?.slice(0, 1) || '导' }}</span>
              </div>
              <div class="my-match-content">
                <DewBadge type="success">✓ 当前加入</DewBadge>
                <h3>{{ myChoice.mentor_name }}</h3>
                <div class="profile-tags">
                  <DewTag v-if="currentMentor?.grade" type="neutral" size="sm">{{ currentMentor.grade }}</DewTag>
                  <DewTag type="info" size="sm">{{ currentMentor?.group_name || '成长伙伴' }}</DewTag>
                </div>
                <p class="my-match-message">{{ myChoice.message ? `“${myChoice.message}”` : '还没有留下想说的话。' }}</p>
              </div>
              <div class="my-match-capacity">
                <div class="capacity-heading">
                  <span>小组进度</span>
                  <strong>{{ currentMentor?.taken ?? '—' }} <em>/ {{ currentMentor?.capacity ?? '—' }}</em></strong>
                </div>
                <DewProgress :percentage="currentMentor ? capacityPercentage(currentMentor) : 0" size="sm" />
                <div class="my-match-actions">
                  <DewButton type="ghost" size="sm" @click="scrollToExplore">探索导生</DewButton>
                  <DewButton size="sm" :active="true" @click="openChoose(currentMentor || { mentor_id: myChoice.mentor_id, name: myChoice.mentor_name })">更新留言</DewButton>
                </div>
              </div>
            </DewCard>
          </section>

          <section ref="exploreSection" class="explore-header page-enter">
            <div>
              <p class="section-kicker">EXPLORE MENTORS</p>
              <h2>{{ myChoice ? '探索更多导生' : '探索导生' }}</h2>
              <p>探索不同方向的导生，找到适合你的成长伙伴。</p>
            </div>
            <div class="explore-meta">
              <DewTag type="primary" size="sm" round>{{ profiles.length }} 位导生</DewTag>
              <DewTag type="info" size="sm" round>随机探索</DewTag>
            </div>
          </section>

          <section :class="['profile-grid', { 'profile-grid--feature': profiles.length === 1 }]" aria-label="导生列表">
            <DewCard
              v-for="(item, index) in profiles"
              :key="item.id"
              class="profile-card"
              variant="elevated"
              :style="{ '--reveal-index': index }"
            >
              <div class="poster">
                <img v-if="item.poster_url" :src="posterSrc(item.poster_url)" :alt="`${item.name} 的自我介绍海报`" />
                <div v-else class="poster-fallback">
                  <span>{{ item.name?.slice(0, 1) || '导' }}</span>
                  <small>自我介绍海报待补充</small>
                </div>
                <DewBadge class="poster-status" :type="profileStateType(item)">{{ profileStateLabel(item) }}</DewBadge>
              </div>

              <div class="profile-content">
                <div class="profile-title-row">
                  <div>
                    <h3>{{ item.name }}</h3>
                    <p>{{ item.grade || '年级待补充' }}</p>
                  </div>
                  <span class="profile-initial">{{ item.name?.slice(0, 1) || '导' }}</span>
                </div>

                <div class="profile-tags">
                  <DewTag type="info" size="sm">{{ item.group_name || '学习搭子招募中' }}</DewTag>
                  <DewTag v-if="myChoice?.mentor_id === item.mentor_id" type="success" size="sm">我的导生</DewTag>
                </div>

                <p class="intro">{{ item.introduction || '这位导生还没有填写介绍。' }}</p>

                <div class="capacity-block">
                  <div class="capacity-heading">
                    <span>小组名额</span>
                    <strong>{{ item.taken }} <em>/ {{ item.capacity }}</em></strong>
                  </div>
                  <DewProgress :percentage="capacityPercentage(item)" size="sm" />
                  <p>{{ capacityDescription(item) }}</p>
                </div>

                <div class="profile-action">
                  <span>{{ myChoice?.mentor_id === item.mentor_id ? '继续与你的导生交流' : '从这里开始认识彼此' }}</span>
                  <DewButton
                    block
                    :active="myChoice?.mentor_id === item.mentor_id"
                    :disabled="item.remaining <= 0 || choosing"
                    @click="openChoose(item)"
                  >
                    {{ choiceButtonLabel(item) }}
                  </DewButton>
                </div>
              </div>
            </DewCard>
          </section>

          <DewCard v-if="!profiles.length" class="notice empty-list" no-hover>
            <span class="empty-orbit">✦</span>
            <h2>导生们正在准备海报</h2>
            <p>请稍后刷新，再来认识未来的学习搭子。</p>
          </DewCard>
        </template>
      </template>
    </main>

    <DewDialog v-model="chooseDialog" :title="myChoice ? '确认更换导生' : '确认选择导生'" :width="460" :close-on-click-modal="false">
      <div v-if="selectedProfile" class="choice-dialog">
        <div class="choice-dialog__mentor">
          <span>{{ selectedProfile.name?.slice(0, 1) || '导' }}</span>
          <div>
            <strong>{{ selectedProfile.name }}</strong>
            <p>{{ selectedProfile.grade || '年级待补充' }} · {{ selectedProfile.group_name || '学习搭子' }}</p>
          </div>
        </div>
        <p class="choice-dialog__copy">你将{{ myChoice ? '更换为' : '选择' }}这位导生作为你的学习搭子。</p>
        <label class="choice-dialog__label" for="mentor-message">留一句想说的话 <span>可选，最多 100 字</span></label>
        <el-input id="mentor-message" v-model="choiceMessage" type="textarea" :rows="4" maxlength="100" show-word-limit placeholder="例如：很期待和大家一起学习、交流！" />
      </div>
      <template #footer>
        <DewButton type="ghost" @click="chooseDialog = false">再想想</DewButton>
        <DewButton :active="true" :disabled="choosing" @click="confirmChoose">
          {{ choosing ? '提交中…' : `确认${myChoice ? '更换' : '选择'}` }}
        </DewButton>
      </template>
    </DewDialog>
  </div>
</template>

<script setup>
import { computed, onBeforeUnmount, onMounted, ref } from 'vue'
import { useRouter } from 'vue-router'
import { useStore } from 'vuex'
import { ElMessage } from 'element-plus'
import api, { API_URL } from '../api'
import DewBadge from '../components/ui/DewBadge.vue'
import DewButton from '../components/ui/DewButton.vue'
import DewCard from '../components/ui/DewCard.vue'
import DewDialog from '../components/ui/DewDialog.vue'
import DewProgress from '../components/ui/DewProgress.vue'
import DewTag from '../components/ui/DewTag.vue'

const router = useRouter()
const store = useStore()
const isDarkMode = computed(() => store.getters.isDarkMode)

const loading = ref(true)
const errorMessage = ref('')
const activity = ref(null)
const canEditProfile = ref(false)
const profile = ref({ grade: '', group_name: '', introduction: '', capacity: 4, poster_url: null })
const profiles = ref([])
const students = ref([])
const myChoice = ref(null)
const savingProfile = ref(false)
const choosing = ref(false)
const chooseDialog = ref(false)
const selectedProfile = ref(null)
const choiceMessage = ref('')
const exploreSection = ref(null)
let refreshTimer = null

const startText = computed(() => activity.value?.starts_at ? `预计 ${activity.value.starts_at.replace('T', ' ')} 开始` : '活动暂未排期')
const gradeOptions = Array.from({ length: 6 }, (_, index) => new Date().getFullYear() - index)
const directionOptions = [
  { label: '硬件组', options: ['电路组', '嵌软组', '拆修小队'] },
  { label: '软件组', options: ['Web开发组', '医学影像组', '计算机视觉组', '机器人控制组', '深度学习组', '大模型组', '虚拟现实组'] },
  { label: '先进制造组', options: ['3D打印组', '建模仿真组', '柔性电子组'] },
]
const capacityOptions = [3, 4, 5, 6, 7, 8]
const posterSrc = (url) => /^https?:\/\//.test(url || '') ? url : `${(API_URL || '').replace(/\/$/, '')}${url || ''}`
const currentMentor = computed(() => profiles.value.find((item) => item.mentor_id === myChoice.value?.mentor_id) || null)
const capacityPercentage = (item) => Math.min(100, (Number(item.taken) || 0) / (Number(item.capacity) || 4) * 100)
const profileStateLabel = (item) => {
  if (myChoice.value?.mentor_id === item.mentor_id) return '已选择'
  return item.remaining > 0 ? '招募中' : '已满员'
}
const profileStateType = (item) => {
  if (myChoice.value?.mentor_id === item.mentor_id) return 'success'
  return item.remaining > 0 ? 'primary' : 'neutral'
}
const capacityDescription = (item) => {
  if (myChoice.value?.mentor_id === item.mentor_id) return '你已在这个小组中'
  return item.remaining > 0 ? `还可加入 ${item.remaining} 人` : '这个小组暂时满员'
}
const choiceButtonLabel = (item) => {
  if (item.remaining <= 0) return '名额已满'
  if (myChoice.value?.mentor_id === item.mentor_id) return '更新留言'
  return myChoice.value ? `更换至此 · 剩 ${item.remaining} 位` : `选择导生 · 剩 ${item.remaining} 位`
}

async function loadActivity(quiet = false) {
  if (!quiet) loading.value = true
  try {
    const featured = await api.get('/mentor-market/featured')
    if (!featured.data?.activity?.id) throw new Error('当前特色营期尚未配置导生拼团活动')
    const detail = await api.get(`/mentor-market/activities/${featured.data.activity.id}`)
    activity.value = detail.data.activity
    canEditProfile.value = !!activity.value.can_edit_profile
    if (canEditProfile.value) {
      const [mine, result] = await Promise.all([
        api.get(`/mentor-market/activities/${activity.value.id}/my-profile`),
        api.get(`/mentor-market/activities/${activity.value.id}/my-students`),
      ])
      profile.value = mine.data.profile || { grade: '', group_name: '', introduction: '', capacity: 4, poster_url: null }
      students.value = result.data.students || []
    } else if (activity.value.is_open) {
      const [list, choice] = await Promise.all([
        api.get(`/mentor-market/activities/${activity.value.id}/profiles`),
        api.get(`/mentor-market/activities/${activity.value.id}/choice`),
      ])
      profiles.value = list.data.profiles || []
      myChoice.value = choice.data.choice
    }
    errorMessage.value = ''
  } catch (error) {
    if (!quiet) errorMessage.value = error.response?.data?.message || error.message || '加载失败'
  } finally {
    loading.value = false
  }
}

async function uploadPoster(event) {
  const file = event.target.files?.[0]
  if (!file || !activity.value) return
  if (file.size > 10 * 1024 * 1024) {
    ElMessage.warning('海报不能超过 10MB')
    return
  }
  const data = new FormData()
  data.append('poster', file)
  try {
    const res = await api.post(`/mentor-market/activities/${activity.value.id}/my-profile/poster`, data, { headers: { 'Content-Type': 'multipart/form-data' } })
    profile.value = { ...profile.value, ...res.data.profile }
    ElMessage.success('海报上传成功')
  } catch (error) {
    ElMessage.error(error.response?.data?.message || '海报上传失败')
  }
  event.target.value = ''
}

async function saveProfile() {
  if (!profile.value.poster_url) {
    ElMessage.warning('请先上传海报，再发布展示卡')
    return
  }
  savingProfile.value = true
  try {
    const res = await api.put(`/mentor-market/activities/${activity.value.id}/my-profile`, { ...profile.value, is_published: true })
    profile.value = res.data.profile
    ElMessage.success('展示卡已发布')
  } catch (error) {
    ElMessage.error(error.response?.data?.message || '保存失败')
  } finally {
    savingProfile.value = false
  }
}

function openChoose(item) {
  selectedProfile.value = item
  choiceMessage.value = ''
  chooseDialog.value = true
}

function scrollToExplore() {
  exploreSection.value?.scrollIntoView({ behavior: 'smooth', block: 'start' })
}

async function confirmChoose() {
  if (!selectedProfile.value) return
  choosing.value = true
  try {
    const res = await api.post(`/mentor-market/activities/${activity.value.id}/profiles/${selectedProfile.value.id}/choose`, { message: choiceMessage.value })
    myChoice.value = { ...res.data.choice, message: choiceMessage.value }
    chooseDialog.value = false
    ElMessage.success('选择成功！')
    await loadActivity(true)
  } catch (error) {
    ElMessage.error(error.response?.data?.message || '选择失败，请刷新后重试')
  } finally {
    choosing.value = false
  }
}

onMounted(() => {
  loadActivity()
  refreshTimer = window.setInterval(() => loadActivity(true), 15000)
})

onBeforeUnmount(() => window.clearInterval(refreshTimer))
</script>

<style scoped>
.mentor-market {
  min-height: 100vh;
  padding: var(--space-xl) var(--space-lg) 72px;
  color: var(--dew-text);
  background:
    radial-gradient(circle at 8% 0%, color-mix(in srgb, var(--color-info-light) 76%, transparent), transparent 30%),
    radial-gradient(circle at 92% 18%, color-mix(in srgb, var(--color-primary-light) 58%, transparent), transparent 27%),
    var(--color-bg-soft);
}

.shell { max-width: 1180px; margin: 0 auto; }
.market-hero { margin-bottom: var(--space-xl); }
.hero-topline, .hero-content, .hero-rules, .section-heading, .choice-summary, .explore-header, .capacity-heading, .capacity-rule, .choice-dialog__mentor { display: flex; }
.hero-topline { align-items: center; justify-content: space-between; }
.hero-content { align-items: flex-end; justify-content: space-between; gap: 32px; padding-top: 18px; }
.eyebrow, .section-kicker { margin: 0; color: var(--color-primary); font-size: var(--text-xs); font-weight: 700; letter-spacing: .13em; }
.hero-content h1 { margin: 7px 0 10px; color: var(--dew-text-heading); font-size: clamp(32px, 5vw, 48px); letter-spacing: -.045em; line-height: 1.06; }
.hero-description { max-width: 570px; margin: 0; color: var(--dew-text-muted); font-size: var(--text-lg); line-height: 1.65; }
.hero-rules { flex-wrap: wrap; justify-content: flex-end; gap: var(--space-sm); max-width: 380px; }
.hero-rules span { padding: 8px 10px; border: 1px solid var(--dew-card-border); border-radius: var(--radius-md); background: color-mix(in srgb, var(--dew-card-bg) 72%, transparent); color: var(--dew-text-muted); font-size: var(--text-xs); white-space: nowrap; }
.hero-rules strong { margin-right: 4px; color: var(--dew-text-heading); font-size: var(--text-sm); }

.notice { margin: 0 auto; padding: 38px 24px; text-align: center; }
.notice h2 { margin: 8px 0; color: var(--dew-text-heading); font-size: var(--text-xl); }
.notice p { margin: 0; color: var(--dew-text-muted); line-height: 1.7; }
.timing-card, .notice--error { max-width: 680px; }

.mentor-workspace { display: grid; grid-template-columns: minmax(0, 1.2fr) minmax(300px, .8fr); gap: var(--space-lg); }
.section-heading { align-items: flex-start; justify-content: space-between; gap: var(--space-md); }
.section-heading h2, .explore-header h2 { margin: 4px 0 5px; color: var(--dew-text-heading); font-size: var(--text-xl); }
.section-heading p:not(.section-kicker), .explore-header p:not(.section-kicker) { margin: 0; color: var(--dew-text-muted); font-size: var(--text-sm); line-height: 1.6; }

.poster-upload { position: relative; display: flex; align-items: center; justify-content: center; min-height: 250px; aspect-ratio: 4 / 3; overflow: hidden; border: 1.5px dashed var(--dew-card-border); border-radius: var(--radius-lg); background: var(--dew-card-inset-bg); color: var(--dew-text-muted); text-align: center; cursor: pointer; transition: border-color .25s ease, transform .25s var(--dew-bounce), box-shadow .25s ease; }
.poster-upload:hover { border-color: var(--color-primary); box-shadow: var(--dew-card-shadow-hover); transform: translateY(-1px); }
.poster-upload img { width: 100%; height: 100%; object-fit: cover; }
.poster-upload input { position: absolute; inset: 0; opacity: 0; cursor: pointer; }
.poster-upload__placeholder { display: grid; gap: 8px; padding: 24px; }
.poster-upload__placeholder b { color: var(--dew-text-heading); font-size: var(--text-base); }
.poster-upload__placeholder small { font-size: var(--text-xs); line-height: 1.5; }
.poster-upload__action { position: absolute; right: 12px; bottom: 12px; padding: 7px 10px; border: 1px solid var(--dew-card-border); border-radius: var(--radius-full); background: color-mix(in srgb, var(--dew-card-bg) 84%, transparent); backdrop-filter: blur(8px); color: var(--dew-text-heading); font-size: var(--text-xs); font-weight: 600; }
.profile-form { margin-top: var(--space-lg); }
.profile-form :deep(.el-form-item) { margin-bottom: 16px; }
.profile-form :deep(.el-form-item__label) { padding-bottom: 5px; color: var(--dew-text-heading); font-size: var(--text-sm); line-height: 1.3; }
.capacity-rule { align-items: center; justify-content: space-between; margin: 0 0 var(--space-lg); padding: 12px 14px; border: 1px solid var(--dew-card-border); border-radius: var(--radius-md); background: var(--dew-card-inset-bg); }
.capacity-rule div { display: grid; gap: 3px; }
.capacity-rule span { color: var(--dew-text-muted); font-size: var(--text-xs); }
.capacity-rule strong { color: var(--dew-text-heading); font-size: var(--text-sm); }

.student-list { display: flex; flex-direction: column; gap: var(--space-sm); }
.student-list article { display: flex; align-items: flex-start; gap: 10px; padding: 11px; border: 1px solid var(--dew-card-border); border-radius: var(--radius-md); background: var(--dew-card-inset-bg); }
.student-avatar, .profile-initial { display: inline-flex; flex: 0 0 auto; align-items: center; justify-content: center; border-radius: 50%; color: var(--color-primary); background: var(--color-primary-light); font-weight: 700; }
.student-avatar { width: 31px; height: 31px; font-size: var(--text-sm); }
.student-list strong { color: var(--dew-text-heading); font-size: var(--text-sm); }
.student-list p { margin: 3px 0 0; color: var(--dew-text-muted); font-size: var(--text-xs); line-height: 1.5; }
.empty-state { display: grid; justify-items: center; padding: 42px 12px; color: var(--dew-text-muted); text-align: center; }
.empty-orbit { display: inline-flex; align-items: center; justify-content: center; width: 42px; height: 42px; margin-bottom: 10px; border-radius: 50%; background: var(--color-primary-light); color: var(--color-primary); font-size: 18px; box-shadow: inset 0 0 0 6px color-mix(in srgb, var(--color-primary-light) 55%, transparent); }
.empty-state h3 { margin: 0; color: var(--dew-text-heading); font-size: var(--text-base); }
.empty-state p { max-width: 230px; margin: 6px 0 0; font-size: var(--text-sm); line-height: 1.6; }

.choice-summary { align-items: center; gap: var(--space-md); margin-bottom: 28px; padding: 19px 20px; }
.choice-summary__mark { display: inline-flex; flex: 0 0 auto; align-items: center; justify-content: center; width: 36px; height: 36px; border-radius: 50%; color: var(--color-success); background: var(--color-success-light); font-size: 18px; font-weight: 800; }
.choice-summary h2 { margin: 3px 0 4px; color: var(--dew-text-heading); font-size: var(--text-lg); }
.choice-summary p:not(.section-kicker) { margin: 0; color: var(--dew-text-muted); font-size: var(--text-sm); }
.choice-summary :last-child { margin-left: auto; }

.explore-header { align-items: flex-end; justify-content: space-between; gap: var(--space-lg); margin: 6px 0 var(--space-md); }
.explore-meta { display: flex; flex-wrap: wrap; justify-content: flex-end; gap: 6px; }
.profile-grid { display: grid; grid-template-columns: repeat(3, minmax(0, 1fr)); gap: var(--space-lg); }
.profile-card { min-width: 0; animation: profile-reveal .5s both ease-out; animation-delay: calc(var(--reveal-index) * 55ms); }
.poster { position: relative; overflow: hidden; aspect-ratio: 4 / 3; background: var(--dew-card-inset-bg); }
.poster img { display: block; width: 100%; height: 100%; object-fit: cover; transition: transform .45s var(--dew-bounce); }
.profile-card:hover .poster img { transform: scale(1.025); }
.poster-fallback { display: grid; place-content: center; gap: 8px; width: 100%; height: 100%; color: var(--color-primary); text-align: center; background: radial-gradient(circle at 30% 20%, var(--color-primary-light), transparent 55%), var(--dew-card-inset-bg); }
.poster-fallback span { display: inline-flex; align-items: center; justify-content: center; width: 64px; height: 64px; margin: auto; border: 1px solid var(--dew-card-border); border-radius: 50%; background: var(--dew-card-bg); color: var(--dew-text-heading); font-size: 28px; font-weight: 700; }
.poster-fallback small { color: var(--dew-text-muted); font-size: var(--text-xs); }
.poster-status { position: absolute; top: 12px; right: 12px; box-shadow: 0 3px 12px rgba(26, 55, 96, .12); }
.profile-content { padding: 16px 2px 2px; }
.profile-title-row { display: flex; align-items: flex-start; justify-content: space-between; gap: var(--space-sm); }
.profile-title-row h3 { margin: 0; color: var(--dew-text-heading); font-size: var(--text-lg); }
.profile-title-row p { margin: 4px 0 0; color: var(--dew-text-muted); font-size: var(--text-sm); }
.profile-initial { width: 30px; height: 30px; font-size: var(--text-xs); }
.profile-tags { display: flex; flex-wrap: wrap; gap: 6px; min-height: 22px; margin-top: 12px; }
.intro { display: -webkit-box; min-height: 46px; margin: 12px 0 15px; overflow: hidden; color: var(--dew-text-muted); font-size: var(--text-sm); line-height: 1.6; -webkit-box-orient: vertical; -webkit-line-clamp: 2; }
.capacity-block { margin: 0 0 16px; padding: 12px; border: 1px solid var(--dew-card-border); border-radius: var(--radius-md); background: var(--dew-card-inset-bg); }
.capacity-heading { align-items: baseline; justify-content: space-between; margin-bottom: 8px; }
.capacity-heading span, .capacity-block p { color: var(--dew-text-muted); font-size: var(--text-xs); }
.capacity-heading strong { color: var(--dew-text-heading); font-size: var(--text-lg); font-variant-numeric: tabular-nums; }
.capacity-heading em { color: var(--dew-text-muted); font-size: var(--text-xs); font-style: normal; font-weight: 500; }
.capacity-block p { margin: 7px 0 0; }
.empty-list { margin-top: var(--space-lg); }

.choice-dialog { display: grid; gap: 14px; }
.choice-dialog__mentor { align-items: center; gap: 10px; padding: 12px; border: 1px solid var(--dew-card-border); border-radius: var(--radius-md); background: var(--dew-card-inset-bg); }
.choice-dialog__mentor > span { display: inline-flex; align-items: center; justify-content: center; width: 38px; height: 38px; border-radius: 50%; color: var(--color-primary); background: var(--color-primary-light); font-weight: 700; }
.choice-dialog__mentor strong { color: var(--dew-text-heading); }
.choice-dialog__mentor p, .choice-dialog__copy { margin: 3px 0 0; color: var(--dew-text-muted); font-size: var(--text-sm); }
.choice-dialog__copy { margin: 0; line-height: 1.65; }
.choice-dialog__label { display: flex; align-items: baseline; justify-content: space-between; gap: 12px; color: var(--dew-text-heading); font-size: var(--text-sm); font-weight: 600; }
.choice-dialog__label span { color: var(--dew-text-muted); font-size: var(--text-xs); font-weight: 400; }

.page-enter { animation: section-reveal .42s ease-out both; }
@keyframes section-reveal { from { opacity: 0; transform: translateY(10px); } to { opacity: 1; transform: translateY(0); } }
@keyframes profile-reveal { from { opacity: 0; transform: translateY(12px); } to { opacity: 1; transform: translateY(0); } }
@media (prefers-reduced-motion: reduce) { .page-enter, .profile-card { animation: none; } .poster img, .poster-upload { transition: none; } }

@media (max-width: 900px) {
  .hero-content, .explore-header { align-items: flex-start; flex-direction: column; }
  .hero-rules, .explore-meta { justify-content: flex-start; max-width: none; }
  .mentor-workspace { grid-template-columns: 1fr; }
  .profile-grid { grid-template-columns: repeat(2, minmax(0, 1fr)); }
}
@media (max-width: 600px) {
  .mentor-market { padding: var(--space-lg) var(--space-md) 48px; }
  .hero-content { gap: 22px; }
  .hero-description { font-size: var(--text-base); }
  .hero-rules { display: grid; grid-template-columns: 1fr; width: 100%; }
  .choice-summary { align-items: flex-start; flex-wrap: wrap; }
  .choice-summary :last-child { margin-left: 48px; }
  .profile-grid { grid-template-columns: 1fr; }
  .explore-meta { gap: 5px; }
}

/* Second-pass layout: community-oriented mentor matching, not an activity notice. */
.shell { max-width: 1360px; }
.market-hero { position: relative; min-height: 300px; isolation: isolate; }
.market-hero :deep(.dew-card__body) { position: relative; min-height: 264px; padding: 22px 28px 26px; overflow: hidden; }
.hero-topline, .hero-content { position: relative; z-index: 2; }
.hero-back { min-width: 112px; border-color: var(--dew-btn-lit-border) !important; background: var(--dew-btn-lit-bg) !important; box-shadow: var(--dew-btn-lit-shadow) !important; color: var(--dew-btn-lit-color) !important; }
.hero-content { min-height: 194px; padding-top: 34px; padding-right: 310px; }
.hero-description { max-width: 480px; }
.hero-rules { justify-content: flex-start; max-width: none; }
.hero-rules span { color: var(--dew-text-heading); font-weight: 600; }
.hero-constellation { position: absolute; z-index: 1; top: 0; right: 0; width: 330px; height: 100%; overflow: hidden; pointer-events: none; }
.hero-orbit { position: absolute; border: 1px solid color-mix(in srgb, var(--color-primary) 28%, transparent); border-radius: 50%; }
.hero-orbit--one { width: 232px; height: 232px; top: 34px; right: 34px; }
.hero-orbit--two { width: 148px; height: 148px; top: 76px; right: 76px; border-color: color-mix(in srgb, var(--color-success) 34%, transparent); }
.hero-bridge { position: absolute; top: 142px; right: 89px; width: 146px; height: 1px; background: linear-gradient(90deg, transparent, var(--color-primary), var(--color-success), transparent); transform: rotate(-24deg); transform-origin: right center; opacity: .58; }
.hero-node { position: absolute; display: inline-flex; align-items: center; justify-content: center; border: 1px solid var(--dew-card-border); border-radius: 50%; background: color-mix(in srgb, var(--dew-card-bg) 85%, transparent); box-shadow: var(--dew-card-shadow); color: var(--dew-text-heading); font-size: var(--text-xs); font-weight: 700; }
.hero-node--medical { top: 70px; right: 174px; width: 55px; height: 55px; color: var(--color-primary); }
.hero-node--engineering { top: 147px; right: 65px; width: 61px; height: 61px; color: var(--color-success); }
.hero-node--spark { top: 32px; right: 69px; width: 31px; height: 31px; color: var(--color-primary); font-size: 15px; }
.hero-constellation__label { position: absolute; right: 95px; bottom: 32px; color: var(--dew-text-muted); font-size: var(--text-xs); line-height: 1.45; letter-spacing: .04em; text-align: center; }
.hero-constellation__label b { color: var(--dew-text-heading); font-weight: 700; }

.my-match { margin: 0 0 34px; }
.section-title-row { display: flex; align-items: flex-end; justify-content: space-between; gap: var(--space-md); margin-bottom: var(--space-md); }
.section-title-row h2 { margin: 4px 0 0; color: var(--dew-text-heading); font-size: var(--text-xl); }
.my-match-card :deep(.dew-card__body) { display: grid; grid-template-columns: 156px minmax(0, 1fr) minmax(260px, .7fr); align-items: center; gap: 24px; padding: 0; }
.my-match-poster { height: 144px; overflow: hidden; background: var(--dew-card-inset-bg); }
.my-match-poster img { display: block; width: 100%; height: 100%; object-fit: cover; }
.my-match-poster > span { display: flex; align-items: center; justify-content: center; width: 100%; height: 100%; color: var(--color-primary); background: var(--color-primary-light); font-size: 32px; font-weight: 700; }
.my-match-content { padding: 20px 0; }
.my-match-content h3 { margin: 9px 0 0; color: var(--dew-text-heading); font-size: 24px; letter-spacing: -.02em; }
.my-match-content .profile-tags { margin-top: 10px; }
.my-match-message { margin: 14px 0 0; color: var(--dew-text-muted); font-size: var(--text-sm); line-height: 1.55; }
.my-match-capacity { align-self: stretch; display: flex; flex-direction: column; justify-content: center; gap: 8px; padding: 20px 24px 20px 0; }
.my-match-actions { display: flex; justify-content: flex-end; gap: 8px; margin-top: 8px; }

.explore-header { margin-top: 0; padding-top: 2px; }
.explore-header h2 { font-size: 24px; letter-spacing: -.025em; }
.profile-grid { grid-template-columns: repeat(3, minmax(0, 1fr)); gap: 20px; }
.profile-card :deep(.dew-card__body) { display: flex; flex-direction: column; height: 100%; }
.profile-content { display: flex; flex: 1; flex-direction: column; padding: 16px 2px 2px; }
.poster { aspect-ratio: 5 / 3; }
.intro { min-height: 44px; margin-bottom: 14px; }
.capacity-block { margin-bottom: 12px; }
.profile-action { margin-top: auto; }
.profile-action > span { display: block; margin: 0 0 8px; color: var(--dew-text-faint); font-size: var(--text-xs); }
.profile-grid--feature { grid-template-columns: 1fr; }
.profile-grid--feature .profile-card :deep(.dew-card__body) { display: grid; grid-template-columns: minmax(300px, .82fr) minmax(0, 1.18fr); min-height: 318px; padding: 0; }
.profile-grid--feature .poster { min-height: 318px; aspect-ratio: auto; }
.profile-grid--feature .profile-content { padding: 28px 30px; }
.profile-grid--feature .profile-title-row h3 { font-size: 25px; }
.profile-grid--feature .intro { max-width: 650px; min-height: auto; -webkit-line-clamp: 3; }
.profile-grid--feature .capacity-block { max-width: 520px; }
.profile-grid--feature .profile-action { max-width: 520px; }

.capacity-rule { align-items: center; gap: var(--space-md); }
.capacity-rule > div { flex: 1; }
.capacity-rule strong { color: var(--dew-text-heading); font-size: var(--text-sm); }
.capacity-rule small { color: var(--dew-text-muted); font-size: var(--text-xs); line-height: 1.45; }
.capacity-select { width: 96px; flex: 0 0 96px; }
.capacity-select :deep(.el-select__wrapper) { min-height: 38px; border-radius: var(--radius-md); }

@media (max-width: 900px) {
  .market-hero :deep(.dew-card__body) { min-height: 300px; }
  .hero-content { min-height: 226px; padding-right: 235px; }
  .hero-constellation { width: 260px; opacity: .8; }
  .hero-orbit--one { right: 12px; }
  .hero-orbit--two { right: 54px; }
  .hero-node--medical { right: 130px; }
  .hero-node--engineering { right: 24px; }
  .hero-node--spark { right: 27px; }
  .hero-bridge { right: 44px; }
  .hero-constellation__label { right: 42px; }
  .my-match-card :deep(.dew-card__body) { grid-template-columns: 130px minmax(0, 1fr); gap: 18px; }
  .my-match-poster { height: 100%; min-height: 172px; }
  .my-match-capacity { grid-column: 1 / -1; padding: 0 20px 20px; }
  .profile-grid { grid-template-columns: repeat(2, minmax(0, 1fr)); }
}

@media (max-width: 600px) {
  .market-hero { min-height: 0; }
  .market-hero :deep(.dew-card__body) { min-height: 0; padding: 18px; }
  .hero-content { min-height: 0; padding: 30px 0 0; }
  .hero-constellation { top: 34px; right: -72px; width: 220px; height: 200px; opacity: .34; }
  .hero-rules { position: relative; z-index: 2; margin-top: 8px; }
  .hero-rules span { padding: 7px 9px; }
  .section-title-row { align-items: flex-start; }
  .section-title-row h2, .explore-header h2 { font-size: 22px; }
  .my-match-card :deep(.dew-card__body), .profile-grid--feature .profile-card :deep(.dew-card__body) { display: flex; min-height: 0; }
  .my-match-poster, .profile-grid--feature .poster { height: 175px; min-height: 175px; aspect-ratio: 5 / 3; }
  .my-match-content, .profile-grid--feature .profile-content { padding: 20px; }
  .my-match-capacity { padding: 0 20px 20px; }
  .my-match-actions { justify-content: flex-start; flex-wrap: wrap; }
  .profile-grid { grid-template-columns: 1fr; }
}
</style>
