<script setup lang="ts">
import { computed, onMounted, onUnmounted, ref } from 'vue';

const DEFAULT_API_URL = 'https://api.xyzrank.top/v1/stats?sort_by=daily_change&limit=10';
const API_URL = import.meta.env.VITE_API_URL || DEFAULT_API_URL;
const REQUEST_TIMEOUT_MS = 10_000;
const AUTO_REFRESH_INTERVAL_MS = 60 * 60 * 1_000;
const PODCAST_DOMAINS = ['xiaoyuzhoufm.com', 'www.xiaoyuzhoufm.com'];

const STATUS_MESSAGES: Record<number, string> = {
  400: '请求参数有误',
  401: '未授权',
  403: '禁止访问',
  404: '接口不存在',
  429: '请求过于频繁，请稍后重试',
  500: '服务器错误',
};

type Podcast = {
  podcast_id: string;
  title: string;
  url: string;
  latest_count: number;
  daily_change: number;
  monthly_change: number;
  updated_at: string;
};

type RankingResponse = {
  total: number;
  limit: number;
  offset: number;
  data: Podcast[];
};

type SortMode = 'daily_change' | 'growth_rate' | 'momentum';

const BLOCKLIST_KEY = 'xyzrank_blocklist';

function loadBlocklist(): string[] {
  try {
    const raw = localStorage.getItem(BLOCKLIST_KEY);
    return raw ? JSON.parse(raw) as string[] : [];
  } catch {
    return [];
  }
}

function saveBlocklist(list: string[]): void {
  localStorage.setItem(BLOCKLIST_KEY, JSON.stringify(list));
}

const podcasts = ref<Podcast[]>([]);
const loading = ref(true);
const error = ref('');
const sortMode = ref<SortMode>('daily_change');
const blocklist = ref<string[]>(loadBlocklist());
const showBlocklistPanel = ref(false);
const blockInput = ref('');
let autoRefreshTimer: number | undefined;

function addToBlocklist(keyword: string): void {
  const trimmed = keyword.trim();
  if (trimmed && !blocklist.value.includes(trimmed)) {
    blocklist.value.push(trimmed);
    saveBlocklist(blocklist.value);
  }
  blockInput.value = '';
}

function removeFromBlocklist(keyword: string): void {
  blocklist.value = blocklist.value.filter((k) => k !== keyword);
  saveBlocklist(blocklist.value);
}

function blockPodcast(title: string): void {
  addToBlocklist(title);
}

function isBlocked(podcast: Podcast): boolean {
  return blocklist.value.some(
    (keyword) => podcast.title.toLowerCase().includes(keyword.toLowerCase()),
  );
}

const lastFetchTime = ref('');

function growthRate(p: Podcast): number {
  return p.latest_count > 0 ? p.daily_change / p.latest_count : 0;
}

function momentum(p: Podcast): number {
  const avgMonthlyDaily = p.monthly_change / 30;
  const surge = avgMonthlyDaily > 0 ? p.daily_change / avgMonthlyDaily : 1;
  const rate = growthRate(p);
  return rate * 0.5 + (surge / 10) * 0.3 + (p.daily_change / 1000) * 0.2;
}

const sortedPodcasts = computed(() => {
  const list = podcasts.value.filter((p) => !isBlocked(p));
  switch (sortMode.value) {
    case 'growth_rate':
      return list.sort((a, b) => growthRate(b) - growthRate(a));
    case 'momentum':
      return list.sort((a, b) => momentum(b) - momentum(a));
    default:
      return list;
  }
});

function formatPercent(value: number): string {
  return (value * 100).toFixed(2) + '%';
}

async function loadRanking(): Promise<void> {
  loading.value = true;
  error.value = '';

  const controller = new AbortController();
  const timeoutId = window.setTimeout(() => controller.abort(), REQUEST_TIMEOUT_MS);

  try {
    const response = await fetch(API_URL, {
      headers: { accept: 'application/json' },
      signal: controller.signal,
    });

    if (!response.ok) {
      throw new Error(STATUS_MESSAGES[response.status] || `接口返回 ${response.status}`);
    }

    const result: unknown = await response.json();
    if (!validateRankingResponse(result)) {
      throw new Error('API 返回格式不匹配');
    }

    podcasts.value = result.data.slice(0, 10);
    lastFetchTime.value = new Date().toLocaleString('zh-CN', { hour12: false });
  } catch (caught) {
    error.value = getRequestErrorMessage(caught);
  } finally {
    window.clearTimeout(timeoutId);
    loading.value = false;
  }
}

function getRequestErrorMessage(caught: unknown): string {
  if (caught instanceof DOMException && caught.name === 'AbortError') {
    return '请求超时，服务器响应过慢';
  }

  if (caught instanceof TypeError) {
    return '网络连接失败，请检查网络后重试';
  }

  return caught instanceof Error ? caught.message : '获取榜单失败';
}

function validateRankingResponse(data: unknown): data is RankingResponse {
  if (!isRecord(data)) {
    return false;
  }

  return (
    typeof data.total === 'number' &&
    typeof data.limit === 'number' &&
    typeof data.offset === 'number' &&
    Array.isArray(data.data) &&
    data.data.every(isPodcast)
  );
}

function isPodcast(data: unknown): data is Podcast {
  if (!isRecord(data)) {
    return false;
  }

  return (
    typeof data.podcast_id === 'string' &&
    typeof data.title === 'string' &&
    typeof data.url === 'string' &&
    typeof data.latest_count === 'number' &&
    typeof data.daily_change === 'number' &&
    typeof data.monthly_change === 'number' &&
    typeof data.updated_at === 'string'
  );
}

function isRecord(value: unknown): value is Record<string, unknown> {
  return typeof value === 'object' && value !== null;
}

function formatNumber(value: number): string {
  return new Intl.NumberFormat('zh-CN').format(value);
}

function formatGrowth(value: number): string {
  return value > 0 ? `+${formatNumber(value)}` : formatNumber(value);
}

function safePodcastUrl(url: string): string | null {
  try {
    const parsed = new URL(url);
    return ['http:', 'https:'].includes(parsed.protocol) && PODCAST_DOMAINS.includes(parsed.hostname) ? parsed.href : null;
  } catch {
    return null;
  }
}

onMounted(() => {
  void loadRanking();
  autoRefreshTimer = window.setInterval(() => {
    if (!loading.value) {
      void loadRanking();
    }
  }, AUTO_REFRESH_INTERVAL_MS);
});

onUnmounted(() => {
  if (autoRefreshTimer !== undefined) {
    window.clearInterval(autoRefreshTimer);
  }
});
</script>

<template>
  <main class="page-shell">
    <section class="hero" aria-labelledby="page-title">
      <p class="eyebrow">Daily podcast ranking</p>
      <div class="hero-content">
        <div>
          <h1 id="page-title">小宇宙每日增长 Top 10</h1>
          <p class="intro">自动获取小宇宙播客日增长榜，快速发现今天正在升温的中文播客。</p>
        </div>
        <button class="refresh-button" type="button" :disabled="loading" @click="loadRanking">
          {{ loading ? '更新中' : '刷新榜单' }}
        </button>
      </div>
      <div class="meta-row">
        <span v-if="lastFetchTime">数据更新时间：{{ lastFetchTime }}</span>
      </div>
    </section>

    <div class="toolbar">
      <nav class="sort-tabs" aria-label="排序方式">
        <button
          :class="['sort-tab', { active: sortMode === 'daily_change' }]"
          type="button"
          @click="sortMode = 'daily_change'"
        >
          日增量
        </button>
        <button
          :class="['sort-tab', { active: sortMode === 'growth_rate' }]"
          type="button"
          @click="sortMode = 'growth_rate'"
        >
          增长率
        </button>
        <button
          :class="['sort-tab', { active: sortMode === 'momentum' }]"
          type="button"
          @click="sortMode = 'momentum'"
        >
          综合热度
        </button>
      </nav>
      <button class="blocklist-toggle" type="button" @click="showBlocklistPanel = !showBlocklistPanel">
        屏蔽管理 ({{ blocklist.length }})
      </button>
    </div>

    <section v-if="showBlocklistPanel" class="blocklist-panel">
      <h3>屏蔽列表</h3>
      <p class="blocklist-hint">标题包含以下关键词的播客将被隐藏</p>
      <form class="blocklist-form" @submit.prevent="addToBlocklist(blockInput)">
        <input
          v-model="blockInput"
          class="blocklist-input"
          type="text"
          placeholder="输入关键词，如播客名称"
        />
        <button class="blocklist-add" type="submit" :disabled="!blockInput.trim()">添加</button>
      </form>
      <ul v-if="blocklist.length" class="blocklist-tags">
        <li v-for="keyword in blocklist" :key="keyword" class="blocklist-tag">
          {{ keyword }}
          <button type="button" class="blocklist-remove" @click="removeFromBlocklist(keyword)">&times;</button>
        </li>
      </ul>
      <p v-else class="blocklist-empty">暂无屏蔽项</p>
    </section>

    <section class="panel" aria-live="polite">
      <div v-if="loading" class="state-card" role="status" aria-live="assertive">
        <span class="loader" aria-hidden="true"></span>
        <span>正在获取今天的小宇宙榜单...</span>
      </div>

      <div v-else-if="error" class="state-card error-card" role="alert">
        <strong>暂时无法获取榜单</strong>
        <span>{{ error }}</span>
        <button class="secondary-button" type="button" @click="loadRanking">重试</button>
      </div>

      <div v-else-if="podcasts.length === 0" class="state-card">今天还没有可展示的数据。</div>

      <ol v-else class="rank-list">
        <li
          v-for="(podcast, index) in sortedPodcasts"
          :key="podcast.podcast_id"
          class="rank-card"
          :aria-label="`第 ${index + 1} 位：${podcast.title}，日增长 ${formatGrowth(podcast.daily_change)}`"
        >
          <div class="rank-number" aria-label="排名">{{ index + 1 }}</div>
          <div class="podcast-main">
            <h2>{{ podcast.title }}</h2>
            <div class="stats-grid">
              <div>
                <span>日增长</span>
                <strong class="growth">{{ formatGrowth(podcast.daily_change) }}</strong>
              </div>
              <div>
                <span>增长率</span>
                <strong class="growth-rate">{{ formatPercent(growthRate(podcast)) }}</strong>
              </div>
              <div>
                <span>订阅数</span>
                <strong>{{ formatNumber(podcast.latest_count) }}</strong>
              </div>
              <div>
                <span>月增长</span>
                <strong>{{ formatGrowth(podcast.monthly_change) }}</strong>
              </div>
            </div>
          </div>
          <div class="card-actions">
            <a
              v-if="safePodcastUrl(podcast.url)"
              class="podcast-link"
              :href="safePodcastUrl(podcast.url) ?? undefined"
              target="_blank"
              rel="noreferrer noopener"
            >
              打开小宇宙
            </a>
            <button class="block-btn" type="button" title="屏蔽此播客" @click="blockPodcast(podcast.title)">
              &times;
            </button>
          </div>
        </li>
      </ol>
    </section>
  </main>
</template>
