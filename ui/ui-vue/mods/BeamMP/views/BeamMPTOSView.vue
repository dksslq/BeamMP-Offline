<template>
  <section class="panel">
    <h1>Welcome to BeamMP — Offline Edition</h1>

    <div class="block intro">
      <p>
        This is a fully <strong>offline</strong> build of BeamMP: no account, no forum key,
        no Discord and no internet connection are required. Host a server on your LAN,
        share its IP address with friends, and join via <strong>Direct Connect</strong>.
      </p>
      <p class="muted">
        The upstream multiplayer rules still apply as a common-sense guideline: be respectful,
        no griefing, and respect server owners' rules.
      </p>
    </div>

    <div class="block">
      <h2>Your player name</h2>
      <p class="muted">
        Pick a display name other players will see. You can change it later from the
        Player screen (top bar) or the Player Identity page. Leave empty to join servers as a Guest.
      </p>
      <div class="input-shell">
        <input
          id="beammp-offline-name"
          v-model="playerName"
          v-bng-text-input
          type="text"
          maxlength="32"
          autocomplete="off"
          autocapitalize="none"
          spellcheck="false"
          placeholder="e.g. SpeedRacer"
          @keyup.enter="proceed"
        />
      </div>
    </div>

    <div class="block">
      <label><input v-model="tosAccepted" type="checkbox" /> I understand this is an offline community build and accept the guideline above</label>
    </div>

    <div class="actions">
      <BngButton :disabled="!canContinue" @click="proceed">{{ startLabel }}</BngButton>
      <BngButton accent="secondary" :disabled="!tosAccepted" @click="proceedAsGuest">Play as Guest</BngButton>
    </div>
  </section>
</template>

<script setup>
// === OFFLINE MODE (BeamMP-Offline) ===
// Upstream this screen forced acceptance of the BeamMP online Terms of Service
// and Rules (links to forum.beammp.com / docs.beammp.com). The offline edition
// replaces it with a local welcome screen that also collects the player name.
import { computed, ref, watch } from "vue"
import { useRouter } from "vue-router"
import { BngButton } from "@/common/components/base"
import { vBngTextInput } from "@/common/directives"
import { BEAMMP_SERVERS_ROUTE_NAME } from "../shared/constants.js"
import { useBeamMPState } from "../shared/beammpState.js"

const router = useRouter()
const { acceptTos, login, guestLogin, state } = useBeamMPState()
const tosAccepted = ref(false)
const playerName = ref(state.auth.value?.username || "")
const canContinue = computed(() => tosAccepted.value)
const startLabel = computed(() => (playerName.value.trim() ? "Start Playing" : "Continue"))

async function proceed() {
  if (!canContinue.value) return
  acceptTos()
  if (playerName.value.trim()) {
    await login(playerName.value.trim(), "")
  }
  router.push({ name: BEAMMP_SERVERS_ROUTE_NAME })
}

async function proceedAsGuest() {
  if (!tosAccepted.value) return
  acceptTos()
  await guestLogin()
  router.push({ name: BEAMMP_SERVERS_ROUTE_NAME })
}

// if the user is somehow already logged in with a name, prefill happened above;
// keep in sync if auth changes while typing (e.g. saved name restored late)
watch(() => state.auth.value?.username, value => {
  if (value && !playerName.value) playerName.value = value
})
</script>

<style scoped lang="scss">
.panel {
  display: flex;
  flex-direction: column;
  gap: 1rem;
}

.block {
  padding: 0.8rem;
  border-radius: var(--bng-corners-1);
  background: rgba(255, 255, 255, 0.04);
}

.intro {
  strong {
    color: var(--bng-orange-300);
  }
}

.muted {
  color: var(--bng-cool-gray-100);
}

.actions {
  display: flex;
  justify-content: flex-end;
  gap: 0.5rem;
}

.input-shell {
  display: flex;
  min-height: 2.7rem;
  align-items: stretch;
  overflow: hidden;
  border: 1px solid rgba(255, 255, 255, 0.22);
  border-radius: var(--bng-corners-1);
  background: rgba(7, 10, 14, 0.78);
  transition: border-color 120ms ease, box-shadow 120ms ease;

  &:hover {
    border-color: rgba(255, 255, 255, 0.42);
  }

  &:focus-within {
    border-color: var(--bng-orange-500);
    box-shadow: 0 0 0 0.13rem rgba(var(--bng-orange-500-rgb), 0.32);
  }

  input {
    flex: 1;
    min-width: 0;
    padding: 0.55rem 0.7rem;
    border: 0;
    outline: 0;
    color: var(--bng-off-white);
    background: transparent;
    font: inherit;
  }
}

p {
  color: var(--bng-cool-gray-100);
}

a {
  color: var(--bng-orange-300);
}
</style>
