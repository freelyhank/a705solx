<script setup>
import { computed, ref, watch } from 'vue'
import { ArrowDown, ArrowUp } from 'lucide-vue-next'
import { calculatePayoff } from '../payoff.js'

const props = defineProps({ spot: { type: Number, default: 250 }, asset: { type: String, default: 'SOL' } })
const cap = ref(5)
const settlement = ref(props.spot)
const showTable = ref(false)
watch(() => props.spot, spot => { settlement.value = spot })
const model = computed(() => calculatePayoff({ spot: props.spot, capPercent: cap.value, settlementPrice: settlement.value }))
const money = value => new Intl.NumberFormat('en-US', { style: 'currency', currency: 'USD', minimumFractionDigits: 2, maximumFractionDigits: 2 }).format(value)
const change = computed(() => `${model.value.spotChangePercent > 0 ? '+' : ''}${model.value.spotChangePercent.toFixed(1)}%`)
const low = computed(() => props.spot * 0.7)
const high = computed(() => props.spot * 1.3)
const chartMax = computed(() => Math.max(high.value * 1.45, 2))
const gridTicks = computed(() => Array.from({ length: 5 }, (_, index) => chartMax.value * index / 4))
const x = value => 74 + (value - low.value) / (high.value - low.value) * 454
const y = value => 276 - value / chartMax.value * 232
const stockPath = computed(() => `M ${x(low.value)} ${y(low.value)} L ${x(high.value)} ${y(high.value)}`)
const incomePath = computed(() => `M ${x(low.value)} ${y(low.value)} L ${x(model.value.strike)} ${y(model.value.strike)} L ${x(high.value)} ${y(model.value.strike)}`)
const upsidePath = computed(() => `M ${x(low.value)} ${y(0)} L ${x(model.value.strike)} ${y(0)} L ${x(high.value)} ${y(high.value - model.value.strike)}`)
const tableRows = computed(() => [...new Set([low.value, props.spot, model.value.strike, high.value, settlement.value])].sort((a, b) => a - b).map(price => ({
  price,
  income: Math.min(price, model.value.strike),
  upside: Math.max(price - model.value.strike, 0),
})))
const chartDescription = computed(() => `At a settlement price of ${money(settlement.value)}, Income is worth ${money(model.value.incomeSettlement)} and Upside is worth ${money(model.value.upsideSettlement)} per unit. Income follows the stock up to the cap of ${money(model.value.strike)}. Upside receives any amount above the cap.`)
</script>

<template>
  <section id="cap" class="payoff" aria-labelledby="cap-heading">
    <div class="payoff-layout">
      <div class="cap-copy">
        <h2 id="cap-heading" class="h2">A defined level.<br />A clear split.</h2>
        <p class="cap-lead">Choose the cap. K is fixed for the epoch.</p>
        <p class="cap-explanation">Set a cap, move the settlement slider, and inspect how the illustrative value splits between Income and Upside. No order or wallet action is created.</p>

        <div class="cap-control">
          <div class="control-label"><label for="cap-range">Cap</label><output for="cap-range">+{{ cap }}%</output></div>
          <div class="range-wrap" :style="{ '--range-fill': `${cap * 10}%` }">
            <span class="range-track" aria-hidden="true"><i></i></span>
            <input id="cap-range" v-model.number="cap" class="range" type="range" min="0" max="10" step="1" :aria-valuetext="`${cap}% cap, strike ${money(model.strike)}`" aria-describedby="cap-model-note" />
          </div>
          <div class="cap-ticks" aria-hidden="true"><span>0%</span><span>2%</span><span :class="{ 'selected-tick': cap === 5 }">5%</span><span>10%</span></div>
          <p id="cap-model-note" class="model-note">{{ asset }} example at <span>{{ money(model.spot) }}</span> / cap price (K) <span>{{ money(model.strike) }}</span> / 30 days / model at 25% vol</p>
        </div>
      </div>

      <div class="payoff-visual">
        <div class="settlement-card panel">
          <div class="panel-head chart-heading">
            <div>
              <h3>{{ asset }} value at settlement</h3>
              <p>Per unit / {{ asset }} example at {{ money(model.spot) }} / cap +{{ cap }}% / K {{ money(model.strike) }}</p>
            </div>
            <button class="btn btn-quiet table-toggle" type="button" :aria-pressed="showTable" aria-controls="payoff-display" @click="showTable = !showTable">{{ showTable ? 'Chart' : 'Table' }}</button>
          </div>
          <div class="panel-body">
            <ul class="chart-legend" aria-label="Chart legend"><li class="legend-stock">{{ asset }} market unit</li><li class="legend-income">Income</li><li class="legend-upside">Upside</li></ul>

            <div id="payoff-display" class="payoff-display">
              <div v-if="showTable" class="payoff-table-wrap" tabindex="0" role="region" aria-label="Settlement values table">
                <table class="ledger payoff-table"><caption>Per-unit settlement value, before premium</caption><thead><tr><th scope="col">Stock price</th><th scope="col" class="num">Income</th><th scope="col" class="num">Upside</th></tr></thead><tbody><tr v-for="row in tableRows" :key="row.price" :class="{ 'is-mark': row.price === settlement }"><th scope="row">{{ money(row.price) }}<span v-if="row.price === settlement" class="current-label">Current</span></th><td class="num">{{ money(row.income) }}</td><td class="num">{{ money(row.upside) }}</td></tr></tbody></table>
              </div>
              <svg v-else class="payoff-chart" viewBox="0 0 650 320" role="img" aria-labelledby="payoff-chart-title payoff-chart-description">
                <title id="payoff-chart-title">Stock, Income and Upside settlement values</title><desc id="payoff-chart-description">{{ chartDescription }}</desc>
                <g class="chart-grid"><template v-for="tick in gridTicks" :key="tick"><line x1="74" x2="528" :y1="y(tick)" :y2="y(tick)"/><text x="63" :y="y(tick) + 5" text-anchor="end">{{ money(tick) }}</text></template></g>
                <g class="chart-axis"><text v-for="tick in [low, model.spot, high]" :key="tick" :x="x(tick)" y="301" text-anchor="middle">{{ money(tick) }}</text></g>
                <line class="strike-line" :x1="x(model.strike)" :x2="x(model.strike)" y1="44" y2="276"/><text class="reference-label" :x="x(model.strike)" y="34" text-anchor="middle">K {{ money(model.strike) }}</text>
                <text class="reference-label" :x="x(model.spot)" y="14" text-anchor="middle">SPOT {{ money(model.spot) }}</text>
                <path class="stock-line" :d="stockPath"/><path class="income-line" :d="incomePath"/><path class="upside-line" :d="upsidePath"/>
                <line class="settlement-line" :x1="x(settlement)" :x2="x(settlement)" y1="44" y2="276"/>
                <text class="settlement-label" :x="x(settlement) + (settlement > 285 ? -12 : 12)" y="66" :text-anchor="settlement > 285 ? 'end' : 'start'">S {{ money(settlement) }}</text>
                <circle class="income-point" :cx="x(settlement)" :cy="y(model.incomeSettlement)" r="7"/><circle class="upside-point" :cx="x(settlement)" :cy="y(model.upsideSettlement)" r="8"/>
                <g class="endpoint stock-endpoint"><circle cx="541" :cy="y(high)" r="3.5"/><text x="550" :y="y(high) + 5">{{ money(high) }}</text></g>
                <g class="endpoint income-endpoint"><circle cx="541" :cy="y(model.strike)" r="3.5"/><text x="550" :y="y(model.strike) + 5">{{ money(model.strike) }}</text></g>
                <g class="endpoint upside-endpoint"><circle cx="541" :cy="y(high - model.strike)" r="3.5"/><text x="550" :y="y(high - model.strike) + 5">{{ money(high - model.strike) }}</text></g>
              </svg>
            </div>

            <div class="settlement-control">
              <div class="settlement-label-row"><label for="settlement-range">Settlement price S</label><output for="settlement-range">{{ money(settlement) }} <small>{{ change }} vs spot</small></output></div>
              <div class="range-wrap" :style="{ '--range-fill': `${(settlement - low) / (high - low) * 100}%` }">
                <span class="range-track" aria-hidden="true"><i></i></span>
                <input id="settlement-range" v-model.number="settlement" class="range" type="range" :min="low" :max="high" :step="Math.max(props.spot / 100, 0.000001)" :aria-valuetext="`${money(settlement)}, ${change} versus initial stock price`" aria-describedby="settlement-summary" />
              </div>
            </div>
            <div id="settlement-summary" class="settlement-values" aria-live="polite" aria-atomic="true">
              <div class="value-row"><span class="value-label legend-income">Income /<br> min(S, K)</span><div><strong>{{ money(model.incomeSettlement) }}</strong><small> + premium {{ money(model.premium) }} if subscribed (model)<br> = {{ money(model.incomeWithPremium) }}</small></div></div>
              <div class="value-row"><span class="value-label legend-upside">Upside / max(S - K, 0)</span><div><strong>{{ money(model.upsideSettlement) }}</strong><small> breakeven {{ money(model.breakeven) }} (model)</small></div></div>
            </div>
            <p class="chart-disclaimer">Indicative premium <b>{{ money(model.premium) }}</b> per unit / model at 25% vol / 30 days / the auction sets the price. Values per unit; Income is paid in SPL market units worth min(S, K); the premium goes to subscribers only.</p>
          </div>
        </div>

        <div class="position-meters">
          <div class="position-card income-card"><div class="position-label"><span class="position-mark"><ArrowDown :size="14" /></span><b>INCOME</b><span>(Up to the cap)</span></div><div class="position-price"><strong>{{ money(model.incomePrice) }}</strong><span>~{{ model.incomeShare.toFixed(1) }}%</span></div><div class="position-track" aria-hidden="true"><span :style="{ width: `${model.incomeShare}%` }"></span></div></div>
          <div class="position-card upside-card"><div class="position-label"><span class="position-mark"><ArrowUp :size="14" /></span><b>UPSIDE</b><span>(Above the cap)</span></div><div class="position-price"><strong>{{ money(model.upsidePrice) }}</strong><span>~{{ model.upsideShare.toFixed(1) }}%</span></div><div class="position-track" aria-hidden="true"><span :style="{ width: `${model.upsideShare}%` }"></span></div></div>
        </div>
      </div>
    </div>
  </section>
</template>

<style scoped>
/* ---------------------------------------------------------------- figure */
.payoff { min-width: 0; scroll-margin-top: calc(var(--strip-h) + 16px); }
.payoff-layout {
  display: grid;
  grid-template-columns: minmax(0, 220px) minmax(0, 1fr);
  gap: 20px;
}
.cap-copy { display: grid; gap: 10px; align-content: start; }
.cap-lead {
  margin: 0;
  color: var(--flare);
  font: 700 10px/1.4 var(--mono); letter-spacing: .16em; text-transform: uppercase;
}
.cap-explanation { margin: 0; color: var(--fg-2); font-size: 13px; line-height: 1.6; }
.cap-control { margin-top: 8px; }
.control-label, .settlement-label-row {
  display: flex; align-items: baseline; justify-content: space-between; gap: 10px;
  color: var(--muted);
  font: 700 10px/1.3 var(--mono); letter-spacing: .12em; text-transform: uppercase;
}
.control-label output, .settlement-label-row output { color: var(--ink); font-size: 12px; letter-spacing: 0; }
.settlement-label-row output small { color: var(--muted); font-size: 10px; }
.cap-ticks {
  display: flex; justify-content: space-between;
  margin-top: 2px;
  color: var(--faint);
  font: 10px/1 var(--mono);
}
.cap-ticks .selected-tick { color: var(--flare); font-weight: 700; }
.model-note { margin: 10px 0 0; color: var(--faint); font: 10px/1.5 var(--mono); }
.model-note span { color: var(--fg-2); }

/* ----------------------------------------------------------- range tracks */
.range-wrap { position: relative; display: grid; align-items: center; margin-top: 6px; }
.range-track {
  position: absolute; top: 50%; left: 0; right: 0;
  height: 8px;
  transform: translateY(-50%);
  border: var(--hair-w) solid var(--ink);
  background: var(--sunken);
  pointer-events: none;
}
.range-track i { display: block; width: var(--range-fill, 0%); height: 100%; background: var(--flare); }
.range-wrap .range {
  position: relative; z-index: 1;
  width: 100%; height: 26px; margin: 0;
  appearance: none; -webkit-appearance: none;
  background: transparent;
  cursor: ew-resize;
}
.range-wrap .range::-webkit-slider-runnable-track { height: 26px; border: 0; background: transparent; }
.range-wrap .range::-webkit-slider-thumb {
  -webkit-appearance: none; appearance: none;
  width: 10px; height: 26px;
  margin-top: 0;
  border: var(--rule-w) solid var(--ink);
  border-radius: 0;
  background: var(--card);
}
.range-wrap .range::-moz-range-track { height: 26px; border: 0; background: transparent; }
.range-wrap .range::-moz-range-thumb {
  width: 10px; height: 26px;
  border: var(--rule-w) solid var(--ink);
  border-radius: 0;
  background: var(--card);
}

/* ----------------------------------------------------------------- chart */
.payoff-visual { display: grid; gap: 16px; min-width: 0; }
.settlement-card .panel-head { align-items: flex-start; }
.settlement-card .chart-heading h3 { font: 700 15px/1.25 var(--sans); text-transform: none; }
.settlement-card .chart-heading p { margin: 4px 0 0; color: var(--muted); font: 10px/1.5 var(--mono); }
.table-toggle { min-height: 32px; padding: 0 12px; flex: none; }
.chart-legend {
  display: flex; flex-wrap: wrap; gap: 16px;
  margin: 0 0 10px; padding: 0;
  list-style: none;
  color: var(--muted);
  font: 10px/1 var(--mono); letter-spacing: .1em; text-transform: uppercase;
}
.chart-legend li { position: relative; padding-left: 14px; }
.chart-legend li::before { position: absolute; top: 50%; left: 0; width: 8px; height: 3px; content: ''; background: currentColor; }
.legend-stock { color: var(--ink); }
.legend-income { color: var(--ledger); }
.legend-upside { color: var(--alert); }

.payoff-display { border: var(--hair-w) solid var(--hair); background: var(--paper); }
.payoff-chart { display: block; width: 100%; height: auto; }
.chart-grid line { stroke: var(--hair); stroke-width: 1; }
/* Chart type is 11px: at 10px the monospace zero renders as a slashed cell that reads
   as a missing-glyph box. 11px keeps the ledger character without the ambiguity. */
.chart-grid text { fill: var(--faint); font-family: var(--mono); font-size: 11px; font-variant-numeric: tabular-nums; }
.chart-axis text { fill: var(--muted); font-family: var(--mono); font-size: 11px; font-variant-numeric: tabular-nums; }
.strike-line { stroke: var(--flare); stroke-width: 3; }
.reference-label { fill: var(--flare); font-family: var(--mono); font-size: 11px; font-weight: 700; letter-spacing: .02em; }
.stock-line { fill: none; stroke: var(--ink); stroke-width: 2; }
.income-line { fill: none; stroke: var(--ledger); stroke-width: 3; }
.upside-line { fill: none; stroke: var(--alert); stroke-width: 3; }
.settlement-line { stroke: var(--ink); stroke-width: 1; stroke-dasharray: 5 4; }
.settlement-label { fill: var(--ink); font-family: var(--mono); font-size: 11px; font-weight: 700; }
.income-point { fill: var(--ledger); stroke: var(--ink); stroke-width: 1; }
.upside-point { fill: var(--alert); stroke: var(--ink); stroke-width: 1; }
.endpoint circle { fill: var(--ink); }
.endpoint text { fill: var(--muted); font-family: var(--mono); font-size: 11px; font-variant-numeric: tabular-nums; }

.payoff-table-wrap { overflow-x: auto; }
.payoff-table { margin: 0; }
.payoff-table caption {
  padding: 11px 12px;
  border-bottom: var(--hair-w) solid var(--hair);
  color: var(--muted);
  font: 10px/1.4 var(--mono); letter-spacing: .12em; text-transform: uppercase;
}
.payoff-table th[scope='row'] { font: 600 13px/1.4 var(--mono); font-variant-numeric: tabular-nums; white-space: nowrap; }
.current-label {
  display: inline-block;
  margin-left: 8px;
  padding: 2px 6px;
  border: var(--hair-w) solid var(--flare);
  color: var(--flare);
  font: 700 9px/1 var(--mono); letter-spacing: .1em; text-transform: uppercase;
  vertical-align: 2px;
}

/* -------------------------------------------------------- settlement read */
.settlement-control { margin-top: 16px; }
.settlement-values { display: grid; margin-top: 14px; border-top: var(--rule-w) solid var(--ink); }
.value-row {
  display: grid; grid-template-columns: minmax(0, 1fr) auto;
  align-items: baseline; gap: 14px;
  padding: 11px 0;
  border-bottom: var(--hair-w) solid var(--hair);
}
.value-label { font: 700 10px/1.4 var(--mono); letter-spacing: .1em; text-transform: uppercase; }
.value-row > div { text-align: right; }
.value-row strong { font: 600 19px/1.2 var(--mono); font-variant-numeric: tabular-nums; }
.value-row small { display: block; margin-top: 3px; color: var(--faint); font: 10px/1.5 var(--mono); }
.chart-disclaimer {
  margin: 14px 0 0;
  padding-top: 12px;
  border-top: var(--hair-w) solid var(--hair);
  color: var(--muted);
  font: 10px/1.6 var(--mono);
}
.chart-disclaimer b { color: var(--ink); }

/* -------------------------------------------------------- position meters */
.position-meters {
  display: grid;
  grid-template-columns: repeat(2, minmax(0, 1fr));
  border: var(--rule-w) solid var(--ink);
  background: var(--card);
}
.position-card { display: grid; gap: 10px; padding: 14px 16px; min-width: 0; }
.position-card + .position-card { border-left: var(--hair-w) solid var(--hair); }
.position-label { display: flex; align-items: center; gap: 8px; }
.position-label b { font: 700 11px/1 var(--mono); letter-spacing: .14em; }
.position-label > span:last-child { color: var(--faint); font: 10px/1.3 var(--mono); }
.position-mark { display: grid; place-items: center; width: 22px; height: 22px; border: var(--hair-w) solid var(--ink); }
.income-card .position-mark { background: var(--ledger); border-color: var(--ledger); color: #fff; }
.upside-card .position-mark { background: var(--alert); border-color: var(--alert); color: #fff; }
.position-price { display: flex; align-items: baseline; justify-content: space-between; gap: 10px; }
.position-price strong { font: 600 19px/1.2 var(--mono); font-variant-numeric: tabular-nums; }
.position-price span { color: var(--muted); font: 11px/1.3 var(--mono); font-variant-numeric: tabular-nums; }
.position-track {
  height: 10px;
  border: var(--hair-w) solid var(--ink);
  background: var(--card);
}
.position-track span { display: block; height: 100%; }
.income-card .position-track span { background: var(--ledger); }
.upside-card .position-track span { background: var(--alert); }

/* ------------------------------------------------------------ responsive */
@media (max-width: 1100px) {
  .payoff-layout { grid-template-columns: minmax(0, 1fr); }
}
@media (max-width: 520px) {
  .position-meters { grid-template-columns: minmax(0, 1fr); }
  .position-card + .position-card { border-left: 0; border-top: var(--hair-w) solid var(--hair); }
  .value-row { grid-template-columns: minmax(0, 1fr); }
  .value-row > div { text-align: left; }
}
</style>
