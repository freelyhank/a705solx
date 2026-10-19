<script setup>
import { computed, nextTick, onBeforeUnmount, onMounted, ref, shallowRef } from 'vue'
import { ExternalLink, ShieldCheck, Wallet, X } from 'lucide-vue-next'
import {
  connectSolanaWallet,
  disconnectSolanaWallet,
  getSolanaWalletSetupError,
  getSolanaWallets,
  initializeSolanaWallets,
  watchAvailableSolanaWallets,
  watchSolanaWallet,
  solanaConfig,
} from '../solana.js'

const emit = defineEmits(['connected', 'disconnected'])
const wallets = shallowRef([])
const activeWallet = shallowRef(null)
const address = ref('')
const modalOpen = ref(false)
const discovering = ref(false)
const pendingWallet = ref('')
const errorMessage = ref('')
const setupMessage = ref('')
const dialogPanel = ref(null)
let stopWalletDiscovery = () => {}
let stopWalletEvents = () => {}
let returnFocus = null

const shortAddress = computed(() => address.value ? address.value.slice(0, 5) + '...' + address.value.slice(-5) : '')
const buttonLabel = computed(() => address.value ? shortAddress.value : 'Connect wallet')

function showModal() {
  returnFocus = document.activeElement
  modalOpen.value = true
  errorMessage.value = ''
  setupMessage.value = ''
  wallets.value = getSolanaWallets()
  discovering.value = true
  void nextTick(() => dialogPanel.value?.querySelector('.wallet-dialog-close')?.focus())
  void initializeSolanaWallets()
    .then((available) => {
      wallets.value = available
      setupMessage.value = getSolanaWalletSetupError()
    })
    .catch((error) => {
      setupMessage.value = error?.message || 'Wallet discovery could not start.'
    })
    .finally(() => { discovering.value = false })
}

function closeModal() {
  modalOpen.value = false
  void nextTick(() => returnFocus?.focus?.())
}

function onDialogKeydown(event) {
  if (event.key === 'Escape') {
    event.preventDefault()
    closeModal()
    return
  }
  if (event.key !== 'Tab' || !dialogPanel.value) return
  const focusable = [...dialogPanel.value.querySelectorAll('button:not([disabled]), a[href], [tabindex]:not([tabindex="-1"])')]
  if (!focusable.length) return
  const first = focusable[0]
  const last = focusable[focusable.length - 1]
  if (event.shiftKey && document.activeElement === first) {
    event.preventDefault()
    last.focus()
  } else if (!event.shiftKey && document.activeElement === last) {
    event.preventDefault()
    first.focus()
  }
}

function syncAddress(nextAddress) {
  const value = typeof nextAddress === 'string' ? nextAddress : nextAddress?.address || ''
  address.value = value
  if (!value) {
    activeWallet.value = null
    stopWalletEvents()
    stopWalletEvents = () => {}
    emit('disconnected')
  } else {
    emit('connected', value, activeWallet.value?.name || '')
  }
}

async function connect(wallet) {
  pendingWallet.value = wallet.name
  errorMessage.value = ''
  try {
    const result = await connectSolanaWallet(wallet)
    stopWalletEvents()
    activeWallet.value = result.wallet
    syncAddress(result.address)
    stopWalletEvents = watchSolanaWallet((event) => {
      const next = event?.accounts?.[0]?.address || event?.address || event?.publicKey?.toString?.() || event?.toString?.() || ''
      syncAddress(next)
    }, result.wallet)
  } catch (error) {
    errorMessage.value = error?.message || `Could not connect to ${wallet.name}.`
  } finally {
    pendingWallet.value = ''
  }
}

async function disconnect() {
  pendingWallet.value = activeWallet.value?.name || 'wallet'
  errorMessage.value = ''
  try {
    await disconnectSolanaWallet(activeWallet.value)
    syncAddress('')
    closeModal()
  } catch (error) {
    errorMessage.value = error?.message || 'Wallet could not disconnect.'
  } finally {
    pendingWallet.value = ''
  }
}

onMounted(() => {
  stopWalletDiscovery = watchAvailableSolanaWallets((available) => { wallets.value = available })
})

onBeforeUnmount(() => {
  stopWalletDiscovery()
  stopWalletEvents()
})
</script>

<template>
  <div class="solana-wallet-control">
    <button
      type="button"
      class="btn wallet-button"
      :class="{ connected: address }"
      :aria-haspopup="'dialog'"
      :aria-expanded="modalOpen"
      :disabled="Boolean(pendingWallet)"
      @click="showModal"
    >
      <Wallet :size="16" />
      {{ pendingWallet ? 'Connecting…' : buttonLabel }}
    </button>

    <Teleport to="body">
      <div v-if="modalOpen" class="solana-wallet-backdrop" @click.self="closeModal" @keydown.esc.stop="closeModal">
        <section ref="dialogPanel" class="solana-wallet-dialog" role="dialog" aria-modal="true" aria-labelledby="wallet-dialog-title" tabindex="-1" @keydown.stop="onDialogKeydown">
          <header class="wallet-dialog-header">
            <div>
              <span class="eyebrow">SOLANA / {{ solanaConfig.cluster.toUpperCase() }}</span>
              <h2 id="wallet-dialog-title" class="h2">{{ address ? 'Wallet connected' : 'Choose a wallet' }}</h2>
            </div>
            <button class="wallet-dialog-close" type="button" aria-label="Close wallet dialog" @click="closeModal"><X :size="18" /></button>
          </header>

          <div v-if="address" class="wallet-account-card">
            <span class="wallet-account-name"><Wallet :size="16" /> {{ activeWallet?.name || 'Solana wallet' }}</span>
            <code>{{ address }}</code>
            <button class="btn btn-quiet btn-full wallet-disconnect" type="button" :disabled="Boolean(pendingWallet)" @click="disconnect">
              {{ pendingWallet ? 'Disconnecting…' : 'Disconnect wallet' }}
            </button>
          </div>

          <template v-else>
            <p class="wallet-dialog-copy">Connect a Solana account to display its public address. This site does not ask for transaction signatures.</p>
            <p class="notice notice-ok wallet-network-note"><ShieldCheck :size="15" /> Use a Solana wallet with an SPL account. The CA preview never asks for a seed phrase.</p>

            <p v-if="discovering" class="notice wallet-discovery" role="status">Looking for MetaMask, Phantom, Solflare and other Solana wallets…</p>
            <div v-else-if="wallets.length" class="wallet-options" aria-label="Available Solana wallets">
              <button v-for="wallet in wallets" :key="wallet.name" class="wallet-option" type="button" :disabled="Boolean(pendingWallet)" @click="connect(wallet)">
                <img v-if="wallet.icon" class="wallet-option-icon" :src="wallet.icon" alt="" />
                <span v-else class="wallet-option-icon wallet-option-placeholder"><Wallet :size="18" /></span>
                <span class="wallet-option-name">{{ wallet.name }}</span>
                <span v-if="pendingWallet === wallet.name" class="wallet-option-state">Connecting…</span>
                <ExternalLink v-else :size="15" class="wallet-option-arrow" />
              </button>
            </div>
            <div v-else class="wallet-empty-state">
              <p>No compatible Solana wallet was detected.</p>
              <p class="wallet-empty-help">Install a wallet, enable its Solana account, then reopen this panel.</p>
              <div class="wallet-install-links">
                <a href="https://metamask.io/download/" target="_blank" rel="noreferrer">MetaMask <ExternalLink :size="12" /></a>
                <a href="https://phantom.app/download" target="_blank" rel="noreferrer">Phantom <ExternalLink :size="12" /></a>
                <a href="https://solflare.com/download" target="_blank" rel="noreferrer">Solflare <ExternalLink :size="12" /></a>
              </div>
            </div>
          </template>

          <p v-if="errorMessage" class="notice notice-alert wallet-dialog-error" role="alert">{{ errorMessage }}</p>
          <p v-if="setupMessage" class="notice notice-flag wallet-dialog-setup" role="status">MetaMask connector: {{ setupMessage }}</p>
          <p class="wallet-dialog-footnote">Only the selected wallet’s public Solana address is read.</p>
        </section>
      </div>
    </Teleport>
  </div>
</template>

<style scoped>
/* ------------------------------------------------------------ connect control */
.solana-wallet-control { min-width: 0; }
.solana-wallet-control .wallet-button {
  width: 100%;
  border-radius: 0;
  box-shadow: var(--lift-1);
}
.solana-wallet-control .wallet-button:hover { background: var(--paper); }
.solana-wallet-control .wallet-button.connected {
  background: var(--ink);
  border-color: var(--ink);
  color: var(--paper);
  box-shadow: none;
}
.solana-wallet-control .wallet-button.connected:hover {
  background: var(--flare);
  border-color: var(--flare);
  color: #fff;
}

/* ----------------------------------------------------------------- scrim */
.solana-wallet-backdrop {
  position: fixed; inset: 0; z-index: 60;
  display: grid; place-items: center;
  padding: 20px;
  background: rgba(13, 13, 16, 0.78);
}

/* ------------------------------------------------------------- ink sheet */
.solana-wallet-dialog {
  width: min(100%, 470px);
  max-height: min(720px, calc(100vh - 40px));
  overflow: auto;
  padding: 18px;
  border: var(--rule-w) solid var(--ink);
  background: var(--card);
  color: var(--fg);
  box-shadow: var(--lift-2);
}
.solana-wallet-dialog:focus { outline: 0; }
.wallet-dialog-header {
  display: flex; align-items: flex-start; justify-content: space-between; gap: 16px;
  margin: -18px -18px 18px;
  padding: 15px 18px 16px;
  border-bottom: var(--rule-w) solid var(--ink);
  background: var(--ink);
  color: var(--on-ink);
}
.wallet-dialog-header .h2 { margin: 8px 0 0; font-size: 22px; color: var(--on-ink); }
.eyebrow {
  display: block;
  color: var(--flare);
  font: 700 10px/1 var(--mono); letter-spacing: .16em; text-transform: uppercase;
}
.wallet-dialog-close {
  display: grid; place-items: center;
  flex: none;
  width: 34px; height: 34px;
  border: var(--hair-w) solid #3a3a44;
  background: transparent;
  color: var(--on-ink);
  cursor: pointer;
}
.wallet-dialog-close:hover { background: var(--flare); border-color: var(--flare); color: #fff; }

.wallet-dialog-copy { margin: 0 0 12px; color: var(--fg-2); font-size: 14px; line-height: 1.6; }
.wallet-network-note { margin: 0 0 16px; }

/* ---------------------------------------------------------- wallet list */
.wallet-options { display: grid; gap: 8px; margin-bottom: 16px; }
.wallet-option {
  display: flex; align-items: center; gap: 12px;
  min-height: 54px;
  padding: 8px 12px;
  border: var(--rule-w) solid var(--ink);
  background: var(--card);
  color: var(--ink);
  text-align: left;
  cursor: pointer;
}
.wallet-option:hover:not(:disabled) { background: var(--paper); }
.wallet-option:disabled { opacity: .5; cursor: not-allowed; }
.wallet-option-icon {
  display: grid; place-items: center;
  flex: none;
  width: 32px; height: 32px;
  border: var(--hair-w) solid var(--ink);
  background: var(--sunken);
}
img.wallet-option-icon { object-fit: contain; padding: 4px; }
.wallet-option-placeholder { color: var(--muted); }
.wallet-option-name { flex: 1 1 auto; min-width: 0; font: 600 13px/1.3 var(--mono); }
.wallet-option-state { color: var(--muted); font: 700 10px/1 var(--mono); letter-spacing: .1em; text-transform: uppercase; }
.wallet-option-arrow { flex: none; color: var(--muted); }

/* -------------------------------------------------------- account state */
.wallet-account-card {
  display: grid; gap: 10px;
  margin-bottom: 16px;
  padding: 14px;
  border: var(--rule-w) solid var(--ink);
  background: var(--sunken);
}
.wallet-account-name { display: inline-flex; align-items: center; gap: 8px; font: 700 11px/1 var(--mono); letter-spacing: .1em; text-transform: uppercase; }
.wallet-account-card code {
  display: block;
  padding: 9px 10px;
  border: var(--hair-w) solid var(--hair);
  background: var(--card);
  font: 12px/1.5 var(--mono);
  word-break: break-all;
}
.wallet-disconnect { justify-self: start; }

/* --------------------------------------------------------- empty + links */
.wallet-empty-state {
  display: grid; gap: 8px;
  margin-bottom: 16px;
  padding: 14px;
  border: var(--rule-w) solid var(--ink);
  background: var(--paper);
}
.wallet-empty-state p { margin: 0; color: var(--fg-2); font-size: 13px; line-height: 1.55; }
.wallet-empty-help { color: var(--muted); font: 11px/1.6 var(--mono); }
.wallet-install-links { display: flex; flex-wrap: wrap; gap: 8px; margin-top: 4px; }
.wallet-install-links a {
  display: inline-flex; align-items: center; gap: 6px;
  padding: 9px 11px;
  border: var(--rule-w) solid var(--ink);
  background: var(--card);
  color: var(--ink);
  font: 700 10px/1 var(--mono); letter-spacing: .1em; text-transform: uppercase;
  text-decoration: none;
}
.wallet-install-links a:hover { background: var(--ink); color: var(--paper); }

/* -------------------------------------------------------------- messages */
.wallet-discovery { margin: 0 0 16px; }
.wallet-dialog-error { margin: 0 0 12px; }
.wallet-dialog-setup { margin: 0 0 12px; }
.wallet-dialog-footnote {
  margin: 16px 0 0;
  padding-top: 12px;
  border-top: var(--hair-w) solid var(--hair);
  color: var(--faint);
  font: 10px/1.6 var(--mono);
}

@media (max-width: 480px) {
  .solana-wallet-backdrop { padding: 0; place-items: stretch; }
  .solana-wallet-dialog {
    width: 100%;
    max-height: 100vh; max-height: 100dvh;
    border: 0;
    border-top: var(--rule-w) solid var(--ink);
  }
}
</style>
