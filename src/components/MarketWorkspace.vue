<script setup>
import { ArrowLeft, ArrowUpRight, Search, SlidersHorizontal, TrendingUp, WalletCards } from 'lucide-vue-next'
import { computed, ref } from 'vue'
import MarketTable from './MarketTable.vue'
import PayoffExplorer from './PayoffExplorer.vue'
import { brand } from '../brand.js'

const props = defineProps({ path: { type: String, default: '/app/markets' } })
const emit = defineEmits(['navigate'])
const query = ref('')
const side = ref('All')
const showFilters = ref(false)
const symbols = ['SOL', 'JUP', 'BONK', 'USDC']
const symbol = computed(() => {
  const match = props.path.match(/\/app\/markets\/([^/]+)/i)
  return match && symbols.includes(match[1].toUpperCase()) ? match[1].toUpperCase() : ''
})
const pageTitle = computed(() => props.path.includes('launchpad') ? 'Launchpad' : props.path.includes('auctions') ? 'Auctions' : props.path.includes('recompose') ? 'Recompose' : symbol.value ? `${symbol.value} market` : 'Markets')
const copy = computed(() => ({ Markets: 'Scan defined-cap series and compare the two sides of each market.', Launchpad: 'Record a candidate market for a future verified listing.', Auctions: 'Review the sample Upside auction surface without placing an order.', Recompose: 'Pair Income and Upside positions back into one market unit.' }[pageTitle.value] || 'Read the illustrative settlement path for this asset.'))
</script>

<template>
  <section class="workspace sheet-body">
    <header class="workspace-head">
      <div class="workspace-title">
        <p class="kick">{{ brand.network }} / paper market</p>
        <h1 class="h1" tabindex="-1">{{ pageTitle }}</h1>
        <p class="lede">{{ copy }}</p>
      </div>
      <a class="btn btn-quiet workspace-back" href="/" @click.prevent="emit('navigate','/')"><ArrowLeft :size="15" /> Markets</a>
    </header>

    <dl class="cells workspace-facts">
      <div><dt class="cell-label">Sheet</dt><dd class="cell-value">{{ pageTitle }}</dd><dd class="cell-note">{{ brand.network }} preview</dd></div>
      <div><dt class="cell-label">Cap default</dt><dd class="cell-value">+5%</dd><dd class="cell-note">30-day epoch</dd></div>
      <div><dt class="cell-label">Feed</dt><dd class="cell-value">Static</dd><dd class="cell-note">Not a live quote</dd></div>
      <div><dt class="cell-label">Unit</dt><dd class="cell-value">{{ brand.asset }}*</dd><dd class="cell-note">Proposed, unissued</dd></div>
    </dl>

    <div v-if="symbol" class="asset-detail">
      <div class="section-head">
        <h2 class="h2">{{ symbol }} / {{ brand.network }}</h2>
        <p class="section-meta">30-day epoch | +5% cap | illustrative values</p>
      </div>
      <div class="detail-grid">
        <aside class="detail-figure">
          <div class="detail-mark">
            <span class="token-mark" aria-hidden="true">{{ symbol.slice(0, 1) }}</span>
            <span class="token-name"><b>{{ symbol }}</b><small>Example series</small></span>
            <span class="chip chip-flare"><i aria-hidden="true"></i>Paper only</span>
          </div>
          <dl class="detail-metrics">
            <div><dt>Reference</dt><dd>{{ symbol === 'SOL' ? '$182.00' : symbol === 'JUP' ? '$1.12' : symbol === 'BONK' ? '$0.000021' : '$1.00' }}</dd></div>
            <div><dt>Income model</dt><dd class="is-income">+5% cap</dd></div>
            <div><dt>Settlement</dt><dd>Not live</dd></div>
          </dl>
          <p class="foot-note">Illustrative values. No live quote, custody, signing or execution.</p>
        </aside>
        <div class="detail-payoff">
          <PayoffExplorer :spot="symbol === 'SOL' ? 182 : symbol === 'JUP' ? 1.12 : symbol === 'BONK' ? 0.000021 : 1" :asset="symbol" />
        </div>
      </div>
    </div>

    <div v-else-if="pageTitle === 'Launchpad'" class="workspace-card panel">
      <div class="panel-head">
        <h2 class="h2">Suggest a market.</h2>
        <span class="chip"><i aria-hidden="true"></i>Listing intake</span>
      </div>
      <div class="panel-body workspace-card-body">
        <span class="card-icon"><WalletCards :size="19" /></span>
        <p>Any SPL market unit with a verified price feed and USDC liquidity path may qualify. This preview records no wallet request or listing transaction.</p>
        <a class="btn btn-primary" href="/docs#listing" @click.prevent="emit('navigate','/docs#listing')">Read listing notes <ArrowUpRight :size="15" /></a>
      </div>
    </div>

    <div v-else-if="pageTitle === 'Recompose'" class="workspace-card panel">
      <div class="panel-head">
        <h2 class="h2">One asset again.</h2>
        <span class="chip"><i aria-hidden="true"></i>Income + Upside</span>
      </div>
      <div class="panel-body workspace-card-body">
        <span class="card-icon"><TrendingUp :size="19" /></span>
        <p>Recomposition would require a wallet transaction. The local preview keeps this action visible while execution remains unavailable.</p>
        <button class="btn" type="button" disabled>Recompose unavailable</button>
      </div>
    </div>

    <template v-else>
      <div class="board">
        <!-- MarketTable renders its own ruled section head ("Market quotes") plus the
             stats band, so the workspace does not repeat a heading here. -->
        <div class="filters">
          <label class="search">
            <Search :size="15" />
            <span class="sr-only">Search markets</span>
            <input v-model="query" class="input" type="search" placeholder="Search token or market" />
          </label>
          <div class="seg side-tabs" aria-label="Position type">
            <button v-for="option in ['All','Income','Upside']" :key="option" type="button" :aria-pressed="side === option" :class="{ 'is-on': side === option }" @click="side = option">{{ option }}</button>
          </div>
          <button class="btn btn-quiet filter-button" type="button" :aria-expanded="showFilters" @click="showFilters = !showFilters"><SlidersHorizontal :size="15" /> Details</button>
        </div>
        <p v-if="showFilters" class="notice filter-details">All examples use a +5% cap, 30-day epoch and 25% model volatility.</p>

        <MarketTable :query="query" :side="side" @reset="query = ''" @navigate="to => emit('navigate',to)" />
      </div>
    </template>
  </section>
</template>

<style scoped>
/* ------------------------------------------------------- ruled header band */
.workspace {
  display: grid;
  gap: 20px;
  padding-top: 34px;
}
.workspace-head {
  display: flex; align-items: flex-end; justify-content: space-between; gap: 24px;
  flex-wrap: wrap;
  padding-bottom: 20px;
  border-bottom: var(--rule-w) solid var(--ink);
}
.workspace-title { display: grid; gap: 10px; min-width: 0; }
.workspace-head .h1 { font-size: clamp(28px, 3.6vw, 46px); }
.workspace-back { flex: none; }

.workspace-facts { margin: 0; }
.workspace-facts dd { margin: 0; }

/* ------------------------------------------------------------ section head */
.section-head {
  display: flex; align-items: baseline; justify-content: space-between; gap: 16px;
  flex-wrap: wrap;
  padding-bottom: 12px;
  border-bottom: var(--rule-w) solid var(--ink);
}
.section-meta { margin: 0; color: var(--muted); font: 11px/1.4 var(--mono); }
.foot-note { margin: 0; color: var(--faint); font: 10px/1.6 var(--mono); }

/* --------------------------------------------------------- asset detail */
.asset-detail { display: grid; gap: 0; }
.detail-grid {
  display: grid;
  grid-template-columns: minmax(0, 300px) minmax(0, 1fr);
  border-bottom: var(--hair-w) solid var(--hair);
}
.detail-figure {
  display: grid; gap: 16px; align-content: start;
  padding: 20px 20px 20px 0;
  border-right: var(--hair-w) solid var(--hair);
}
.detail-mark { display: flex; align-items: center; gap: 10px; }
.detail-mark .chip { margin-left: auto; }
.detail-metrics { margin: 0; border-top: var(--rule-w) solid var(--ink); }
.detail-metrics > div {
  display: flex; align-items: baseline; justify-content: space-between; gap: 10px;
  padding: 10px 0;
  border-bottom: var(--hair-w) solid var(--hair);
}
.detail-metrics dt { color: var(--muted); font: 10px/1.4 var(--mono); letter-spacing: .12em; text-transform: uppercase; }
.detail-metrics dd { margin: 0; font: 600 14px/1.4 var(--mono); font-variant-numeric: tabular-nums; }
.detail-metrics dd.is-income { color: var(--ledger); }
.detail-payoff { min-width: 0; padding: 20px 0 20px 20px; }

/* ------------------------------------------------------------- filter bar */
.board { display: grid; gap: 16px; }
.filters { display: flex; align-items: center; gap: 10px; flex-wrap: wrap; }
.search {
  display: flex; align-items: center; gap: 8px;
  flex: 1 1 240px;
  max-width: 340px;
  color: var(--muted);
}
.search .input { min-height: 42px; }
.side-tabs { flex: none; }
.filters .seg > * { min-height: 42px; }
.filter-button { margin-left: auto; }
.filter-details { margin: 0; }

/* ------------------------------------------------------------- card branch */
.workspace-card { max-width: 720px; }
.workspace-card-body { display: grid; gap: 14px; justify-items: start; }
.workspace-card-body p { margin: 0; max-width: 60ch; color: var(--fg-2); font-size: 14px; line-height: 1.6; }
.card-icon {
  display: grid; place-items: center;
  width: 38px; height: 38px;
  border: var(--hair-w) solid var(--ink);
  background: var(--sunken);
  color: var(--ink);
}

/* ------------------------------------------------------------ responsive */
@media (max-width: 980px) {
  .detail-grid { grid-template-columns: minmax(0, 1fr); }
  .detail-figure { padding: 20px 0; border-right: 0; border-bottom: var(--hair-w) solid var(--hair); }
  .detail-payoff { padding: 20px 0 0; }
}
@media (max-width: 720px) {
  .workspace { padding-top: 24px; }
  .filters { align-items: stretch; }
  .search { max-width: none; }
  .filter-button { margin-left: 0; }
}
</style>
