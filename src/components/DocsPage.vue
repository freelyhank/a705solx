<script setup>
import { computed } from 'vue'
import { ArrowRight, ExternalLink } from 'lucide-vue-next'
import { concepts } from '../concepts.js'
import { brand } from '../brand.js'

const props = defineProps({
  path: { type: String, default: '/docs' },
})
const emit = defineEmits(['navigate'])

const sections = [
  ['mechanism', 'Mechanism'],
  ['concepts', 'Concepts'],
  ['settlement', 'Settlement'],
  ['oracle', 'Oracle'],
  ['fees', 'Fees'],
  ['listing', 'Listing'],
  ['risks', 'Risks'],
  ['token', 'Token'],
  ['documents', 'Journal'],
  ['terms', 'Terms'],
  ['privacy', 'Privacy'],
]

const steps = [
  ['01', 'Choose a market unit', 'Select an SPL asset from the watchlist. Prices in this build are illustrative examples.'],
  ['02', 'Set a cap', 'The model pairs an Income position with an Upside position around cap price K.'],
  ['03', 'Price the scenario', 'Auction and premium mechanics are part of the reference design; no auction runs here.'],
  ['04', 'Review settlement', 'Change the assumed settlement price to see how the modeled value splits at K.'],
  ['05', 'Recompose the pair', 'The design describes matching positions recombining into one market unit. This action is not implemented.'],
]

const legalPage = computed(() => ['/risk', '/terms', '/privacy'].includes(props.path))
const legalTitle = computed(() => ({ '/risk': 'Risk notice.', '/terms': 'Terms of this demo.', '/privacy': 'Privacy notice.' }[props.path] || ''))
const legalBody = computed(() => ({
  '/risk': [
    'This local interface is an educational scenario lab. It does not create, sell, or settle a financial product.',
    'Income positions retain downside exposure to the underlying SPL market unit. Upside can expire without a payoff and its maximum modeled loss is the premium paid. Verify the underlying, oracle, liquidity, and legal status independently before relying on any model output.',
  ],
  '/terms': [
    'This website is a local product demonstration. Prices, markets, wallets, and contract references are illustrative unless a connected production integration says otherwise.',
    `Use of this demo does not create an account, custody relationship, order, or agreement with ${brand.name}. Contract references and the scenario lab are read-only in this build.`,
  ],
  '/privacy': [
    'The demo keeps journal state in your browser. You can read the pages without connecting a wallet, and this interface does not send a trade order.',
    'External links, such as the chart provider or X, are governed by their own policies. Do not enter secrets, seed phrases, or private keys into this local demo.',
  ],
}[props.path] || []))

function go(to) { emit('navigate', to) }
</script>

<template>
  <section v-if="legalPage" class="docs-page legal-page sheet-body">
    <header class="docs-hero">
      <p class="kick">Local interface preview</p>
      <h1 class="h1" tabindex="-1">{{ legalTitle }}</h1>
      <p class="lede">This page describes how this local product demonstration should be read. It is not a production protocol policy, legal agreement, or investment recommendation.</p>
    </header>
    <div class="legal-copy">
      <p v-for="paragraph in legalBody" :key="paragraph">{{ paragraph }}</p>
      <div class="legal-actions">
        <a class="btn btn-primary" href="/docs" @click.prevent="go('/docs')">Read how it works <ArrowRight :size="15" /></a>
        <a class="btn btn-quiet" href="/" @click.prevent="go('/')">Back to markets <ArrowRight :size="14" /></a>
      </div>
    </div>
  </section>

  <section v-else class="docs-page sheet-body">
    <header class="docs-hero">
      <p class="kick">{{ brand.network }} / Model notes</p>
      <h1 class="h1" tabindex="-1">How {{ brand.name }} works.</h1>
      <p class="lede">This preview models an illustrative split around a cap price. It does not custody SPL assets, run an auction, connect an oracle, or settle transactions.</p>
      <p class="foot-note">All displayed market values are examples. No token or Solana program has been issued by this preview.</p>
    </header>

    <div class="docs-layout">
      <nav class="docs-toc" aria-label="On this page">
        <p class="toc-label">On this page</p>
        <a v-for="([id, label], index) in sections" :key="id" :href="`#${id}`"><b>{{ String(index + 1).padStart(2, '0') }}</b><span>{{ label }}</span></a>
      </nav>

      <div class="docs-content">
        <section id="mechanism" class="docs-section">
          <p class="kick">Mechanism</p>
          <ol class="mechanism-list">
            <li v-for="[number, title, body] in steps" :key="number">
              <span class="step-number">{{ number }}</span>
              <div><h2 class="h3">{{ title }}</h2><p>{{ body }}</p></div>
            </li>
          </ol>
          <div class="role-grid">
            <article class="role-card"><p class="role-label income">Income</p><h3 class="h3">Value up to K, plus any premium.</h3><p>Income still bears the underlying asset's downside. A premium does not protect the market unit from a price decline.</p></article>
            <article class="role-card"><p class="role-label upside">Upside</p><h3 class="h3">Value above K.</h3><p>The modeled payoff is max(0, settlement price - K). A buyer's maximum modeled loss is the hypothetical premium paid.</p></article>
          </div>
        </section>

        <section id="concepts" class="docs-section">
          <div class="section-heading">
            <p class="kick">Concepts</p>
            <h2 class="h2">Three parts of the model.</h2>
            <p>These illustrations describe a product concept, not deployed Solana programs.</p>
          </div>
          <ul class="concept-grid">
            <li v-for="(concept, index) in concepts" :id="`concept-${concept.id}`" :key="concept.id" class="concept-card">
              <figure class="concept-figure">
                <img :src="concept.image" :alt="concept.title" width="1200" height="750" loading="lazy" />
                <figcaption>
                  <span class="concept-index">{{ String(index + 1).padStart(2, '0') }}</span>
                  <h3 class="h3">{{ concept.title }}</h3>
                  <p>{{ concept.description }}</p>
                </figcaption>
              </figure>
            </li>
          </ul>
        </section>

        <section id="settlement" class="docs-section split-section">
          <div>
            <p class="kick">Settlement</p>
            <h2 class="h2">A defined cap sets the split.</h2>
            <p>For starting price P0 and cap rate c, K = P0 x (1 + c). At assumed settlement price S, Upside receives max(0, S - K). Income receives value up to K plus any hypothetical auction premium.</p>
          </div>
          <div class="formula-card">
            <p class="formula-label">Illustrative scenario</p>
            <strong>P0 $250 / 5% cap / K $262.50</strong>
            <dl class="cells formula-cells" style="--cols: 3"><div><dt class="cell-label">Settlement $200</dt><dd class="cell-value">Upside $0</dd></div><div><dt class="cell-label">Settlement $250</dt><dd class="cell-value">Upside $0</dd></div><div><dt class="cell-label">Settlement $300</dt><dd class="cell-value">Upside $37.50</dd></div></dl>
          </div>
        </section>

        <section id="oracle" class="docs-section prose-section"><p class="kick">Oracle</p><h2 class="h2">No live price source is connected.</h2><p>A production series would need a named feed, market session, fallback behavior, and settlement window. This preview uses local assumptions and performs no oracle reads.</p></section>
        <section id="fees" class="docs-section prose-section"><p class="kick">Fees</p><h2 class="h2">No fee is charged by this demo.</h2><p>A 5% auction fee appeared in an earlier reference model. It is only a design assumption and is not active, binding, or a production term.</p></section>
        <section id="listing" class="docs-section prose-section"><p class="kick">Listing</p><h2 class="h2">Assets need a verified market path.</h2><p>A candidate SPL market unit would need a reliable price source and a USDC liquidity path. Listing, custody, jurisdiction, and program permissions require separate review before production.</p></section>
        <section id="risks" class="docs-section prose-section"><p class="kick">Risks</p><h2 class="h2">A premium does not remove risk.</h2><p>Income still bears underlying downside. Upside may expire without a payoff. Oracle, liquidity, smart contract, counterparty, market, and regulatory risks remain outside this interface.</p></section>
        <section id="token" class="docs-section prose-section"><p class="kick">Token</p><h2 class="h2">The proposed {{ brand.asset }} symbol is separate.</h2><p>Income and Upside describe modeled positions. {{ brand.asset }} is a proposed symbol only; it has not been issued, and no supply or utility is enabled in this preview.</p></section>
        <section id="documents" class="docs-section prose-section"><p class="kick">Journal</p><h2 class="h2">Keep assumptions in this browser.</h2><p>The scenario journal stores its record locally and can export it as JSON. It does not submit a transaction or publish the record on chain.</p><a class="text-link" href="/signal" @click.prevent="go('/signal')">Open the journal <ArrowRight :size="14" /></a></section>
        <section id="terms" class="docs-section prose-section"><p class="kick">Terms</p><h2 class="h2">Terms for this local preview.</h2><p>This page is a readable product notice for the interface. It does not replace production terms or create a contractual relationship.</p><a class="text-link" href="/terms" @click.prevent="go('/terms')">Read the demo terms <ExternalLink :size="13" /></a></section>
        <section id="privacy" class="docs-section prose-section"><p class="kick">Privacy</p><h2 class="h2">Your browser holds the journal state.</h2><p>No trade order or private key is requested by this documentation view. External links have their own privacy practices.</p><a class="text-link" href="/privacy" @click.prevent="go('/privacy')">Read the demo privacy notice <ExternalLink :size="13" /></a></section>

        <section id="overview" class="legacy-anchor" aria-label="Legacy overview anchor"></section>
        <section id="scenario" class="legacy-anchor" aria-label="Legacy scenario anchor"></section>
        <section id="journal" class="legacy-anchor" aria-label="Legacy journal anchor"></section>
        <section id="limits" class="legacy-anchor" aria-label="Legacy limits anchor"></section>
      </div>
    </div>
  </section>
</template>

<style scoped>
/* ------------------------------------------------------------------- hero */
.docs-page { padding-top: 34px; }
.docs-hero { display: grid; gap: 10px; padding-bottom: 24px; }
.docs-hero .h1 { font-size: clamp(28px, 3.6vw, 46px); }
.foot-note { margin: 0; color: var(--faint); font: 10px/1.6 var(--mono); }

/* ----------------------------------------------------- index + ruled sheet */
.docs-layout {
  display: grid;
  grid-template-columns: 176px minmax(0, 1fr);
  border-top: var(--rule-w) solid var(--ink);
}
.docs-toc {
  position: sticky; top: calc(var(--strip-h) + 18px);
  align-self: start;
  display: grid; align-content: start;
  padding: 20px 16px 24px 0;
  border-right: var(--hair-w) solid var(--hair);
}
.toc-label {
  margin: 0 0 10px;
  color: var(--muted);
  font: 700 10px/1.3 var(--mono); letter-spacing: .14em; text-transform: uppercase;
}
.docs-toc a {
  display: grid; grid-template-columns: 26px minmax(0, 1fr);
  align-items: baseline; gap: 4px;
  padding: 7px 0;
  border-bottom: var(--hair-w) solid var(--hair);
  color: var(--fg-2);
  font: 700 10px/1.4 var(--mono); letter-spacing: .1em; text-transform: uppercase;
  text-decoration: none;
}
.docs-toc a:last-child { border-bottom: 0; }
.docs-toc a b { color: var(--faint); font-weight: 400; }
.docs-toc a:hover { color: var(--ink); box-shadow: inset 3px 0 0 var(--flare); padding-left: 8px; }
.docs-toc a:hover b { color: var(--flare); }

.docs-content { min-width: 0; padding: 0 0 40px 26px; }
.docs-section { padding: 26px 0; scroll-margin-top: calc(var(--strip-h) + 16px); }
.docs-section + .docs-section { border-top: var(--rule-w) solid var(--ink); }
.docs-section h2 { margin: 12px 0 10px; }
.docs-section p { color: var(--fg-2); font-size: 14px; line-height: 1.65; }
.docs-section .kick { margin-bottom: 0; }

/* ------------------------------------------------------------- mechanism */
.mechanism-list { display: grid; margin: 20px 0 0; padding: 0; list-style: none; border-top: var(--hair-w) solid var(--hair); }
.mechanism-list li {
  display: grid; grid-template-columns: 34px minmax(0, 1fr);
  gap: 14px;
  padding: 14px 0;
  border-bottom: var(--hair-w) solid var(--hair);
}
.mechanism-list h2 { margin: 0 0 4px; }
.mechanism-list p { margin: 0; }
.step-number { color: var(--flare); font: 700 11px/1.5 var(--mono); letter-spacing: .1em; }
.role-grid { display: grid; grid-template-columns: repeat(2, minmax(0, 1fr)); gap: 14px; margin-top: 18px; }
.role-card {
  min-width: 0;
  padding: 16px;
  border: var(--rule-w) solid var(--ink);
  background: var(--card);
}
.role-label { margin: 0 0 8px; font: 700 10px/1.3 var(--mono); letter-spacing: .14em; text-transform: uppercase; }
.role-label.income { color: var(--ledger); }
.role-label.upside { color: var(--alert); }
.role-card h3 { margin: 0 0 6px; }
.role-card p { margin: 0; font-size: 13px; }

/* -------------------------------------------------------------- concepts */
.section-heading { margin-bottom: 18px; }
.section-heading h2 { margin: 12px 0 8px; }
.section-heading p { margin: 0; }
.concept-grid { display: grid; grid-template-columns: repeat(3, minmax(0, 1fr)); gap: 16px; margin: 0; padding: 0; list-style: none; }
.concept-card { min-width: 0; }
.concept-figure {
  display: grid;
  margin: 0;
  border: var(--rule-w) solid var(--ink);
  background: var(--card);
  box-shadow: var(--lift-1);
}
.concept-figure img {
  width: 100%; height: auto;
  aspect-ratio: 16 / 10;
  object-fit: contain;
  border-bottom: var(--rule-w) solid var(--ink);
  background: var(--card);
}
.concept-figure figcaption { padding: 13px 14px 15px; }
.concept-index {
  display: block;
  margin-bottom: 6px;
  color: var(--flare);
  font: 700 10px/1 var(--mono); letter-spacing: .16em;
}
.concept-figure h3 { margin: 0 0 6px; }
.concept-figure p { margin: 0; font-size: 12px; color: var(--muted); }

/* ------------------------------------------------------------ settlement */
.split-section { display: grid; grid-template-columns: minmax(0, 1.15fr) minmax(260px, 0.85fr); gap: 26px; align-items: start; }
.formula-card { border: var(--rule-w) solid var(--ink); background: var(--card); padding: 16px; }
.formula-label { margin: 0; color: var(--muted); font: 700 10px/1.3 var(--mono); letter-spacing: .14em; text-transform: uppercase; }
.formula-card strong { display: block; margin: 12px 0 14px; font: 600 14px/1.5 var(--mono); font-variant-numeric: tabular-nums; }
.formula-cells { margin: 0; grid-template-columns: minmax(0, 1fr); }
.formula-cells > div {
  display: flex; align-items: baseline; justify-content: space-between; gap: 12px;
  padding: 10px 12px;
  border-left: 0;
}
.formula-cells dd { margin: 0; }
.formula-cells .cell-value { margin-top: 0; font-size: 15px; }

/* -------------------------------------------------------------- prose */
.prose-section { max-width: 780px; }
.text-link {
  display: inline-flex; align-items: center; gap: 6px;
  margin-top: 10px;
  border-bottom: var(--hair-w) solid var(--signal);
  color: var(--signal);
  font: 700 10px/1.6 var(--mono); letter-spacing: .12em; text-transform: uppercase;
  text-decoration: none;
}
.text-link:hover { color: var(--flare); border-bottom-color: var(--flare); }
.legacy-anchor { height: 1px; padding: 0; border: 0; scroll-margin-top: calc(var(--strip-h) + 16px); }

/* ---------------------------------------------------------------- legal */
.legal-copy { max-width: 760px; padding-top: 24px; }
.legal-copy p { margin: 0 0 18px; color: var(--fg-2); font-size: 15px; line-height: 1.75; }
.legal-actions { display: flex; flex-wrap: wrap; align-items: center; gap: 12px; margin-top: 26px; }

/* ----------------------------------------------------------- responsive */
@media (max-width: 860px) {
  .docs-page { padding-top: 24px; }
  .docs-layout { grid-template-columns: minmax(0, 1fr); }
  .docs-toc {
    position: static;
    grid-template-columns: repeat(auto-fill, minmax(148px, 1fr));
    padding: 0;
    border-right: 0;
    border-bottom: var(--rule-w) solid var(--ink);
  }
  .toc-label { grid-column: 1 / -1; padding: 12px 0 10px; margin: 0; }
  .docs-toc a { padding: 10px 12px; border-bottom: 0; border-right: var(--hair-w) solid var(--hair); }
  .docs-toc a:hover { box-shadow: inset 3px 0 0 var(--flare); padding-left: 12px; }
  .docs-content { padding: 0 0 32px; }
  .concept-grid, .split-section, .role-grid { grid-template-columns: minmax(0, 1fr); }
}
@media (max-width: 520px) {
  .docs-section { padding: 22px 0; }
  .legal-actions { align-items: stretch; flex-direction: column; }
  .legal-actions .btn { width: 100%; }
}
</style>
