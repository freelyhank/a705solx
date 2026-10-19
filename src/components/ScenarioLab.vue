<script setup>
import { computed, nextTick, onMounted, ref } from 'vue'
import { ArrowDownToLine, ArrowUpRight, Check, ChevronRight, FlaskConical, ShieldCheck } from 'lucide-vue-next'
import { brand, storageKeys } from '../brand.js'
import { DEFAULT_BASE_PRICE, MAX_NOTE_LENGTH, calculateScenario, createJournalRecord, restoreJournalRecord, validateScenarioInputs } from '../scenario.js'

defineProps({ wallet: { type: String, default: '' } })

const root = ref(null)
const heading = ref(null)
const amount = ref('1000')
const basePrice = ref(DEFAULT_BASE_PRICE)
const movePct = ref('-15')
const note = ref('')
const step = ref('form')
const record = ref(null)
const errors = ref({})
const feedback = ref('')
const storageNotice = ref('')
const isSaved = ref(false)
const stages = ['Inputs recorded', 'Assumption reviewed', 'Limits reviewed', 'Review complete']
const legacyJournalKeys = ['Tikriva:journal:v1', 'Orbinza:journal:v1', 'Kovrane:journal:v1', 'kinovra:journal:v1', 'Kinovra:journal:v1', 'Scenarill:journal:v1', 'Solenzi:journal:v1']
const input = computed(() => ({ amount: amount.value, basePrice: basePrice.value, movePct: movePct.value, note: note.value }))
const result = computed(() => record.value ? calculateScenario(record.value) : null)
const remaining = computed(() => MAX_NOTE_LENGTH - note.value.length)
const currency = (value, digits = 2) => new Intl.NumberFormat('en-US', { style: 'currency', currency: 'USD', minimumFractionDigits: digits, maximumFractionDigits: digits }).format(value)
const percent = value => `${Number(value) > 0 ? '+' : ''}${Number(value).toFixed(2)}%`

async function focusHeading() {
  await nextTick()
  heading.value?.focus({ preventScroll: true })
}

function fillInputs(value) {
  amount.value = String(value.amount)
  basePrice.value = String(value.basePrice)
  movePct.value = String(value.movePct)
  note.value = value.note
}

async function review() {
  const validation = validateScenarioInputs(input.value)
  errors.value = validation.errors
  feedback.value = ''
  if (!validation.valid) {
    await nextTick()
    root.value?.querySelector('[aria-invalid="true"]')?.focus()
    return
  }
  record.value = createJournalRecord(validation.value, { id: `SC-${crypto.randomUUID()}`, createdAt: new Date().toISOString() })
  isSaved.value = false
  step.value = 'review'
  await focusHeading()
}

function persist() {
  if (!record.value) return false
  feedback.value = ''
  try {
    localStorage.setItem(storageKeys.journal, JSON.stringify(record.value))
    isSaved.value = true
    storageNotice.value = ''
    feedback.value = 'Saved locally on this device.'
    return true
  } catch {
    isSaved.value = false
    storageNotice.value = 'This record could not be saved to your browser. Export JSON to keep a copy, or retry saving.'
    return false
  }
}

async function save() {
  persist()
  step.value = 'journal'
  await focusHeading()
}

async function edit() {
  if (record.value) fillInputs(record.value)
  errors.value = {}
  feedback.value = ''
  step.value = 'form'
  await focusHeading()
}

async function removeJournal() {
  feedback.value = ''
  try {
    // Remove the older namespace too, so a deleted record cannot reappear on reload.
    legacyJournalKeys.forEach(key => localStorage.removeItem(key))
    localStorage.removeItem(storageKeys.journal)
  } catch {
    storageNotice.value = 'The browser could not delete your saved journal. Your current record is still available; try again or export a copy.'
    return
  }
  record.value = null
  isSaved.value = false
  step.value = 'form'
  amount.value = '1000'
  basePrice.value = DEFAULT_BASE_PRICE
  movePct.value = '-15'
  note.value = ''
  errors.value = {}
  storageNotice.value = ''
  feedback.value = 'Local journal removed.'
  await focusHeading()
}

function changeStage(stage) {
  if (!record.value) return
  record.value = { ...record.value, stage: Math.max(0, Math.min(3, stage)) }
  persist()
}

function downloadJournal() {
  if (!record.value) return
  const data = {
    ...record.value,
    calculations: result.value,
    disclaimer: 'User-entered price and move. Not a quote or forecast. This interface does not enable wallet transactions.',
  }
  const url = URL.createObjectURL(new Blob([JSON.stringify(data, null, 2)], { type: 'application/json' }))
  const link = document.createElement('a')
  link.href = url
  link.download = `ivorane-${record.value.id.toLowerCase()}.json`
  link.click()
  URL.revokeObjectURL(url)
  feedback.value = 'Journal export prepared.'
}

onMounted(() => {
  try {
    const activeRaw = localStorage.getItem(storageKeys.journal)
    const legacyKey = legacyJournalKeys.find(key => localStorage.getItem(key))
    const legacyRaw = legacyKey ? localStorage.getItem(legacyKey) : null
    const raw = activeRaw || legacyRaw
    if (!raw) return
    const restored = restoreJournalRecord(JSON.parse(raw))
    if (!restored.valid) {
      storageNotice.value = 'The saved journal has an unsupported format. You can start a new scenario.'
      return
    }
    record.value = restored.value
    fillInputs(restored.value)
    step.value = 'journal'
    isSaved.value = true
    if (restored.migrated || legacyRaw) {
      if (persist()) feedback.value = 'Your earlier record is ready. Its original price is retained as your own assumption.'
    }
  } catch {
    storageNotice.value = 'Saved journal could not be restored. You can still calculate a scenario and export it.'
  }
})

defineExpose({ focusHeading })
</script>

<template>
  <section ref="root" class="scenario-lab sheet-body" aria-labelledby="scenario-title">
    <header class="sl-header">
      <p class="kick sl-eyebrow"><FlaskConical :size="15" /> YOUR PRIVATE WORKSPACE</p>
      <h1 id="scenario-title" ref="heading" class="h1" tabindex="-1">Local scenario lab<span>.</span></h1>
      <p class="lede">Put a number behind your next idea. Enter a reference price, explore a hypothetical move, and keep your reasoning close.</p>
    </header>
    <p v-if="storageNotice" class="notice notice-alert" role="alert">{{ storageNotice }}</p>
    <p v-if="feedback" class="notice notice-ok" role="status">{{ feedback }}</p>

    <div v-if="step === 'form'" class="sl-layout">
      <form class="sl-card sl-form" @submit.prevent="review">
        <div class="sl-card-heading"><div><span class="sl-step">01 / INPUTS</span><h2 class="h2">Set your assumption.</h2></div><span class="chip chip-flare">LOCAL ONLY</span></div>
        <div class="sl-field-pair">
          <div class="formrow">
            <label class="flabel" for="amount">Amount <span>{{ brand.asset }} units</span></label>
            <input id="amount" class="input" v-model="amount" inputmode="decimal" :aria-invalid="Boolean(errors.amount)" :aria-describedby="errors.amount ? 'amount-error' : undefined" />
            <p v-if="errors.amount" id="amount-error" class="sl-error">{{ errors.amount }}</p>
          </div>
          <div class="formrow">
            <label class="flabel" for="base-price">Reference price <span>USD / {{ brand.asset }}</span></label>
            <input id="base-price" class="input" v-model="basePrice" inputmode="decimal" :aria-invalid="Boolean(errors.basePrice)" :aria-describedby="errors.basePrice ? 'price-error' : undefined" />
            <p v-if="errors.basePrice" id="price-error" class="sl-error">{{ errors.basePrice }}</p>
          </div>
        </div>
        <div class="formrow">
          <label class="flabel" for="move">Hypothetical move <output>{{ movePct }}%</output></label>
          <input id="move" v-model="movePct" class="range sl-range" type="range" min="-90" max="100" step="0.5" />
          <div class="range-scale"><span>-90%</span><span>0</span><span>+100%</span></div>
        </div>
        <div class="formrow">
          <label class="flabel" for="move-text">Exact percentage <span>%</span></label>
          <input id="move-text" class="input" v-model="movePct" inputmode="decimal" :aria-invalid="Boolean(errors.movePct)" :aria-describedby="errors.movePct ? 'move-error' : undefined" />
          <p v-if="errors.movePct" id="move-error" class="sl-error">{{ errors.movePct }}</p>
        </div>
        <div class="formrow">
          <label class="flabel" for="note">Thesis note <span>OPTIONAL</span></label>
          <textarea id="note" class="input" v-model="note" :maxlength="MAX_NOTE_LENGTH" :aria-invalid="Boolean(errors.note)" :aria-describedby="errors.note ? 'note-error' : 'note-count'" placeholder="What would make this move plausible?"></textarea>
          <p v-if="errors.note" id="note-error" class="sl-error">{{ errors.note }}</p>
          <p id="note-count" class="sl-char-count">{{ remaining }} characters remaining</p>
        </div>
        <button class="btn btn-primary btn-full" type="submit">Review scenario <ArrowUpRight :size="17" /></button>
      </form>
      <aside class="sl-card sl-aside"><div class="sl-aside-icon"><ShieldCheck :size="27" /></div><span class="sl-step">A LITTLE SPACE TO THINK</span><h2 class="h2">Your idea.<br />Your assumptions.</h2><p>This is a private what-if calculator. Reference prices are entered by you; they are not live quotes or execution prices.</p><dl class="sl-aside-rows"><div><dt>Asset</dt><dd>{{ brand.asset }} / USD</dd></div><div><dt>Network</dt><dd>{{ brand.network }} - preview</dd></div><div><dt>Wallet</dt><dd>Not required</dd></div><div><dt>Storage</dt><dd>This browser only</dd></div></dl><p class="sl-aside-foot">Fees, slippage and taxes are excluded. No transaction is created.</p></aside>
    </div>

    <div v-else-if="step === 'review'" class="panel sl-card sl-review">
      <div class="panel-body">
        <div class="sl-card-heading"><div><span class="sl-step">02 / REVIEW</span><h2 class="h2">Check the numbers.</h2></div><ShieldCheck :size="26" /></div><p class="sl-muted">Calculated from your assumptions.</p>
        <dl class="sl-review-rows"><div><dt>Reference amount</dt><dd>{{ Number(record.amount).toLocaleString() }} {{ brand.asset }}</dd></div><div><dt>Reference price</dt><dd>{{ currency(result.basePrice, 4) }}</dd></div><div><dt>Reference value</dt><dd>{{ currency(result.baselineValue, 4) }}</dd></div><div><dt>Scenario move</dt><dd :class="record.movePct >= 0 ? 'sl-positive' : 'sl-negative'">{{ percent(record.movePct) }}</dd></div><div><dt>Scenario value</dt><dd>{{ currency(result.scenarioValue, 4) }}</dd></div></dl>
        <div class="sl-delta"><span>Change in reference value</span><strong :class="result.deltaUsd >= 0 ? 'sl-positive' : 'sl-negative'">{{ result.deltaUsd >= 0 ? '+' : '' }}{{ currency(result.deltaUsd, 4) }}</strong></div>
        <div class="sl-actions"><button class="btn btn-primary" @click="save">Save to journal <Check :size="17" /></button><button class="btn btn-quiet" @click="edit">Edit inputs</button></div>
      </div>
    </div>

    <div v-else class="panel sl-card sl-journal">
      <div class="panel-body">
        <div class="sl-card-heading"><div><span class="sl-step">03 / JOURNAL</span><h2 class="h2">Your local record.</h2></div><span class="sl-badge" :class="{ 'sl-unsaved': !isSaved }"><Check v-if="isSaved" :size="13" />{{ isSaved ? 'SAVED' : 'NOT SAVED' }}</span></div>
        <div class="sl-stage-track" aria-label="Journal review stages"><button v-for="(label, i) in stages" :key="label" :class="{ current: record.stage === i, done: record.stage > i }" :aria-current="record.stage === i ? 'step' : undefined" @click="changeStage(i)"><span class="sl-stage-number"><Check v-if="record.stage > i" :size="13" /><template v-else>{{ i + 1 }}</template></span><span>{{ label }}</span><ChevronRight v-if="i < stages.length - 1" class="sl-stage-arrow" :size="14" /></button></div>
        <dl class="sl-summary"><div><dt>Amount</dt><dd>{{ Number(record.amount).toLocaleString() }} {{ brand.asset }}</dd></div><div><dt>Reference price</dt><dd>{{ currency(record.basePrice, 4) }}</dd></div><div><dt>Move</dt><dd :class="record.movePct >= 0 ? 'sl-positive' : 'sl-negative'">{{ percent(record.movePct) }}</dd></div><div><dt>Scenario value</dt><dd>{{ currency(result.scenarioValue, 4) }}</dd></div></dl>
        <blockquote v-if="record.note">{{ record.note }}</blockquote><p class="sl-muted">Created {{ new Date(record.createdAt).toLocaleString() }}. {{ isSaved ? 'Stored only in this browser.' : 'Export a copy to keep this record.' }}</p>
        <div class="sl-actions"><button class="btn btn-primary" @click="downloadJournal">Export JSON <ArrowDownToLine :size="16" /></button><button v-if="!isSaved" class="btn btn-quiet" @click="persist">Retry saving</button><button class="btn btn-quiet" @click="edit">Edit</button><button class="sl-delete" @click="removeJournal">Delete local record</button></div>
      </div>
    </div>
  </section>
</template>

<style scoped>
/* ------------------------------------------------------------------ sheet */
.scenario-lab { padding-top: 34px; }
.sl-header {
  display: grid; gap: 10px;
  padding-bottom: 20px;
  border-bottom: var(--rule-w) solid var(--ink);
}
.sl-header .sl-eyebrow { margin-bottom: 0; }
.sl-header .h1 { font-size: clamp(28px, 3.6vw, 46px); }
.sl-header .h1 span { color: var(--flare); }
.scenario-lab .notice { margin-top: 16px; }

/* -------------------------------------------------------- two-column rail */
.sl-layout {
  display: grid;
  grid-template-columns: minmax(0, 1.45fr) minmax(260px, 0.55fr);
  border-bottom: var(--hair-w) solid var(--hair);
}
.sl-form { padding: 22px 24px 26px 0; border-right: var(--hair-w) solid var(--hair); }
.sl-aside { padding: 22px 0 26px 24px; display: grid; gap: 12px; align-content: start; }

.sl-card-heading {
  display: flex; align-items: flex-start; justify-content: space-between; gap: 16px;
  padding-bottom: 12px;
  margin-bottom: 16px;
  border-bottom: var(--rule-w) solid var(--ink);
}
.sl-card-heading h2 { margin: 7px 0 0; }
.sl-step {
  display: block;
  color: var(--flare);
  font: 700 10px/1.3 var(--mono); letter-spacing: .16em; text-transform: uppercase;
}

/* ------------------------------------------------------------------ form */
.sl-field-pair { display: grid; grid-template-columns: repeat(2, minmax(0, 1fr)); gap: 0 14px; }
.formrow { display: grid; gap: 6px; margin-bottom: 14px; }
.formrow .flabel span { color: var(--faint); font-weight: 400; }
.formrow output { color: var(--ink); font-weight: 700; }
.sl-error { margin: 0; color: var(--alert); font: 11px/1.5 var(--mono); }
.sl-char-count { margin: 0; color: var(--faint); font: 10px/1.5 var(--mono); text-align: right; }
.sl-muted { margin: 0 0 14px; color: var(--muted); font: 11px/1.6 var(--mono); }

/* ------------------------------------------------------------------ aside */
.sl-aside-icon {
  display: grid; place-items: center;
  width: 46px; height: 46px;
  border: var(--rule-w) solid var(--ink);
  background: var(--sunken);
  color: var(--ink);
}
.sl-aside .h2 { font-size: clamp(20px, 2vw, 26px); }
.sl-aside p { margin: 0; color: var(--fg-2); font-size: 13px; line-height: 1.6; }
.sl-aside-rows { margin: 4px 0 0; border-top: var(--rule-w) solid var(--ink); }
.sl-aside-rows > div {
  display: flex; align-items: baseline; justify-content: space-between; gap: 10px;
  padding: 9px 0;
  border-bottom: var(--hair-w) solid var(--hair);
}
.sl-aside-rows dt { color: var(--muted); font: 10px/1.4 var(--mono); letter-spacing: .12em; text-transform: uppercase; }
.sl-aside-rows dd { margin: 0; font: 600 13px/1.4 var(--mono); font-variant-numeric: tabular-nums; }
.sl-aside-foot {
  margin: 0;
  padding-top: 12px;
  border-top: var(--hair-w) solid var(--hair);
}
.sl-aside .sl-aside-foot {
  color: var(--faint);
  font: 10px/1.6 var(--mono);
}

/* ---------------------------------------------------------- record sheets */
.sl-review, .sl-journal { margin-top: 20px; }
.sl-review-rows, .sl-summary { margin: 0 0 16px; border-top: var(--rule-w) solid var(--ink); }
.sl-review-rows > div, .sl-summary > div {
  display: flex; align-items: baseline; justify-content: space-between; gap: 12px;
  padding: 10px 0;
  border-bottom: var(--hair-w) solid var(--hair);
}
.sl-review-rows dt, .sl-summary dt {
  color: var(--muted);
  font: 10px/1.4 var(--mono); letter-spacing: .12em; text-transform: uppercase;
}
.sl-review-rows dd, .sl-summary dd {
  margin: 0;
  font: 600 14px/1.4 var(--mono); font-variant-numeric: tabular-nums;
}
.sl-positive { color: var(--ledger); }
.sl-negative { color: var(--alert); }
.sl-delta {
  display: flex; align-items: baseline; justify-content: space-between; gap: 14px;
  margin-bottom: 16px;
  padding: 14px 16px;
  border: var(--rule-w) solid var(--ink);
  background: var(--sunken);
}
.sl-delta span { color: var(--muted); font: 700 10px/1.3 var(--mono); letter-spacing: .12em; text-transform: uppercase; }
.sl-delta strong { font: 900 24px/1 var(--display); letter-spacing: -.01em; }

/* ------------------------------------------------------------ stage strip */
.sl-stage-track {
  display: grid;
  grid-template-columns: repeat(4, minmax(0, 1fr));
  margin-bottom: 18px;
  border: var(--rule-w) solid var(--ink);
  background: var(--card);
}
.sl-stage-track button {
  display: flex; align-items: center; gap: 8px;
  min-height: 46px;
  padding: 0 12px;
  border: 0;
  border-left: var(--hair-w) solid var(--hair);
  background: transparent;
  color: var(--muted);
  font: 700 10px/1.2 var(--mono); letter-spacing: .08em; text-transform: uppercase;
  text-align: left;
  cursor: pointer;
}
.sl-stage-track button:first-child { border-left: 0; }
.sl-stage-track button:hover { background: var(--paper); color: var(--ink); }
.sl-stage-track button.done { color: var(--ledger); }
.sl-stage-track button.current { background: var(--ink); color: var(--paper); }
.sl-stage-number {
  display: grid; place-items: center;
  flex: none;
  width: 22px; height: 22px;
  border: var(--hair-w) solid currentColor;
  font: 700 10px/1 var(--mono);
}
.sl-stage-arrow { margin-left: auto; opacity: .6; }

/* ---------------------------------------------------------------- journal */
.sl-badge {
  display: inline-flex; align-items: center; gap: 6px;
  flex: none;
  padding: 5px 9px;
  border: var(--hair-w) solid var(--ledger);
  color: var(--ledger);
  font: 700 10px/1 var(--mono); letter-spacing: .12em; text-transform: uppercase;
  white-space: nowrap;
}
.sl-unsaved { border-color: var(--flag); color: var(--flag); }
.sl-journal blockquote {
  margin: 0 0 16px;
  padding: 13px 16px;
  border: var(--hair-w) solid var(--hair);
  border-left: 4px solid var(--flare);
  background: var(--paper);
  color: var(--ink);
  font: 13px/1.65 var(--sans);
  white-space: pre-wrap;
}
.sl-actions { display: flex; flex-wrap: wrap; gap: 10px; margin-top: 4px; }
.sl-delete {
  border: 0;
  border-bottom: var(--hair-w) solid var(--alert);
  padding: 0 0 3px;
  background: none;
  color: var(--alert);
  font: 700 10px/1.6 var(--mono); letter-spacing: .12em; text-transform: uppercase;
  cursor: pointer;
}
.sl-delete:hover { color: var(--ink); border-bottom-color: var(--ink); }

/* ------------------------------------------------------------ responsive */
@media (max-width: 860px) {
  .sl-layout { grid-template-columns: minmax(0, 1fr); }
  .sl-form { padding: 20px 0; border-right: 0; border-bottom: var(--hair-w) solid var(--hair); }
  .sl-aside { padding: 20px 0 24px; }
}
@media (max-width: 640px) {
  .sl-field-pair { grid-template-columns: minmax(0, 1fr); }
  .sl-stage-track { grid-template-columns: repeat(2, minmax(0, 1fr)); }
  .sl-stage-track button:nth-child(3) { border-left: 0; }
  .sl-stage-track button:nth-child(n + 3) { border-top: var(--hair-w) solid var(--hair); }
  .sl-stage-arrow { display: none; }
}
</style>
