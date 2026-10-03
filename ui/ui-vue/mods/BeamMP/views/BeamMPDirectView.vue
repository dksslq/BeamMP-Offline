<template>
  <section class="direct-wrap">
    <header class="direct-header">
      <h2>{{ $tt("ui.common.beammp.direct_connect") }}</h2>
      <p>Connect to a BeamMP server using its address and port.</p>
      <p class="identity-line">
        Playing as <strong>{{ currentName || "Guest" }}</strong> — change your name on the Player Identity page.
      </p>
    </header>

    <div class="direct-card">
      <div class="fields">
        <label class="field field-address">
          <span>{{ $tt("ui.beammp.serverBrowser.serverIp") }}</span>
          <div class="input-shell">
                        <span class="field-prefix">IP</span>
            <BngInput
              v-model.trim="ip"
                          class="direct-input"
              type="text"
              placeholder="127.0.0.1"
              :show-external-button="false"
            />
          </div>
        </label>

        <label class="field field-port">
          <span>{{ $tt("ui.beammp.serverBrowser.serverPort") }}</span>
          <div class="input-shell">
                        <span class="field-prefix">:</span>
            <BngInput
              v-model.trim="port"
              type="text"
              inputmode="numeric"
              autocomplete="off"
              placeholder="30814"
              :show-external-button="false"
            />
          </div>
        </label>
      </div>

      <p v-if="connectError" class="connect-error">{{ connectError }}</p>

      <div class="actions">
        <BngButton accent="secondary" @click="pasteFromClipboard">{{ $tt("ui.common.beammp.pasteFromClipboard") }}</BngButton>
        <BngButton :disabled="Boolean(connectError)" @click="connect">{{ $tt("ui.common.beammp.connect") }}</BngButton>
        <BngButton accent="secondary" :disabled="Boolean(connectError)" @click="favorite">{{ $tt("ui.beammp.serverBrowser.saveAsFavorite") }}</BngButton>
      </div>
    </div>
  </section>
</template>

<script setup>
// === OFFLINE MODE (BeamMP-Offline) ===
// Polish on top of upstream's direct connect: remembers the last used
// address, validates the input before connecting, and shows the local
// player identity. No account is involved.
import { computed, onMounted, ref } from "vue"
import { BngButton, BngDropdown, BngInput, ACCENTS } from "@/common/components/base"
import { useBeamMPState } from "../shared/beammpState.js"

const LAST_DIRECT_KEY = "beammpOfflineLastDirect"
const ip = ref("")
const port = ref("")
const { addFavorite, connectToServer, directConnectFromClipboard, state } = useBeamMPState()

const currentName = computed(() => String(state.auth.value?.username || ""))

const addressPattern = /^[A-Za-z0-9](?:[A-Za-z0-9._-]{0,251}[A-Za-z0-9])?$/
const portValue = computed(() => Number(port.value))
const connectError = computed(() => {
  const value = String(ip.value || "").trim()
  if (!value) return "" // empty means 127.0.0.1 (local server)
  if (!addressPattern.test(value)) return "Invalid address: use an IPv4 address or a hostname (e.g. 192.168.1.20)"
  const rawPort = String(port.value || "").trim()
  if (rawPort === "") return ""
  if (!/^[0-9]+$/.test(rawPort) || portValue.value < 1 || portValue.value > 65535) {
    return "Invalid port: use a number between 1 and 65535 (leave empty for 30814)"
  }
  return ""
})

function restoreLastDirect() {
  try {
    const saved = JSON.parse(localStorage.getItem(LAST_DIRECT_KEY) || "null")
    if (saved && typeof saved === "object") {
      if (typeof saved.ip === "string") ip.value = saved.ip
      if (typeof saved.port === "string" || typeof saved.port === "number") port.value = String(saved.port)
    }
  } catch {
    // ignore malformed saved data
  }
}

function rememberLastDirect(nextIp, nextPort) {
  try {
    localStorage.setItem(LAST_DIRECT_KEY, JSON.stringify({ ip: nextIp, port: nextPort }))
  } catch {
    // storage may be unavailable - non-critical
  }
}

onMounted(restoreLastDirect)

async function pasteFromClipboard() {
  const text = String(await directConnectFromClipboard() || "")
  if (!text.includes(".")) return
  const [nextIp, nextPort] = text.split(":")
  ip.value = nextIp || ip.value
  if (nextPort) port.value = nextPort
}

async function connect() {
  if (connectError.value) return
  const useIp = ip.value || "127.0.0.1"
  const usePort = port.value || "30814"
  rememberLastDirect(useIp, usePort)
  await connectToServer(useIp, usePort)
}

async function favorite() {
  if (connectError.value) return
  const ipFav = ip.value || "127.0.0.1"
  const portFav = port.value || "30814"
  rememberLastDirect(ipFav, portFav)
  bngVue.toastr.success(`Adding ${ipFav}:${portFav} to favorites`, "BeamMP")
  addFavorite({
    ip: ipFav,
    port: portFav,
    sname: new Date().toLocaleString(),
    strippedName: new Date().toLocaleString(),
    custom: true,
    tags: "",
    map: "",
    location: "--",
  })
}
</script>

<style scoped lang="scss">
.direct-wrap {
  width: min(48rem, 100%);
  display: flex;
  flex-direction: column;
  gap: 0.9rem;
  padding: 0.35rem;
}

.direct-header {
  h2 {
    margin: 0;
    font-size: 1.3rem;
  }

  p {
    margin: 0.25rem 0 0;
    color: var(--bng-cool-gray-300);
  }

  .identity-line {
    font-size: 0.85rem;

    strong {
      color: var(--bng-orange-300);
    }
  }
}

.connect-error {
  margin: 0;
  padding: 0.45rem 0.6rem;
  border: 1px solid rgba(var(--bng-add-red-500-rgb), 0.65);
  border-radius: var(--bng-corners-1);
  background: rgba(var(--bng-add-red-500-rgb), 0.14);
  color: var(--bng-add-red-300);
  font-size: 0.85rem;
}

.direct-card {
  display: flex;
  flex-direction: column;
  gap: 1rem;
  padding: 1rem;
  border: 1px solid rgba(255, 255, 255, 0.14);
  border-radius: var(--bng-corners-2);
  background:
    linear-gradient(135deg, rgba(27, 31, 38, 0.94), rgba(13, 16, 21, 0.9)),
    rgba(0, 0, 0, 0.35);
  box-shadow: inset 0.22rem 0 var(--bng-orange-500);
}

.fields {
  display: grid;
  grid-template-columns: minmax(14rem, 2fr) minmax(9rem, 1fr);
  gap: 0.75rem;
}

.field {
  display: flex;
  flex-direction: column;
  gap: 0.35rem;
  color: var(--bng-cool-gray-100);
  font-size: 0.85rem;
  font-weight: 600;
}

.input-shell {
  display: flex;
  align-items: stretch;
  min-height: 2.7rem;
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

  .direct-input {
        width: 100%;
  }
}

.field-prefix {
  display: grid;
  min-width: 2.6rem;
  place-items: center;
  padding: 0 0.5rem;
  border-right: 1px solid rgba(255, 255, 255, 0.14);
  color: var(--bng-cool-gray-200);
  background: rgba(255, 255, 255, 0.07);
  font-size: 0.78rem;
  font-weight: 700;
}

.actions {
  display: flex;
  gap: 0.5rem;
  flex-wrap: wrap;
}

@media (max-width: 700px) {
  .fields {
    grid-template-columns: 1fr;
  }
}
</style>
