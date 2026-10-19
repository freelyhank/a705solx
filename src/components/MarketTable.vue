<script setup>
import { computed } from 'vue'
import { ArrowRight } from 'lucide-vue-next'
import { calculatePayoff } from '../payoff.js'

const props = defineProps({
  compact: { type: Boolean, default: false },
  query: { type: String, default: '' },
  side: { type: String, default: 'All' },
})

const emit = defineEmits(['navigate', 'reset'])

const money = value => new Intl.NumberFormat('en-US', {
  style: 'currency',
  currency: 'USD',
  minimumFractionDigits: 2,
  maximumFractionDigits: 2,
}).format(value)

const assets = Object.freeze([
  { symbol: 'SOL', name: 'Solana', spot: 182, initials: 'S' },
  { symbol: 'JUP', name: 'Jupiter', spot: 1.12, initials: 'J' },
  { symbol: 'BONK', name: 'Bonk', spot: 0.000021, initials: 'B' },
  { symbol: 'USDC', name: 'USD Coin', spot: 1, initials: '$' },
])

const allRows = assets.map(asset => ({
  ...asset,
  payoff: calculatePayoff({ spot: asset.spot, capPercent: 5, settlementPrice: asset.spot }),
}))

const rows = computed(() => {
  const needle = props.query.trim().toLocaleLowerCase()
  const source = allRows
  if (!needle) return source
  return source.filter(asset => `${asset.symbol} ${asset.name}`.toLocaleLowerCase().includes(needle))
})

const activeSide = computed(() => {
  const value = props.side.trim().toLocaleLowerCase()
  return value === 'income' || value === 'upside' ? value : ''
})

const stats = Object.freeze([
  { label: 'Premium received', detail: 'Model epoch' },
  { label: 'Open series', detail: 'Current epoch' },
  { label: 'Active income', detail: 'Indicative' },
  { label: 'Total volume', detail: 'Model ledger' },
])

function navigate(symbol) {
  emit('navigate', `/app/markets/${symbol}`)
}

function resetSearch() {
  emit('reset')
}
</script>

<template>
  <section class="market-table-section" :class="{ 'compact-market-table': compact }" :aria-label="compact ? 'Market quote table' : undefined" :aria-labelledby="!compact ? 'market-table-title' : undefined">
    <div v-if="!compact" class="section-head market-table-heading">
      <h2 id="market-table-title" class="h2">Market quotes</h2>
      <p class="section-meta market-feed-note"><span class="feed-dot" aria-hidden="true"></span>Example quotes / no live feed</p>
    </div>

    <div v-if="!compact" class="market-stats">
      <p class="stats-heading"><span>On chain stats</span><span class="stats-note">Interface preview / no live feed</span></p>
      <dl class="cells" aria-label="On chain stats preview">
        <div v-for="stat in stats" :key="stat.label">
          <dt class="cell-label">{{ stat.label }}<span class="chip chip-flare preview-badge">Preview</span></dt>
          <dd class="cell-value">—</dd>
          <dd class="cell-note">{{ stat.detail }}</dd>
        </div>
      </dl>
    </div>

    <div class="market-table-toolbar">
      <p class="table-intro">Cap +5% / 30-day epoch / model values per unit</p>
      <p v-if="props.query.trim()" class="search-state" aria-live="polite">{{ rows.length }} result{{ rows.length === 1 ? '' : 's' }} for “{{ props.query.trim() }}”</p>
    </div>

    <div v-if="rows.length" class="table-scroll">
      <table class="ledger market-table">
        <caption class="sr-only">Model market values. Quotes are examples and are not live prices.</caption>
        <thead>
          <tr>
            <th scope="col">Asset</th>
            <th scope="col" class="num">Cap</th>
            <th scope="col" class="num" :class="{ 'is-emphasized': activeSide === 'income' }">Income</th>
            <th scope="col" class="num" :class="{ 'is-emphasized': activeSide === 'upside' }">Upside</th>
            <th scope="col" class="num">Epoch</th>
            <th scope="col">Status</th>
          </tr>
        </thead>
        <tbody>
          <tr v-for="asset in rows" :key="asset.symbol">
            <td>
              <a class="token asset-link" :href="`/app/markets/${asset.symbol}`" :aria-label="`Open ${asset.symbol} market, example quote ${money(asset.spot)}`" @click.prevent="navigate(asset.symbol)">
                <span class="token-mark" aria-hidden="true">{{ asset.initials }}</span>
                <span class="token-name"><b>{{ asset.symbol }}</b><small>{{ asset.name }} / Example quote</small></span>
              </a>
            </td>
            <td class="num"><span class="cap-value">+5%</span></td>
            <td class="num" :class="{ 'is-emphasized': activeSide === 'income' }"><span class="model-value income-value">{{ money(asset.payoff.incomePrice) }}</span><small class="quote-label">Model</small></td>
            <td class="num" :class="{ 'is-emphasized': activeSide === 'upside' }"><span class="model-value upside-value">{{ money(asset.payoff.upsidePrice) }}</span><small class="quote-label">Model</small></td>
            <td class="num"><span class="epoch-value">30 days</span></td>
            <td><span class="chip"><i aria-hidden="true"></i>Model</span></td>
          </tr>
        </tbody>
      </table>
    </div>
    <div v-if="rows.length" class="market-cards" aria-label="Market cards">
      <a v-for="asset in rows" :key="`card-${asset.symbol}`" class="market-mobile-card" :href="`/app/markets/${asset.symbol}`" @click.prevent="navigate(asset.symbol)">
        <span class="token-mark" aria-hidden="true">{{ asset.initials }}</span>
        <span class="mobile-asset-copy"><b>{{ asset.symbol }}</b><small>{{ asset.name }} / Example quote</small></span>
        <span class="mobile-asset-values"><span><small>Income</small><b>{{ money(asset.payoff.incomePrice) }}</b></span><span><small>Upside</small><b>{{ money(asset.payoff.upsidePrice) }}</b></span><ArrowRight :size="14" /></span>
      </a>
    </div>
    <div v-else class="empty-market-state notice notice-flag" role="status" aria-live="polite">
      <p>No model markets match “{{ props.query.trim() }}”.</p>
      <button class="btn btn-quiet" type="button" @click="resetSearch">Reset search</button>
    </div>

    <div class="market-table-footnote"><span>Indicative model values use a 30-day epoch and 25% volatility.</span><span>Example quotes are not live prices.</span></div>
  </section>
</template>

<style scoped>
/* --------------------------------------------------------------- heading */
.section-head {
  display: flex; align-items: baseline; justify-content: space-between; gap: 16px;
  flex-wrap: wrap;
  padding-bottom: 12px;
  border-bottom: var(--rule-w) solid var(--ink);
}
.section-meta { margin: 0; color: var(--muted); font: 11px/1.4 var(--mono); }
.market-table-heading { margin-bottom: 20px; }
.market-feed-note { display: inline-flex; align-items: center; gap: 8px; }
.feed-dot { width: 7px; height: 7px; background: var(--ledger); }

/* ----------------------------------------------------------- stats band */
.market-stats { display: grid; gap: 10px; }
.stats-heading {
  display: flex; align-items: baseline; justify-content: space-between; gap: 18px;
  margin: 0;
  color: var(--ink);
  font: 700 11px/1 var(--mono); letter-spacing: .12em; text-transform: uppercase;
}
.stats-note { color: var(--muted); font-weight: 400; letter-spacing: .06em; }
.market-stats dl { margin: 0; }
.market-stats dd { margin: 0; }
.preview-badge { margin-left: 8px; vertical-align: 1px; }

/* -------------------------------------------------------------- toolbar */
.market-table-toolbar {
  display: flex; align-items: baseline; justify-content: space-between; gap: 20px;
  margin: 22px 0 10px;
}
.table-intro, .search-state { margin: 0; color: var(--muted); font: 10px/1.5 var(--mono); }
.search-state { color: var(--flare); text-align: right; }

/* ---------------------------------------------------------------- table */
.table-scroll { overflow-x: auto; }
.market-table { min-width: 720px; }
.market-table th[scope='row'] { padding: 0; }
.asset-link { width: 100%; }
.asset-link small { color: var(--muted); font: 11px/1.2 var(--sans); }
.cap-value { font: 600 13px/1.4 var(--mono); color: var(--signal); }
.model-value { display: block; font: 600 13px/1.4 var(--mono); font-variant-numeric: tabular-nums; }
.income-value { color: var(--ledger); }
.upside-value { color: var(--flare); }
.quote-label {
  display: block;
  margin-top: 3px;
  color: var(--faint);
  font: 9px/1.2 var(--mono); letter-spacing: .1em; text-transform: uppercase;
}
.epoch-value { color: var(--muted); font: 11px/1.4 var(--mono); font-variant-numeric: tabular-nums; }
.market-table thead th.is-emphasized { color: var(--flare); border-bottom-color: var(--flare); }
.market-table tbody td.is-emphasized { background: var(--sunken); box-shadow: inset 3px 0 0 var(--flare); }
.market-table tbody td.is-emphasized .model-value { font-weight: 700; }

/* ----------------------------------------------------------- card switch */
.market-cards { display: none; }
.market-mobile-card {
  display: flex; align-items: center; gap: 11px;
  padding: 13px 0;
  border-bottom: var(--hair-w) solid var(--hair);
  color: inherit;
  text-decoration: none;
}
.market-mobile-card:last-child { border-bottom: 0; }
.mobile-asset-copy { display: grid; gap: 2px; min-width: 0; }
.mobile-asset-copy b { font: 700 13px/1.2 var(--mono); }
.mobile-asset-copy small { overflow: hidden; color: var(--muted); font: 11px/1.2 var(--sans); text-overflow: ellipsis; white-space: nowrap; }
.mobile-asset-values { display: flex; align-items: center; gap: 12px; margin-left: auto; }
.mobile-asset-values > span { display: grid; gap: 2px; text-align: right; }
.mobile-asset-values small { color: var(--faint); font: 10px/1.2 var(--mono); letter-spacing: .08em; text-transform: uppercase; }
.mobile-asset-values b { font: 600 12px/1.2 var(--mono); font-variant-numeric: tabular-nums; }
.mobile-asset-values > span:first-child b { color: var(--ledger); }
.mobile-asset-values > span:nth-child(2) b { color: var(--flare); }
.mobile-asset-values svg { color: var(--muted); }

/* ---------------------------------------------------------- empty state */
.empty-market-state { justify-content: space-between; gap: 15px; margin-top: 8px; }
.empty-market-state p { margin: 0; }
.empty-market-state .btn { min-height: 34px; }

/* ------------------------------------------------------------- footnote */
.market-table-footnote {
  display: flex; justify-content: space-between; gap: 20px;
  margin-top: 12px;
  color: var(--faint);
  font: 10px/1.6 var(--mono);
}

/* ---------------------------------------------------------------- compact */
.compact-market-table .market-table { min-width: 0; table-layout: fixed; }
.compact-market-table .market-table th:nth-child(5), .compact-market-table .market-table td:nth-child(5),
.compact-market-table .market-table th:nth-child(6), .compact-market-table .market-table td:nth-child(6) { display: none; }
.compact-market-table .market-table th, .compact-market-table .market-table td { padding-inline: 8px; }
.compact-market-table .market-table th:first-child, .compact-market-table .market-table td:first-child { width: 42%; }

/* ------------------------------------------------------------ responsive */
@media (max-width: 780px) {
  .market-table-heading { align-items: flex-start; flex-direction: column; gap: 8px; }
  .stats-heading { align-items: flex-start; flex-direction: column; gap: 5px; }
  .market-table-toolbar { align-items: flex-start; flex-direction: column; gap: 6px; }
  .search-state { text-align: left; }
  .market-table-footnote { flex-direction: column; gap: 4px; }
}
@media (max-width: 580px) {
  .table-scroll { display: none; }
  .market-cards { display: block; }
}
@media (max-width: 360px) {
  .market-mobile-card { flex-wrap: wrap; }
  .mobile-asset-values { width: 100%; margin-left: 0; justify-content: flex-start; gap: 20px; }
}
</style>
