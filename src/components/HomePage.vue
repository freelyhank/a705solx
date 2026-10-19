<script setup>
import { computed, ref } from 'vue'
import { ArrowRight } from 'lucide-vue-next'
import { calculatePayoff } from '../payoff.js'
import { brand } from '../brand.js'

const emit = defineEmits(['navigate'])

const assets = [
  { symbol: 'SOL', name: 'Solana', spot: 182.4, change: 4.82, initials: 'S' },
  { symbol: 'JUP', name: 'Jupiter', spot: 1.12, change: 2.1, initials: 'J' },
  { symbol: 'BONK', name: 'Bonk', spot: 0.000021, change: -1.34, initials: 'B' },
  { symbol: 'USDC', name: 'USD Coin', spot: 1, change: 0.01, initials: '$' },
]

const selected = ref('SOL')
const side = ref('Income')
const units = ref(1)
const cap = ref(5)
const query = ref('')

const asset = computed(() => assets.find(item => item.symbol === selected.value) || assets[0])
const model = computed(() => calculatePayoff({
  spot: asset.value.spot,
  capPercent: cap.value,
  settlementPrice: asset.value.spot,
}))
const unitPrice = computed(() => (side.value === 'Income' ? model.value.incomePrice : model.value.upsidePrice))
const amount = computed(() => {
  const value = Number(units.value)
  return Number.isFinite(value) && value > 0 ? value : 0
})

const money = value => new Intl.NumberFormat('en-US', {
  style: 'currency', currency: 'USD', maximumFractionDigits: 2,
}).format(value)
const spot = value => (value < 0.01 ? `$${value.toFixed(6)}` : money(value))
const pct = value => `${value > 0 ? '+' : ''}${value.toFixed(2)}%`
const signed = value => `${value > 0 ? '+' : ''}${money(value)}`

const tape = computed(() => assets.map(item => ({
  ...item,
  model: calculatePayoff({ spot: item.spot, capPercent: 5, settlementPrice: item.spot }),
})))

function toBoundary() {
  document.getElementById('cap')?.scrollIntoView({ behavior: 'smooth', block: 'start' })
}

const filtered = computed(() => {
  const needle = query.value.trim().toLowerCase()
  if (!needle) return tape.value
  return tape.value.filter(item => `${item.symbol} ${item.name}`.toLowerCase().includes(needle))
})

const briefing = [
  ['01', 'Board', 'Every listed pair on one ruled ledger with a stated cap and the two modelled unit prices.', '/app/markets'],
  ['02', 'Boundary', 'Set the cap yourself and read how value splits at settlement, before anything else.', '#cap'],
  ['03', 'Journal', 'Write the thesis and the risk into a record that stays in this browser.', '/signal'],
  ['04', 'Reference', 'How the payoff maths works, what the preview does and where it stops.', '/docs'],
]
</script>

<template>
  <div class="sheet-body">
    <!-- masthead -->
    <header class="mast">
      <div class="mast-head">
        <p class="kick">Sheet {{ brand.asset }} / {{ brand.network }} / preview build</p>
        <h1 class="h1" tabindex="-1">Read the chain.<br />Price the boundary.<br /><em>Keep the record.</em></h1>
      </div>
      <div class="mast-side">
        <p class="lede">{{ brand.supportingLine }}</p>
        <div class="mast-actions">
          <a class="btn btn-primary" href="/app/markets" @click.prevent="emit('navigate', '/app/markets')">
            Open the board <ArrowRight :size="15" />
          </a>
          <button class="btn btn-quiet" type="button" @click="toBoundary">Price a boundary</button>
        </div>
        <p class="mast-note">
          Sample quotes, a user-priced model and a browser-local journal. Nothing here is live and
          nothing is signed.
        </p>
      </div>
    </header>

    <!-- ruled load rail -->
    <dl class="loadrail-inner">
      <div class="loadrail-cell"><dt>Pairs listed</dt><dd>04</dd></div>
      <div class="loadrail-cell"><dt>Cap default</dt><dd>+5%</dd></div>
      <div class="loadrail-cell"><dt>Epoch</dt><dd>30d</dd></div>
      <div class="loadrail-cell"><dt>Feed</dt><dd>Static</dd></div>
      <div class="loadrail-cell"><dt>Records</dt><dd>Local</dd></div>
      <div class="loadrail-cell"><dt>Unit</dt><dd>{{ brand.asset }}*</dd></div>
    </dl>

    <div class="editorial">
      <!-- board -->
      <section class="col-a1" aria-labelledby="board-title">
        <div class="section-head">
          <h2 id="board-title" class="h2">The board</h2>
          <div class="section-tools">
            <div class="seg" role="group" aria-label="Position type">
              <button v-for="option in ['All', 'Income', 'Upside']" :key="option" type="button"
                      :class="{ 'is-on': side === option }" :aria-pressed="side === option"
                      @click="side = option">{{ option }}</button>
            </div>
            <label class="search">
              <span class="sr-only">Filter the board</span>
              <input v-model="query" type="search" placeholder="Filter by symbol" />
            </label>
          </div>
        </div>

        <table class="ledger board">
          <caption class="sr-only">Illustrative {{ brand.network }} pairs with modelled unit prices</caption>
          <thead>
            <tr>
              <th scope="col">Pair</th>
              <th scope="col">Sample spot</th>
              <th scope="col">Move</th>
              <th scope="col" :class="{ 'is-loud': side === 'Income' }">Income / unit</th>
              <th scope="col" :class="{ 'is-loud': side === 'Upside' }">Upside / unit</th>
              <th scope="col">Open</th>
            </tr>
          </thead>
          <tbody>
            <tr v-for="item in filtered" :key="item.symbol" :class="{ 'is-mark': item.symbol === selected }">
              <th scope="row">
                <button class="token" type="button" :aria-pressed="item.symbol === selected" @click="selected = item.symbol">
                  <span class="token-mark" :class="{ 'is-flare': item.symbol === selected }" aria-hidden="true">{{ item.initials }}</span>
                  <span class="token-name"><b>{{ item.symbol }}</b><small>{{ item.name }}</small></span>
                </button>
              </th>
              <td class="num">{{ spot(item.spot) }}</td>
              <td class="num" :class="item.change >= 0 ? 'is-up' : 'is-down'">{{ pct(item.change) }}</td>
              <td class="num">{{ money(item.model.incomePrice) }}</td>
              <td class="num">{{ money(item.model.upsidePrice) }}</td>
              <td><a class="rowlink" :href="`/app/markets/${item.symbol}`"
                     :aria-label="`Open the ${item.symbol} sheet`"
                     @click.prevent="emit('navigate', `/app/markets/${item.symbol}`)">Sheet</a></td>
            </tr>
            <tr v-if="!filtered.length">
              <td class="empty" colspan="6">
                Nothing matches “{{ query }}”.
                <button class="rowlink" type="button" @click="query = ''">Clear filter</button>
              </td>
            </tr>
          </tbody>
        </table>
        <p class="foot-note">* {{ brand.asset }} is a proposed symbol. No token has been issued and no contract address exists.</p>
      </section>

      <!-- quote calculator -->
      <aside class="col-b1 panel" aria-labelledby="quote-title">
        <div class="panel-head">
          <h2 id="quote-title">Unit quote</h2>
          <span class="chip chip-flare"><i aria-hidden="true"></i>Sandbox</span>
        </div>
        <div class="panel-body">
          <div class="formrow">
            <label for="quote-asset">Pair</label>
            <select id="quote-asset" v-model="selected" class="input">
              <option v-for="item in assets" :key="item.symbol" :value="item.symbol">{{ item.symbol }} / {{ item.name }}</option>
            </select>
          </div>

          <div class="formrow">
            <span class="flabel">Leg</span>
            <div class="seg" role="group" aria-label="Position leg">
              <button v-for="option in ['Income', 'Upside']" :key="option" type="button"
                      :class="{ 'is-on': side === option }" :aria-pressed="side === option"
                      @click="side = option">{{ option }}</button>
            </div>
          </div>

          <div class="formrow">
            <label for="quote-units">Units</label>
            <span class="field-group">
              <input id="quote-units" v-model.number="units" type="number" inputmode="decimal"
                     min="0.01" max="1000000" step="0.01" />
              <b>{{ selected }}</b>
            </span>
          </div>

          <div class="formrow">
            <label for="quote-cap">Cap K <output>+{{ cap }}%</output></label>
            <input id="quote-cap" v-model.number="cap" class="range" type="range" min="0" max="10" step="1" />
            <span class="range-scale"><span>0%</span><span>10%</span></span>
          </div>

          <dl class="readout">
            <div><dt>Sample spot</dt><dd>{{ spot(asset.spot) }}</dd></div>
            <div><dt>Cap price K</dt><dd>{{ money(model.strike) }}</dd></div>
            <div><dt>Model / unit</dt><dd>{{ money(unitPrice) }}</dd></div>
            <div><dt>Leg</dt><dd>{{ side }}</dd></div>
          </dl>
          <p class="total"><span>Modelled value</span><strong>{{ money(unitPrice * amount) }}</strong></p>
          <button class="btn btn-primary btn-full" type="button" :disabled="!amount" @click="toBoundary">
            See the split <ArrowRight :size="15" />
          </button>
          <p class="foot-note">Example arithmetic only. No live quote is read and no order is placed.</p>
        </div>
      </aside>

      <!-- payoff figure -->
      <section id="cap" class="col-a2" aria-labelledby="split-title">
        <div class="section-head">
          <h2 id="split-title" class="h2">Settlement split</h2>
          <p class="section-meta">{{ selected }} / cap +{{ cap }}% / settlement at the sample spot</p>
        </div>
        <div class="split-grid">
          <div class="split-figure">
            <figure class="payoff">
              <svg viewBox="0 0 640 300" role="img" :aria-label="`${selected} settlement partition at a ${cap} percent cap`">
                <title>{{ selected }} settlement partition</title>
                <desc>Value below the cap is an income leg priced {{ money(model.incomePrice) }} per unit; value above the cap is an upside leg priced {{ money(model.upsidePrice) }} per unit.</desc>
                <g class="figure-grid">
                  <line v-for="t in 5" :key="`g${t}`" :x1="60" :x2="620" :y1="40 + t * 44" :y2="40 + t * 44" />
                </g>
                <rect x="60" :y="40" :width="496 * model.incomeShare" height="220" class="band-income" />
                <rect :x="60 + 496 * model.incomeShare" :y="40" :width="496 * model.upsideShare" height="220" class="band-upside" />
                <line x1="60" y1="40" x2="60" y2="260" class="figure-axis" />
                <line x1="60" y1="260" x2="620" y2="260" class="figure-axis" />
                <line :x1="60 + 496 * model.incomeShare" y1="40" :x2="60 + 496 * model.incomeShare" y2="260" class="figure-cap" />
                <text :x="60 + 496 * model.incomeShare + 8" y="58" class="figure-label">CAP K {{ money(model.strike) }}</text>
                <text x="72" y="98" class="figure-band">INCOME</text>
                <text :x="620 - 8" y="98" class="figure-band is-end" text-anchor="end">UPSIDE</text>
                <text x="72" y="134" class="figure-value">{{ money(model.incomePrice) }} / unit</text>
                <text :x="620 - 8" y="134" class="figure-value is-end" text-anchor="end">{{ money(model.upsidePrice) }} / unit</text>
                <line x1="60" y1="292" x2="620" y2="292" class="figure-axis" />
                <text x="60" y="288" class="figure-tick">0</text>
                <text :x="60 + 496 * model.incomeShare" y="288" class="figure-tick" text-anchor="middle">K</text>
                <text x="620" y="288" class="figure-tick" text-anchor="end">settlement</text>
              </svg>
              <figcaption>{{ model.incomeShare >= model.upsideShare ? 'Most of the sample value sits below the cap.' : 'Most of the sample value sits above the cap.' }} Example arithmetic.</figcaption>
            </figure>
          </div>
          <dl class="split-readout">
            <div><dt>Income share</dt><dd>{{ (model.incomeShare * 100).toFixed(1) }}%</dd></div>
            <div><dt>Upside share</dt><dd>{{ (model.upsideShare * 100).toFixed(1) }}%</dd></div>
            <div><dt>Income at settlement</dt><dd>{{ money(model.incomeSettlement) }}</dd></div>
            <div><dt>Upside at settlement</dt><dd>{{ money(model.upsideSettlement) }}</dd></div>
            <div><dt>Model premium</dt><dd>{{ money(model.premium) }}</dd></div>
            <div><dt>Breakeven</dt><dd>{{ money(model.breakeven) }}</dd></div>
            <div><dt>Sample move</dt><dd :class="model.spotChangePercent >= 0 ? 'is-up' : 'is-down'">{{ signed(model.spotChangePercent) }}</dd></div>
            <div><dt>Cap</dt><dd>+{{ cap }}%</dd></div>
          </dl>
        </div>
      </section>

      <!-- tape -->
      <section class="col-b2" aria-labelledby="tape-title">
        <div class="section-head">
          <h2 id="tape-title" class="h2">Sample tape</h2>
          <p class="section-meta">Not a feed</p>
        </div>
        <ul class="tape">
          <li v-for="item in tape" :key="item.symbol">
            <button type="button" :aria-pressed="item.symbol === selected"
                    :class="{ 'is-on': item.symbol === selected }" @click="selected = item.symbol">
              <b>{{ item.symbol }}</b>
              <em>{{ spot(item.spot) }}</em>
              <i :class="item.change >= 0 ? 'is-up' : 'is-down'">{{ pct(item.change) }}</i>
              <small>K {{ money(item.model.strike) }}</small>
            </button>
          </li>
        </ul>
      </section>

      <!-- briefing -->
      <section class="col-c1" aria-labelledby="brief-title">
        <div class="section-head">
          <h2 id="brief-title" class="h2">Briefing</h2>
          <p class="section-meta">Four leaves</p>
        </div>
        <ol class="brief">
          <li v-for="[num, title, body, to] in briefing" :key="num">
            <a :href="to" @click.prevent="to.startsWith('#') ? toBoundary() : emit('navigate', to)">
              <b>{{ num }}</b>
              <span>
                <strong>{{ title }}</strong>
                <em>{{ body }}</em>
              </span>
              <ArrowRight :size="16" aria-hidden="true" />
            </a>
          </li>
        </ol>
      </section>

      <!-- disclosure -->
      <section class="col-c2 panel panel-ink" aria-labelledby="disc-title">
        <div class="panel-head">
          <h2 id="disc-title">Limits of this preview</h2>
        </div>
        <div class="panel-body">
          <ul class="disclosure">
            <li>Quotes are illustrative samples, not a market feed.</li>
            <li>The payoff model uses values you enter.</li>
            <li>Journal records stay in this browser's storage.</li>
            <li>No custody, no signing, no trade execution.</li>
            <li>No contract address is published or implied.</li>
          </ul>
          <button class="btn btn-flare btn-full" type="button" @click="emit('navigate', '/risk')">Read the limits</button>
        </div>
      </section>
    </div>
  </div>
</template>

<style scoped>
/* ---------------------------------------------------------------- masthead */
.mast {
  display: grid;
  grid-template-columns: minmax(0, 1.25fr) minmax(0, 1fr);
  gap: 34px;
  align-items: end;
  padding: 34px 0 28px;
}
.mast-head .h1 { margin-top: 6px; }
.mast-head .h1 em { color: var(--flare); font-style: normal; }
.mast-side { display: grid; gap: 16px; align-content: end; }
.mast-actions { display: flex; flex-wrap: wrap; gap: 10px; }
.mast-note {
  margin: 0;
  padding-top: 12px;
  border-top: var(--hair-w) solid var(--hair);
  color: var(--muted);
  font: 11px/1.7 var(--mono);
}

/* ------------------------------------------------------------- ruled grid */
.editorial {
  display: grid;
  grid-template-columns: repeat(12, minmax(0, 1fr));
  border-top: var(--rule-w) solid var(--ink);
}
.editorial > * {
  min-width: 0;
  padding: 20px;
  border-bottom: var(--hair-w) solid var(--hair);
  border-right: var(--hair-w) solid var(--hair);
}
.editorial > .col-b1,
.editorial > .col-b2,
.editorial > .col-c2 { border-right: 0; }
.editorial > .col-c1,
.editorial > .col-c2 { border-bottom: 0; }

.col-a1 { grid-column: span 8; }
.col-b1 { grid-column: span 4; }
.col-a2 { grid-column: span 8; }
.col-b2 { grid-column: span 4; }
.col-c1 { grid-column: span 7; }
.col-c2 { grid-column: span 5; }

.section-head {
  display: flex; align-items: baseline; justify-content: space-between; gap: 16px;
  flex-wrap: wrap;
  padding-bottom: 12px;
  border-bottom: var(--rule-w) solid var(--ink);
  margin-bottom: 4px;
}
.section-meta { margin: 0; color: var(--muted); font: 11px/1.4 var(--mono); }
.section-tools { display: flex; align-items: center; gap: 10px; flex-wrap: wrap; }

.search input {
  min-height: 34px;
  min-width: 190px;
  padding: 0 11px;
  border: var(--rule-w) solid var(--ink);
  background: var(--card);
  font: 600 11px/1 var(--mono);
}
.search input::placeholder { color: var(--faint); }

/* ----------------------------------------------------------------- board */
.board { margin-top: 0; }
.board th[scope='row'] { padding: 0; }
.board .token {
  width: 100%;
  padding: 9px 12px;
  border: 0;
  background: transparent;
  text-align: left;
  cursor: pointer;
}
.board .token:hover { background: var(--paper); }
.board .is-loud { color: var(--flare); }
.rowlink {
  border: 0;
  padding: 0;
  background: none;
  color: var(--signal);
  font: 700 10px/1 var(--mono); letter-spacing: .1em; text-transform: uppercase;
  text-decoration: none;
  cursor: pointer;
}
.rowlink:hover { color: var(--flare); text-decoration: underline; }
.empty { padding: 22px 12px; color: var(--muted); font: 12px/1.6 var(--mono); }
.foot-note {
  margin: 12px 0 0;
  color: var(--faint);
  font: 10px/1.6 var(--mono);
}

/* ----------------------------------------------------------- quote panel */
.readout { margin: 18px 0 0; padding: 0; border-top: var(--rule-w) solid var(--ink); }
.readout > div {
  display: flex; align-items: baseline; justify-content: space-between; gap: 10px;
  padding: 8px 0;
  border-bottom: var(--hair-w) solid var(--hair);
}
.readout dt { color: var(--muted); font: 10px/1.4 var(--mono); letter-spacing: .1em; text-transform: uppercase; }
.readout dd { margin: 0; font: 600 12px/1.4 var(--mono); font-variant-numeric: tabular-nums; }
.total {
  display: flex; align-items: baseline; justify-content: space-between; gap: 10px;
  margin: 14px 0;
  padding: 12px 0 0;
  border-top: var(--hair-w) solid var(--hair);
  color: var(--muted);
  font: 10px/1.3 var(--mono); letter-spacing: .12em; text-transform: uppercase;
}
.total strong { color: var(--ink); font: 900 26px/1 var(--display); letter-spacing: -.01em; }

/* ------------------------------------------------------------ split figure */
.split-grid { display: grid; grid-template-columns: minmax(0, 1.5fr) minmax(0, 1fr); gap: 20px; align-items: start; }
.payoff { margin: 0; }
.payoff svg { width: 100%; height: auto; display: block; border: var(--hair-w) solid var(--hair); background: var(--paper); }
.payoff figcaption { margin-top: 10px; color: var(--muted); font: 11px/1.6 var(--mono); }
.figure-grid line { stroke: var(--hair); stroke-width: 1; }
.figure-axis { stroke: var(--ink); stroke-width: 2; }
.figure-cap { stroke: var(--flare); stroke-width: 3; }
.figure-label { fill: var(--flare); font: 700 11px var(--mono); }
.figure-band { fill: var(--ink); font: 700 12px var(--mono); letter-spacing: .14em; }
.figure-value { fill: var(--muted); font: 11px var(--mono); }
.figure-tick { fill: var(--muted); font: 10px var(--mono); }
.band-income { fill: #d8ece4; }
.band-upside { fill: #f7dcd3; }
.split-readout { display: grid; margin: 0; border-top: var(--rule-w) solid var(--ink); }
.split-readout > div {
  display: flex; align-items: baseline; justify-content: space-between; gap: 10px;
  padding: 9px 0;
  border-bottom: var(--hair-w) solid var(--hair);
}
.split-readout dt { color: var(--muted); font: 10px/1.4 var(--mono); letter-spacing: .08em; text-transform: uppercase; }
.split-readout dd { margin: 0; font: 600 13px/1.4 var(--mono); font-variant-numeric: tabular-nums; }
.split-readout dd.is-up { color: var(--ledger); }
.split-readout dd.is-down { color: var(--alert); }

/* ------------------------------------------------------------------ tape */
.tape { display: grid; gap: 0; margin: 4px 0 0; padding: 0; list-style: none; }
.tape button {
  display: grid;
  grid-template-columns: 60px minmax(0, 1fr) auto auto;
  align-items: baseline;
  gap: 12px;
  width: 100%;
  padding: 13px 0;
  border: 0;
  border-bottom: var(--hair-w) solid var(--hair);
  background: transparent;
  text-align: left;
  cursor: pointer;
}
.tape li:last-child button { border-bottom: 0; }
.tape button:hover { background: var(--paper); }
.tape button.is-on { box-shadow: inset 3px 0 0 var(--flare); padding-left: 10px; }
.tape b { font: 700 13px/1.3 var(--mono); }
.tape em { font: 600 13px/1.3 var(--mono); font-style: normal; font-variant-numeric: tabular-nums; }
.tape i { font: 600 12px/1.3 var(--mono); font-style: normal; }
.tape i.is-up { color: var(--ledger); }
.tape i.is-down { color: var(--alert); }
.tape small { color: var(--faint); font: 11px/1.3 var(--mono); }

/* -------------------------------------------------------------- briefing */
.brief { display: grid; margin: 4px 0 0; padding: 0; list-style: none; }
.brief li { border-bottom: var(--hair-w) solid var(--hair); }
.brief li:last-child { border-bottom: 0; }
.brief a {
  display: grid;
  grid-template-columns: 40px minmax(0, 1fr) auto;
  align-items: center;
  gap: 16px;
  padding: 16px 0;
  text-decoration: none;
}
.brief a:hover { background: var(--paper); }
.brief a:hover strong { color: var(--flare); }
.brief b { color: var(--faint); font: 700 12px/1 var(--mono); }
.brief span { display: grid; gap: 4px; min-width: 0; }
.brief strong { font: 700 16px/1.2 var(--sans); transition: color .12s; }
.brief em { color: var(--muted); font: 12px/1.6 var(--sans); font-style: normal; }

/* ------------------------------------------------------------ disclosure */
.disclosure { display: grid; gap: 10px; margin: 0 0 18px; padding: 0; list-style: none; }
.disclosure li {
  position: relative;
  padding-left: 18px;
  color: var(--on-ink-2);
  font: 12px/1.6 var(--mono);
}
.disclosure li::before {
  position: absolute; top: 7px; left: 0;
  width: 8px; height: 2px;
  content: '';
  background: var(--flare);
}

/* ---------------------------------------------------------------- responsive */
@media (max-width: 1180px) {
  .mast { grid-template-columns: minmax(0, 1fr); gap: 24px; }
  .col-a1, .col-a2 { grid-column: span 12; }
  .col-b1, .col-b2 { grid-column: span 6; }
  .col-b2 { border-right: 0; }
  .col-b1 { border-right: var(--hair-w) solid var(--hair); }
  .col-c1, .col-c2 { grid-column: span 12; }
  .col-c1 { border-right: 0; }
}
@media (max-width: 860px) {
  .editorial > * { padding: 16px 0; border-right: 0; }
  .col-b1, .col-b2 { grid-column: span 12; }
  .split-grid { grid-template-columns: minmax(0, 1fr); }
  .mast { padding: 22px 0 20px; }
  .board thead th:nth-child(3), .board tbody td:nth-child(3) { display: none; }
}
@media (max-width: 620px) {
  .board thead th:nth-child(2), .board tbody td:nth-child(2),
  .board thead th:nth-child(5), .board tbody td:nth-child(5) { display: none; }
  .section-tools { width: 100%; }
  .search, .search input { width: 100%; min-width: 0; }
  .tape button { grid-template-columns: 54px minmax(0, 1fr) auto; }
  .tape small { display: none; }
  .brief a { grid-template-columns: 30px minmax(0, 1fr) auto; gap: 12px; }
}
</style>
