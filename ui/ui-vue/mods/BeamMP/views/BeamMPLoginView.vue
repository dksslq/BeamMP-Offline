<template>
  <section class="login-layout">
    <article class="login-popup">
      <img :src="logoSrc" class="beammp-logo" alt="BeamMP" @error="onLogoError" />

      <p v-if="state.loginError.value && hasTriedToLogin" class="error">{{ state.loginError.value }}</p>

      <h2 class="login-title">Player Identity</h2>
      <p class="guest-copy">
        Offline edition — pick the name other players will see. No account or password needed.
      </p>

      <div class="input-group">
        <label for="beammp-login-username">Player name</label>
        <div class="input-shell">
          <input
            id="beammp-login-username"
            v-model="username"
            v-bng-text-input
            type="text"
            maxlength="32"
            autocomplete="off"
            autocapitalize="none"
            spellcheck="false"
            placeholder="e.g. SpeedRacer"
            @keyup.enter="submitLogin"
          />
        </div>
      </div>

      <div class="actions">
        <BngButton :disabled="!canSave" @click="submitLogin">Save Name</BngButton>
        <BngButton accent="secondary" @click="goBack">Back</BngButton>
      </div>

      <p class="current-name" v-if="state.auth.value?.username">
        Current name: <strong>{{ state.auth.value.username }}</strong>
      </p>
    </article>
  </section>
</template>

<script setup>
// === OFFLINE MODE (BeamMP-Offline) ===
// Upstream this screen authenticated against the BeamMP forum account system
// (username + password sent to auth.beammp.com via the launcher) and offered a
// registration link. The offline edition has no accounts: the player just picks
// a local display name which the launcher stores and offline servers accept.
// Since v1.0.3 the name is mandatory and the "Play as Guest" option is gone.
import { computed, ref } from "vue"
import { useRouter } from "vue-router"
import { BngButton } from "@/common/components/base"
import { vBngTextInput } from "@/common/directives"
import { BEAMMP_SERVERS_ROUTE_NAME } from "../shared/constants.js"
import { useBeamMPState } from "../shared/beammpState.js"

const router = useRouter()
const username = ref("")
const hasTriedToLogin = ref(false)
const LEGACY_LOGO_PATH = "ui/assets/BeamMP/beammp_new_cropped.png"
const LOGO_FALLBACK = "/ui/assets/BeamMP/icons/account-multiple.svg"
const logoSrc = ref(LEGACY_LOGO_PATH)
const { login, state } = useBeamMPState()
const canSave = computed(() => Boolean(username.value.trim()))

function onLogoError() {
  if (logoSrc.value !== LOGO_FALLBACK) {
    logoSrc.value = LOGO_FALLBACK
  }
}

async function submitLogin() {
  const saved = await login(username.value, "")
  hasTriedToLogin.value = true
  if (!saved) return
  username.value = ""
}

function goBack() {
  router.back()
}

function toServers() {
  router.replace({ name: BEAMMP_SERVERS_ROUTE_NAME })
}

// once a name is saved (the launcher confirms via the auth events) jump to the server list
import { watch } from "vue"
watch(() => state.loggedIn.value, value => {
  if (value) toServers()
})
</script>

<style scoped lang="scss">
.login-layout {
  min-height: min(40rem, 70vh);
  width: 100%;
  display: grid;
  place-items: center;
}

.login-popup {
  width: min(36rem, 96%);
  padding: 1rem;
  border-radius: var(--bng-corners-3);
  border: 2px solid rgba(var(--bng-cool-gray-600-rgb), 0.95);
  background: linear-gradient(145deg, rgba(29, 29, 29, 0.92), rgba(20, 20, 20, 0.9));
  display: flex;
  flex-direction: column;
  gap: 0.7rem;
  color: var(--bng-off-white);
}

.beammp-logo {
  width: auto;
  max-width: 14rem;
  height: 4rem;
  object-fit: contain;
  margin: 0 auto 0.25rem;
}

.login-title {
  margin: 0;
  text-align: center;
  font-weight: 700;
}

.input-group {
  display: flex;
  flex-direction: column;
  gap: 0.35rem;

  label {
    font-size: 0.85rem;
    font-weight: 600;
    color: var(--bng-cool-gray-100);
  }
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

.guest-copy {
  color: var(--bng-cool-gray-100);
  margin: 0;
  text-align: center;
}

.current-name {
  margin: 0;
  text-align: center;
  color: var(--bng-cool-gray-100);
}

.actions {
  justify-content: center;
  display: flex;
  gap: 0.5rem;
  flex-wrap: wrap;

  :deep(button),
  :deep(.bng-button) {
    margin: 0;
  }
}

.error {
  margin: 0;
  text-align: center;
  color: var(--bng-add-red-500);
}

@media (max-width: 680px) {
  .actions {
    > * {
      flex: 1 1 100%;
    }
  }
}
</style>
