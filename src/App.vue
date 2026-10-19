<script setup>
import { computed, nextTick, onBeforeUnmount, onMounted, ref, watch } from 'vue'
import { Menu, X } from 'lucide-vue-next'
import HomePage from './components/HomePage.vue'
import MarketWorkspace from './components/MarketWorkspace.vue'
import DocsPage from './components/DocsPage.vue'
import ScenarioLab from './components/ScenarioLab.vue'
import SolanaWalletButton from './components/SolanaWalletButton.vue'
import { brand } from './brand.js'

const pathname = ref(location.pathname.replace(/\/$/, '') || '/')
const railOpen = ref(false)
const appPage = computed(() => pathname.value.startsWith('/app/') || pathname.value === '/stats')

const nav = [
  ['01', 'Overview', '/', 'tape'],
  ['02', 'Board', '/app/markets', 'watchlist'],
  ['03', 'Journal', '/signal', 'local'],
  ['04', 'Reference', '/docs', 'method'],
  ['05', 'Boundary', '/#cap', 'model'],
  ['06', 'Limits', '/risk', 'disclosure'],
]
const colophonLinks = [
  ['Board', '/app/markets'],
  ['Journal', '/signal'],
  ['Reference', '/docs'],
  ['Limits', '/risk'],
  ['Terms', '/terms'],
  ['Privacy', '/privacy'],
]

const isCurrent = to => {
  if (to === '/') return pathname.value === '/'
  if (to === '/app/markets') return pathname.value.startsWith('/app/') || pathname.value === '/stats'
  if (to === '/signal') return pathname.value === '/signal'
  if (to === '/docs') return ['/docs'].includes(pathname.value)
  if (to === '/risk') return ['/risk', '/terms', '/privacy'].includes(pathname.value)
  return false
}

function scrollToHash() {
  if (location.hash) document.getElementById(decodeURIComponent(location.hash.slice(1)))?.scrollIntoView({ behavior: 'instant' })
}

async function navigate(to) {
  railOpen.value = false
  const url = new URL(to, location.origin)
  if (url.pathname === location.pathname && url.hash) {
    history.pushState({}, '', url.pathname + url.hash)
    await nextTick()
    scrollToHash()
    return
  }
  history.pushState({}, '', url.pathname + url.hash)
  pathname.value = url.pathname.replace(/\/$/, '') || '/'
  await nextTick()
  if (url.hash) scrollToHash()
  else {
    window.scrollTo({ top: 0, behavior: 'instant' })
    document.querySelector('main h1')?.focus({ preventScroll: true })
  }
}

async function popState() {
  pathname.value = location.pathname.replace(/\/$/, '') || '/'
  railOpen.value = false
  await nextTick()
  scrollToHash()
}

function onKeydown(event) {
  if (event.key === 'Escape' && railOpen.value) {
    railOpen.value = false
    document.querySelector('.railtoggle')?.focus()
  }
}

watch(pathname, path => {
  const title = path === '/' ? 'Overview'
    : path === '/signal' ? 'Journal'
      : path === '/docs' ? 'Reference'
        : path.split('/').pop().replace(/^./, letter => letter.toUpperCase())
  document.title = `${title} / ${brand.name}`
}, { immediate: true })

onMounted(() => {
  addEventListener('popstate', popState)
  addEventListener('keydown', onKeydown)
  nextTick(scrollToHash)
})

onBeforeUnmount(() => {
  removeEventListener('popstate', popState)
  removeEventListener('keydown', onKeydown)
})
</script>

<template>
  <a href="#main" class="skip-link">Skip to content</a>

  <div class="shell" :class="{ 'is-rail-open': railOpen }">
    <button class="rail-scrim" type="button" tabindex="-1" aria-hidden="true" @click="railOpen = false"></button>

    <aside id="ivorane-rail" class="rail" aria-label="Ivorane navigation">
      <div class="rail-head">
        <a class="rail-mark" href="/" :aria-label="`${brand.name} overview`" @click.prevent="navigate('/')">
          <img src="/icon.svg" alt="" width="34" height="34" decoding="async" aria-hidden="true" />
          <span>{{ brand.name }}</span>
        </a>
        <p class="rail-line">{{ brand.network }} market terminal<br />Preview build</p>
      </div>

      <nav class="rail-nav" aria-label="Primary">
        <a v-for="[num, label, to, note] in nav" :key="to" :href="to"
           :aria-current="isCurrent(to) ? 'page' : undefined" @click.prevent="navigate(to)">
          <b>{{ num }}</b><span>{{ label }}</span><small>{{ note }}</small>
        </a>
      </nav>

      <div class="rail-foot">
        <div class="rail-actions">
          <SolanaWalletButton class="rail-wallet" />
          <a class="btn btn-quiet btn-full" href="/app/markets" @click.prevent="navigate('/app/markets')">Open the board</a>
        </div>
        <dl class="rail-readout">
          <div><dt>Feed</dt><dd class="is-hold">Static samples</dd></div>
          <div><dt>Unit</dt><dd>{{ brand.asset }} / unissued</dd></div>
          <div><dt>Stores</dt><dd>This browser</dd></div>
          <div><dt>Target</dt><dd>{{ brand.domain }}</dd></div>
        </dl>
        <p class="rail-tiny">Illustrative values. No live quote, custody, signing or execution.</p>
      </div>
    </aside>

    <div class="shell-main">
      <div class="strip" role="status" aria-label="Build status">
        <button class="railtoggle" type="button" :aria-expanded="railOpen" aria-controls="ivorane-rail"
                :aria-label="railOpen ? 'Close navigation' : 'Open navigation'" @click="railOpen = !railOpen">
          <X v-if="railOpen" :size="18" /><Menu v-else :size="18" />
        </button>
        <span class="strip-section"><i aria-hidden="true"></i>Mode <b>Preview</b></span>
        <span class="strip-section strip-hide-sm">Network <b>{{ brand.network }}</b></span>
        <span class="strip-section strip-hide-sm">Unit <b>{{ brand.asset }} / not issued</b></span>
        <span class="strip-section">Records <b>local only</b></span>
        <span class="strip-tail">{{ brand.handle }} / {{ brand.domain }} / targets, not registered</span>
      </div>

      <main id="main" class="sheet">
        <HomePage v-if="pathname === '/'" @navigate="navigate" />
        <MarketWorkspace v-else-if="appPage" :path="pathname" @navigate="navigate" />
        <ScenarioLab v-else-if="pathname === '/signal'" />
        <DocsPage v-else-if="['/docs', '/risk', '/terms', '/privacy'].includes(pathname)" :path="pathname" @navigate="navigate" />
        <section v-else class="sheet-body notfound">
          <p class="kick kick-ink">Error 404</p>
          <h1 class="h1" tabindex="-1">No such leaf.</h1>
          <p class="lede">That path is not in this preview. Start from the overview or open the board.</p>
          <p><a class="btn btn-primary" href="/" @click.prevent="navigate('/')">Back to overview</a></p>
        </section>

        <footer class="colophon">
          <div class="colophon-brand">
            <img src="/icon.svg" alt="" width="24" height="24" decoding="async" aria-hidden="true" />
            <b>{{ brand.name }}</b>
          </div>
          <p>{{ brand.network }} preview / illustrative values / no transactions / {{ brand.handle }} and {{ brand.domain }} are unregistered targets</p>
          <nav aria-label="Site links">
            <a v-for="[label, to] in colophonLinks" :key="to" :href="to" @click.prevent="navigate(to)">{{ label }}</a>
          </nav>
        </footer>
      </main>
    </div>
  </div>
</template>

<style scoped>
.notfound { display: grid; gap: 18px; align-content: start; padding-top: 46px; max-width: 60ch; }
.notfound .btn { justify-self: start; }
.rail-wallet :deep(.wallet-button) { width: 100%; }
</style>
