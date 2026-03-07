<script setup lang="ts">
import { computed, ref } from 'vue'
import { RouterLink } from 'vue-router'

type QueryParam = {
  key: string
  value: string
}

type RegexMatch = {
  index: number
  text: string
  groups: string[]
}

type OutputTone = 'default' | 'success' | 'error'

type CopyStatus = {
  key: string
  state: OutputTone
  message: string
}

const jsonInput = ref('{\n  "name": "J2TEAM",\n  "tools": ["json", "base64", "regex"],\n  "active": true\n}')
const base64Text = ref('Xin chào J2TEAM')
const base64Value = ref('WGluIGNow6BvIEoyVEVBTQ==')
const urlInput = ref('https://example.com/search?q=vibe%20code&lang=vi&sort=newest')
const timestampInput = ref(Math.floor(Date.now() / 1000).toString())
const dateTimeInput = ref(formatDateTimeLocal(new Date()))
const regexPattern = ref('\\b(dev|tool)\\b')
const regexFlags = ref('gi')
const regexSample = ref('Dev Pocket Tools giúp dev kiểm tra regex ngay trong trình duyệt.')
const slugInput = ref('Bộ công cụ bỏ túi dành cho lập trình viên Việt Nam')
const copyStatus = ref<CopyStatus | null>(null)

const jsonResult = computed(() => {
  const source = jsonInput.value.trim()

  if (!source) {
    return { tone: 'default' as const, message: 'Dán JSON thô vào đây để format hoặc kiểm tra cú pháp.', output: '' }
  }

  try {
    const parsed: object = JSON.parse(source) as object
    return { tone: 'success' as const, message: 'JSON hợp lệ.', output: JSON.stringify(parsed, null, 2) }
  } catch (error) {
    return {
      tone: 'error' as const,
      message: error instanceof Error ? error.message : 'Không thể phân tích JSON.',
      output: '',
    }
  }
})

const urlResult = computed(() => {
  const source = urlInput.value.trim()

  if (!source) {
    return {
      tone: 'default' as const,
      encoded: '',
      decoded: '',
      origin: '',
      pathname: '',
      queryParams: [] as QueryParam[],
      message: 'Nhập URL đầy đủ hoặc chỉ phần query để encode, decode và parse.',
    }
  }

  const encoded = encodeURIComponent(source)
  let decoded = source

  try {
    decoded = decodeURIComponent(source)
  } catch {
    decoded = 'Chuỗi hiện tại không phải dữ liệu URL-encoded hợp lệ để decode toàn phần.'
  }

  try {
    const url = normalizeUrlInput(source)
    return {
      tone: 'success' as const,
      encoded,
      decoded,
      origin: url.origin,
      pathname: `${url.pathname}${url.hash}`,
      queryParams: Array.from(url.searchParams.entries()).map(([key, value]) => ({ key, value })),
      message: url.searchParams.size ? `Đã parse ${url.searchParams.size} query param.` : 'URL hợp lệ nhưng không có query param.',
    }
  } catch (error) {
    return {
      tone: 'error' as const,
      encoded,
      decoded,
      origin: '',
      pathname: '',
      queryParams: [] as QueryParam[],
      message: error instanceof Error ? error.message : 'Không thể phân tích URL.',
    }
  }
})

const timestampResult = computed(() => {
  const source = timestampInput.value.trim()

  if (!source) {
    return { tone: 'default' as const, normalized: '', iso: '', local: '', message: 'Nhập Unix timestamp theo giây hoặc mili giây.' }
  }

  const numericValue = Number(source)
  if (!Number.isFinite(numericValue)) {
    return { tone: 'error' as const, normalized: '', iso: '', local: '', message: 'Timestamp phải là số hợp lệ.' }
  }

  const milliseconds = source.length <= 10 ? numericValue * 1000 : numericValue
  const date = new Date(milliseconds)
  if (Number.isNaN(date.getTime())) {
    return { tone: 'error' as const, normalized: '', iso: '', local: '', message: 'Timestamp nằm ngoài phạm vi ngày giờ hợp lệ.' }
  }

  return {
    tone: 'success' as const,
    normalized: Math.floor(milliseconds / 1000).toString(),
    iso: date.toISOString(),
    local: date.toLocaleString('vi-VN'),
    message: source.length <= 10 ? 'Đang đọc dưới dạng giây.' : 'Đang đọc dưới dạng mili giây.',
  }
})

const dateTimeToTimestamp = computed(() => {
  if (!dateTimeInput.value) {
    return { tone: 'default' as const, seconds: '', milliseconds: '', message: 'Chọn ngày giờ để đổi ngược sang timestamp.' }
  }

  const date = new Date(dateTimeInput.value)
  if (Number.isNaN(date.getTime())) {
    return { tone: 'error' as const, seconds: '', milliseconds: '', message: 'Ngày giờ không hợp lệ.' }
  }

  return {
    tone: 'success' as const,
    seconds: Math.floor(date.getTime() / 1000).toString(),
    milliseconds: date.getTime().toString(),
    message: 'Đã chuyển từ ngày giờ cục bộ sang Unix timestamp.',
  }
})

const regexResult = computed(() => {
  if (!regexPattern.value) {
    return { tone: 'default' as const, message: 'Nhập pattern và flags để kiểm tra regex.', matches: [] as RegexMatch[] }
  }

  try {
    const expression = new RegExp(regexPattern.value, regexFlags.value)
    const matches = collectRegexMatches(expression, regexSample.value)
    return {
      tone: matches.length ? ('success' as const) : ('default' as const),
      message: matches.length ? `Tìm thấy ${matches.length} match.` : 'Không có đoạn nào khớp.',
      matches,
    }
  } catch (error) {
    return {
      tone: 'error' as const,
      message: error instanceof Error ? error.message : 'Regex không hợp lệ.',
      matches: [] as RegexMatch[],
    }
  }
})

const slugResult = computed(() => {
  const source = slugInput.value.trim()
  if (!source) {
    return { tone: 'default' as const, slug: '', message: 'Nhập tiêu đề hoặc cụm từ để tạo slug.' }
  }

  const slug = toSlug(source)
  return {
    tone: slug ? ('success' as const) : ('error' as const),
    slug,
    message: slug ? 'Slug đã sẵn sàng để dùng trong URL.' : 'Không thể tạo slug từ nội dung hiện tại.',
  }
})

function formatJson() {
  if (jsonResult.value.output) jsonInput.value = jsonResult.value.output
}

function encodeBase64() {
  base64Value.value = encodeTextToBase64(base64Text.value)
}

function decodeBase64() {
  try {
    base64Text.value = decodeBase64ToText(base64Value.value)
  } catch (error) {
    setCopyStatus('base64', 'error', error instanceof Error ? error.message : 'Base64 không hợp lệ.')
  }
}

async function copyText(key: string, value: string) {
  if (!value) {
    setCopyStatus(key, 'error', 'Chưa có dữ liệu để copy.')
    return
  }

  try {
    await navigator.clipboard.writeText(value)
    setCopyStatus(key, 'success', 'Đã copy vào clipboard.')
  } catch {
    setCopyStatus(key, 'error', 'Trình duyệt không cho phép copy tự động.')
  }
}

function setTimestampNow() {
  timestampInput.value = Math.floor(Date.now() / 1000).toString()
  dateTimeInput.value = formatDateTimeLocal(new Date())
}

function useConvertedTimestamp() {
  if (dateTimeToTimestamp.value.seconds) timestampInput.value = dateTimeToTimestamp.value.seconds
}

function useDecodedUrl() {
  if (urlResult.value.decoded && !urlResult.value.decoded.startsWith('Chuỗi hiện tại')) urlInput.value = urlResult.value.decoded
}

let copyTimer = 0

function setCopyStatus(key: string, state: OutputTone, message: string) {
  copyStatus.value = { key, state, message }
  window.clearTimeout(copyTimer)
  copyTimer = window.setTimeout(() => {
    copyStatus.value = null
  }, 2200)
}

function normalizeUrlInput(source: string) {
  if (source.startsWith('?')) return new URL(source, 'https://example.dev')
  try {
    return new URL(source)
  } catch {
    return new URL(source.startsWith('/') ? source : `/${source}`, 'https://example.dev')
  }
}

function collectRegexMatches(expression: RegExp, sample: string) {
  if (!sample) return [] as RegexMatch[]
  if (expression.global) {
    return Array.from(sample.matchAll(expression)).map((match) => ({
      index: match.index ?? 0,
      text: match[0],
      groups: match.slice(1).filter((group): group is string => typeof group === 'string'),
    }))
  }

  const firstMatch = expression.exec(sample)
  if (!firstMatch) return [] as RegexMatch[]

  return [{ index: firstMatch.index ?? 0, text: firstMatch[0], groups: firstMatch.slice(1).filter((group): group is string => typeof group === 'string') }]
}

function encodeTextToBase64(value: string) {
  const bytes = new TextEncoder().encode(value)
  let binary = ''
  bytes.forEach((byte) => {
    binary += String.fromCharCode(byte)
  })
  return btoa(binary)
}

function decodeBase64ToText(value: string) {
  const binary = atob(value.trim())
  const bytes = Uint8Array.from(binary, (character) => character.charCodeAt(0))
  return new TextDecoder().decode(bytes)
}

function toSlug(value: string) {
  return value
    .normalize('NFD')
    .replace(/[\u0300-\u036f]/g, '')
    .replace(/đ/g, 'd')
    .replace(/Đ/g, 'd')
    .toLowerCase()
    .replace(/[^a-z0-9]+/g, '-')
    .replace(/^-+|-+$/g, '')
}

function formatDateTimeLocal(date: Date) {
  const year = date.getFullYear()
  const month = String(date.getMonth() + 1).padStart(2, '0')
  const day = String(date.getDate()).padStart(2, '0')
  const hours = String(date.getHours()).padStart(2, '0')
  const minutes = String(date.getMinutes()).padStart(2, '0')
  return `${year}-${month}-${day}T${hours}:${minutes}`
}
</script>

<template>
  <div class="min-h-screen bg-bg-deep text-text-primary">
    <div class="mx-auto flex max-w-6xl flex-col px-4 py-6 sm:px-6 lg:px-8">
      <header class="animate-fade-up border border-border-default bg-bg-surface p-5 sm:p-8">
        <div class="flex flex-col gap-6 lg:flex-row lg:items-start lg:justify-between">
          <div class="max-w-3xl">
            <RouterLink
              to="/"
              class="inline-flex items-center gap-2 border border-border-default px-4 py-2 text-sm text-text-secondary transition hover:border-accent-coral hover:bg-bg-elevated hover:text-text-primary"
            >
              &larr; Về trang chủ
            </RouterLink>

            <p class="mt-6 font-display text-sm tracking-[0.25em] text-accent-coral">// DEV TOOLKIT</p>
            <h1 class="mt-4 font-display text-4xl font-bold tracking-tight text-text-primary sm:text-6xl lg:text-7xl">
              Dev Pocket Tools
            </h1>
            <p class="mt-4 max-w-2xl text-base leading-7 text-text-secondary sm:text-lg">
              Một bộ công cụ bỏ túi cho những việc lặp đi lặp lại trong ngày của dev: format JSON, mã hóa Base64,
              bóc tách URL, đổi timestamp, test regex và tạo slug sạch cho URL.
            </p>
            <div class="mt-6 flex flex-wrap gap-3 text-xs font-medium tracking-wide text-text-secondary">
              <span class="border border-border-default px-3 py-1">100% client-side</span>
              <span class="border border-border-default px-3 py-1">Không gửi dữ liệu đi đâu</span>
              <span class="border border-border-default px-3 py-1">Không cần dependency mới</span>
            </div>
          </div>

          <div class="inline-flex w-fit self-start bg-accent-coral px-4 py-2 font-display text-xs font-bold tracking-[0.25em] text-bg-deep rotate-2">
            VOL.01 / 2026
          </div>
        </div>

        <div class="mt-8 flex gap-1.5">
          <span v-for="n in 36" :key="n" class="h-1 w-1 rounded-full bg-border-default" />
        </div>
      </header>

      <section class="animate-fade-up animate-delay-2 mt-8">
        <h2 class="mb-6 flex items-center gap-3 font-display text-2xl font-semibold text-text-primary">
          <span class="font-display text-sm tracking-widest text-accent-coral">//</span>
          Bộ công cụ
        </h2>

        <div class="grid gap-5 lg:grid-cols-2">
          <article class="animate-fade-up animate-delay-2 border border-border-default bg-bg-surface p-5 transition-all duration-300 hover:-translate-y-1 hover:border-accent-coral hover:bg-bg-elevated hover:shadow-lg hover:shadow-accent-coral/5">
            <div class="flex items-start justify-between gap-4">
              <div>
                <p class="font-display text-xs tracking-[0.25em] text-accent-coral">01</p>
                <h3 class="mt-2 font-display text-xl font-semibold">JSON Formatter / Validator</h3>
                <p class="mt-2 text-sm leading-6 text-text-secondary">Dán JSON thô để kiểm tra cú pháp, format lại cho dễ đọc và copy nhanh kết quả.</p>
              </div>
              <button type="button" class="border border-border-default px-3 py-2 text-xs text-text-secondary transition hover:border-accent-coral hover:text-text-primary" @click="copyText('json', jsonResult.output)">Copy</button>
            </div>

            <label class="mt-5 block text-xs font-display tracking-[0.2em] text-text-dim">INPUT</label>
            <textarea v-model="jsonInput" rows="10" spellcheck="false" class="mt-2 min-h-44 w-full border border-border-default bg-bg-deep px-4 py-3 font-mono text-sm text-text-primary outline-none transition focus:border-accent-coral" />

            <div class="mt-4 flex flex-wrap gap-3">
              <button type="button" class="border border-accent-coral bg-accent-coral px-4 py-2 text-sm font-medium text-bg-deep transition hover:opacity-90" @click="formatJson">Format JSON</button>
            </div>

            <p class="mt-4 text-sm" :class="{ 'text-text-secondary': jsonResult.tone === 'default', 'text-accent-sky': jsonResult.tone === 'success', 'text-accent-amber': jsonResult.tone === 'error' }">
              {{ jsonResult.message }}
            </p>

            <label class="mt-4 block text-xs font-display tracking-[0.2em] text-text-dim">OUTPUT</label>
            <pre class="mt-2 min-h-32 overflow-x-auto border border-border-default bg-bg-deep px-4 py-3 text-sm leading-6 whitespace-pre-wrap"><code>{{ jsonResult.output || 'Kết quả formatted JSON sẽ hiện ở đây.' }}</code></pre>
          </article>

          <article class="animate-fade-up animate-delay-3 border border-border-default bg-bg-surface p-5 transition-all duration-300 hover:-translate-y-1 hover:border-accent-coral hover:bg-bg-elevated hover:shadow-lg hover:shadow-accent-coral/5">
            <div class="flex items-start justify-between gap-4">
              <div>
                <p class="font-display text-xs tracking-[0.25em] text-accent-amber">02</p>
                <h3 class="mt-2 font-display text-xl font-semibold">Base64 Encode / Decode</h3>
                <p class="mt-2 text-sm leading-6 text-text-secondary">Mã hóa hoặc giải mã chuỗi text có Unicode mà không cần rời khỏi trình duyệt.</p>
              </div>
              <button type="button" class="border border-border-default px-3 py-2 text-xs text-text-secondary transition hover:border-accent-coral hover:text-text-primary" @click="copyText('base64', base64Value)">Copy Base64</button>
            </div>

            <label class="mt-5 block text-xs font-display tracking-[0.2em] text-text-dim">TEXT</label>
            <textarea v-model="base64Text" rows="4" spellcheck="false" class="mt-2 w-full border border-border-default bg-bg-deep px-4 py-3 text-sm text-text-primary outline-none transition focus:border-accent-coral" />

            <div class="mt-4 flex flex-wrap gap-3">
              <button type="button" class="border border-accent-coral bg-accent-coral px-4 py-2 text-sm font-medium text-bg-deep transition hover:opacity-90" @click="encodeBase64">Encode</button>
              <button type="button" class="border border-border-default px-4 py-2 text-sm text-text-secondary transition hover:border-accent-amber hover:text-text-primary" @click="decodeBase64">Decode</button>
            </div>

            <label class="mt-4 block text-xs font-display tracking-[0.2em] text-text-dim">BASE64</label>
            <textarea v-model="base64Value" rows="4" spellcheck="false" class="mt-2 w-full border border-border-default bg-bg-deep px-4 py-3 text-sm text-text-primary outline-none transition focus:border-accent-coral" />
          </article>

          <article class="animate-fade-up animate-delay-4 border border-border-default bg-bg-surface p-5 transition-all duration-300 hover:-translate-y-1 hover:border-accent-coral hover:bg-bg-elevated hover:shadow-lg hover:shadow-accent-coral/5">
            <div class="flex items-start justify-between gap-4">
              <div>
                <p class="font-display text-xs tracking-[0.25em] text-accent-sky">03</p>
                <h3 class="mt-2 font-display text-xl font-semibold">URL Encode / Decode + Parser</h3>
                <p class="mt-2 text-sm leading-6 text-text-secondary">Encode, decode và bóc tách query params để nhìn rõ dữ liệu đang được truyền trên URL.</p>
              </div>
              <button type="button" class="border border-border-default px-3 py-2 text-xs text-text-secondary transition hover:border-accent-coral hover:text-text-primary" @click="copyText('url', urlResult.encoded)">Copy Encoded</button>
            </div>

            <label class="mt-5 block text-xs font-display tracking-[0.2em] text-text-dim">URL OR QUERY</label>
            <textarea v-model="urlInput" rows="5" spellcheck="false" class="mt-2 w-full border border-border-default bg-bg-deep px-4 py-3 text-sm text-text-primary outline-none transition focus:border-accent-coral" />

            <div class="mt-4 flex flex-wrap gap-3">
              <button type="button" class="border border-border-default px-4 py-2 text-sm text-text-secondary transition hover:border-accent-sky hover:text-text-primary" @click="useDecodedUrl">Dùng bản decoded</button>
            </div>

            <p class="mt-4 text-sm" :class="{ 'text-text-secondary': urlResult.tone === 'default', 'text-accent-sky': urlResult.tone === 'success', 'text-accent-amber': urlResult.tone === 'error' }">
              {{ urlResult.message }}
            </p>

            <div class="mt-4 grid gap-4 sm:grid-cols-2">
              <div class="border border-border-default bg-bg-deep px-4 py-3">
                <p class="text-xs font-display tracking-[0.2em] text-text-dim">ENCODED</p>
                <p class="mt-2 break-all text-sm">{{ urlResult.encoded || '...' }}</p>
              </div>
              <div class="border border-border-default bg-bg-deep px-4 py-3">
                <p class="text-xs font-display tracking-[0.2em] text-text-dim">DECODED</p>
                <p class="mt-2 break-all text-sm">{{ urlResult.decoded || '...' }}</p>
              </div>
            </div>

            <div class="mt-4 border border-border-default bg-bg-deep px-4 py-3">
              <p class="text-xs font-display tracking-[0.2em] text-text-dim">PATH</p>
              <p class="mt-2 text-sm text-text-secondary">{{ urlResult.origin || 'N/A' }}{{ urlResult.pathname }}</p>
            </div>

            <div class="mt-4 border border-border-default bg-bg-deep px-4 py-3">
              <p class="text-xs font-display tracking-[0.2em] text-text-dim">QUERY PARAMS</p>
              <div v-if="urlResult.queryParams.length" class="mt-3 space-y-2">
                <div v-for="param in urlResult.queryParams" :key="`${param.key}-${param.value}`" class="flex flex-col gap-1 border border-border-default px-3 py-2 text-sm sm:flex-row sm:items-center sm:justify-between">
                  <span class="text-accent-amber">{{ param.key }}</span>
                  <span class="break-all text-text-secondary">{{ param.value }}</span>
                </div>
              </div>
              <p v-else class="mt-2 text-sm text-text-secondary">Chưa có query param để hiển thị.</p>
            </div>
          </article>

          <article class="animate-fade-up animate-delay-5 border border-border-default bg-bg-surface p-5 transition-all duration-300 hover:-translate-y-1 hover:border-accent-coral hover:bg-bg-elevated hover:shadow-lg hover:shadow-accent-coral/5">
            <div class="flex items-start justify-between gap-4">
              <div>
                <p class="font-display text-xs tracking-[0.25em] text-accent-coral">04</p>
                <h3 class="mt-2 font-display text-xl font-semibold">Timestamp Converter</h3>
                <p class="mt-2 text-sm leading-6 text-text-secondary">Đổi qua lại giữa Unix timestamp và ngày giờ đọc được ngay trên máy của bạn.</p>
              </div>
              <button type="button" class="border border-border-default px-3 py-2 text-xs text-text-secondary transition hover:border-accent-coral hover:text-text-primary" @click="copyText('timestamp', timestampResult.iso)">Copy ISO</button>
            </div>

            <label class="mt-5 block text-xs font-display tracking-[0.2em] text-text-dim">TIMESTAMP</label>
            <div class="mt-2 flex flex-col gap-3 sm:flex-row">
              <input v-model="timestampInput" type="text" class="min-w-0 flex-1 border border-border-default bg-bg-deep px-4 py-3 text-sm text-text-primary outline-none transition focus:border-accent-coral" />
              <button type="button" class="border border-border-default px-4 py-3 text-sm text-text-secondary transition hover:border-accent-coral hover:text-text-primary" @click="setTimestampNow">Dùng thời gian hiện tại</button>
            </div>

            <p class="mt-4 text-sm" :class="{ 'text-text-secondary': timestampResult.tone === 'default', 'text-accent-sky': timestampResult.tone === 'success', 'text-accent-amber': timestampResult.tone === 'error' }">
              {{ timestampResult.message }}
            </p>

            <div class="mt-4 grid gap-4 sm:grid-cols-2">
              <div class="border border-border-default bg-bg-deep px-4 py-3">
                <p class="text-xs font-display tracking-[0.2em] text-text-dim">ISO</p>
                <p class="mt-2 break-all text-sm">{{ timestampResult.iso || '...' }}</p>
              </div>
              <div class="border border-border-default bg-bg-deep px-4 py-3">
                <p class="text-xs font-display tracking-[0.2em] text-text-dim">LOCAL</p>
                <p class="mt-2 text-sm">{{ timestampResult.local || '...' }}</p>
              </div>
            </div>

            <label class="mt-5 block text-xs font-display tracking-[0.2em] text-text-dim">DATETIME LOCAL</label>
            <div class="mt-2 flex flex-col gap-3 sm:flex-row">
              <input v-model="dateTimeInput" type="datetime-local" class="min-w-0 flex-1 border border-border-default bg-bg-deep px-4 py-3 text-sm text-text-primary outline-none transition focus:border-accent-coral" />
              <button type="button" class="border border-border-default px-4 py-3 text-sm text-text-secondary transition hover:border-accent-amber hover:text-text-primary" @click="useConvertedTimestamp">Dùng timestamp này</button>
            </div>

            <div class="mt-4 grid gap-4 sm:grid-cols-2">
              <div class="border border-border-default bg-bg-deep px-4 py-3">
                <p class="text-xs font-display tracking-[0.2em] text-text-dim">SECONDS</p>
                <p class="mt-2 text-sm">{{ dateTimeToTimestamp.seconds || '...' }}</p>
              </div>
              <div class="border border-border-default bg-bg-deep px-4 py-3">
                <p class="text-xs font-display tracking-[0.2em] text-text-dim">MILLISECONDS</p>
                <p class="mt-2 text-sm">{{ dateTimeToTimestamp.milliseconds || '...' }}</p>
              </div>
            </div>
          </article>

          <article class="animate-fade-up animate-delay-6 border border-border-default bg-bg-surface p-5 transition-all duration-300 hover:-translate-y-1 hover:border-accent-coral hover:bg-bg-elevated hover:shadow-lg hover:shadow-accent-coral/5">
            <div class="flex items-start justify-between gap-4">
              <div>
                <p class="font-display text-xs tracking-[0.25em] text-accent-amber">05</p>
                <h3 class="mt-2 font-display text-xl font-semibold">Regex Tester</h3>
                <p class="mt-2 text-sm leading-6 text-text-secondary">Nhập pattern, flags và sample text để kiểm tra match hoặc lỗi cú pháp ngay lập tức.</p>
              </div>
              <button type="button" class="border border-border-default px-3 py-2 text-xs text-text-secondary transition hover:border-accent-coral hover:text-text-primary" @click="copyText('regex', regexPattern)">Copy Pattern</button>
            </div>

            <div class="mt-5 grid gap-4 sm:grid-cols-[minmax(0,1fr)_120px]">
              <div>
                <label class="block text-xs font-display tracking-[0.2em] text-text-dim">PATTERN</label>
                <input v-model="regexPattern" type="text" spellcheck="false" class="mt-2 w-full border border-border-default bg-bg-deep px-4 py-3 text-sm text-text-primary outline-none transition focus:border-accent-coral" />
              </div>
              <div>
                <label class="block text-xs font-display tracking-[0.2em] text-text-dim">FLAGS</label>
                <input v-model="regexFlags" type="text" spellcheck="false" class="mt-2 w-full border border-border-default bg-bg-deep px-4 py-3 text-sm text-text-primary outline-none transition focus:border-accent-coral" />
              </div>
            </div>

            <label class="mt-4 block text-xs font-display tracking-[0.2em] text-text-dim">SAMPLE TEXT</label>
            <textarea v-model="regexSample" rows="6" spellcheck="false" class="mt-2 w-full border border-border-default bg-bg-deep px-4 py-3 text-sm text-text-primary outline-none transition focus:border-accent-coral" />

            <p class="mt-4 text-sm" :class="{ 'text-text-secondary': regexResult.tone === 'default', 'text-accent-sky': regexResult.tone === 'success', 'text-accent-amber': regexResult.tone === 'error' }">
              {{ regexResult.message }}
            </p>

            <div class="mt-4 border border-border-default bg-bg-deep px-4 py-3">
              <p class="text-xs font-display tracking-[0.2em] text-text-dim">MATCHES</p>
              <div v-if="regexResult.matches.length" class="mt-3 space-y-2">
                <div v-for="match in regexResult.matches" :key="`${match.index}-${match.text}`" class="border border-border-default px-3 py-2 text-sm">
                  <p><span class="text-accent-coral">Match:</span> {{ match.text }}</p>
                  <p class="mt-1 text-text-secondary">Vị trí: {{ match.index }}</p>
                  <p class="mt-1 text-text-secondary">Groups: {{ match.groups.length ? match.groups.join(', ') : 'Không có capture group.' }}</p>
                </div>
              </div>
              <p v-else class="mt-2 text-sm text-text-secondary">Chưa có match để hiển thị.</p>
            </div>
          </article>

          <article class="animate-fade-up animate-delay-7 border border-border-default bg-bg-surface p-5 transition-all duration-300 hover:-translate-y-1 hover:border-accent-coral hover:bg-bg-elevated hover:shadow-lg hover:shadow-accent-coral/5">
            <div class="flex items-start justify-between gap-4">
              <div>
                <p class="font-display text-xs tracking-[0.25em] text-accent-sky">06</p>
                <h3 class="mt-2 font-display text-xl font-semibold">Slug Generator</h3>
                <p class="mt-2 text-sm leading-6 text-text-secondary">Chuyển tiêu đề có dấu sang slug ASCII ổn định, sạch và thân thiện với URL.</p>
              </div>
              <button type="button" class="border border-border-default px-3 py-2 text-xs text-text-secondary transition hover:border-accent-coral hover:text-text-primary" @click="copyText('slug', slugResult.slug)">Copy Slug</button>
            </div>

            <label class="mt-5 block text-xs font-display tracking-[0.2em] text-text-dim">TEXT</label>
            <textarea v-model="slugInput" rows="5" spellcheck="false" class="mt-2 w-full border border-border-default bg-bg-deep px-4 py-3 text-sm text-text-primary outline-none transition focus:border-accent-coral" />

            <p class="mt-4 text-sm" :class="{ 'text-text-secondary': slugResult.tone === 'default', 'text-accent-sky': slugResult.tone === 'success', 'text-accent-amber': slugResult.tone === 'error' }">
              {{ slugResult.message }}
            </p>

            <div class="mt-4 border border-border-default bg-bg-deep px-4 py-3">
              <p class="text-xs font-display tracking-[0.2em] text-text-dim">SLUG</p>
              <p class="mt-2 break-all font-mono text-sm">{{ slugResult.slug || 'slug-se-hien-o-day' }}</p>
            </div>
          </article>
        </div>
      </section>

      <section class="animate-fade-up animate-delay-3 mt-10 border border-border-default bg-bg-surface p-5 sm:p-6">
        <h2 class="mb-4 flex items-center gap-3 font-display text-2xl font-semibold text-text-primary">
          <span class="font-display text-sm tracking-widest text-accent-sky">//</span>
          Quy ước xử lý
        </h2>
        <p class="max-w-3xl text-sm leading-7 text-text-secondary sm:text-base">
          Mọi thao tác trên trang này đều chạy trực tiếp trong trình duyệt. Không có network call, không lưu backend, không gửi dữ liệu của bạn ra ngoài. Đây là một utility hub self-contained đúng theo tinh thần của repo.
        </p>

        <div v-if="copyStatus" class="mt-4 border px-4 py-3 text-sm" :class="{ 'border-border-default text-text-secondary': copyStatus.state === 'default', 'border-accent-sky text-accent-sky': copyStatus.state === 'success', 'border-accent-amber text-accent-amber': copyStatus.state === 'error' }">
          {{ copyStatus.message }}
        </div>

        <RouterLink
          to="/"
          class="mt-6 inline-flex items-center gap-2 border border-border-default px-4 py-2 text-sm text-text-secondary transition hover:border-accent-coral hover:bg-bg-elevated hover:text-text-primary"
        >
          &larr; Quay lại launcher
        </RouterLink>
      </section>
    </div>
  </div>
</template>
