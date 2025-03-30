  <template>
    <div class="sonos-button">
      <b-button
        title="Play on Sonos"
        variant="link"
        class="m-0"
        @click.stop="toggleDropdown"
      >
        <svg viewBox="0 0 24 24" width="16" height="16" fill="currentColor">
          <path d="M14.5 12.5a2.5 2.5 0 1 1-5 0 2.5 2.5 0 0 1 5 0z" />
          <path d="M9.49 8.675c.8-1.2 2.24-2 3.89-2 2.56 0 4.62 2.02 4.62 4.5s-2.06 4.5-4.62 4.5c-1.65 0-3.09-.8-3.89-2" stroke="currentColor" fill="none" />
          <path d="M6.79 6.055c1.52-2.08 4.05-3.43 6.92-3.43 4.69 0 8.46 3.73 8.46 8.5 0 4.77-3.77 8.5-8.46 8.5-2.87 0-5.4-1.35-6.92-3.43" stroke="currentColor" fill="none" />
        </svg>
      </b-button>

      <b-popover ref="popover" :target="() => $el.firstChild" triggers="click blur" placement="left" no-fade>
        <template #title>
          Play on Sonos
        </template>
        <template v-if="loading">
          <div class="text-center p-2">
            <b-spinner small variant="secondary" />
            <div class="mt-2">
              Looking for devices...
            </div>
          </div>
        </template>

        <template v-else-if="!sonosApiConfigured">
          <div class="p-2">
            <div class="form-group">
              <label for="sonos-api-url">Sonos API URL</label>
              <input
                id="sonos-api-url"
                v-model="sonosApiUrl"
                class="form-control form-control-sm"
                placeholder="http://localhost:5005" />
              <small class="text-muted">
                URL zum node-sonos-http-api Server
              </small>
            </div>
            <div class="form-group">
              <label for="external-url">External URL (Optional)</label>
              <input
                id="external-url"
                v-model="externalUrl"
                class="form-control form-control-sm"
                placeholder="http://192.168.1.100:4040" />
              <small class="text-muted">
                URL für Sonos zum Zugriff auf den Server
              </small>
            </div>
            <div class="text-right mt-2">
              <b-button size="sm" variant="primary" @click="saveSettings">
                Save
              </b-button>
            </div>
          </div>
        </template>

        <template v-else-if="error">
          <div class="text-center p-2">
            <div>{{ error }}</div>
            <b-button class="mt-2" size="sm" variant="link" @click="showSettings">
              Configure Sonos API
            </b-button>
          </div>
        </template>

        <template v-else-if="devices.length === 0">
          <div class="text-center p-2">
            No Sonos devices found
          </div>
        </template>

        <template v-else>
          <b-list-group flush>
            <b-list-group-item
              v-for="device in devices"
              :key="device.uuid"
              button
              @click="playOnDevice(device)"
            >
              <div class="d-flex justify-content-between align-items-center">
                <div>{{ device.roomName }}</div>
                <small class="text-muted">{{ device.state }}</small>
              </div>
            </b-list-group-item>
          </b-list-group>
          <div class="text-right p-2">
            <b-button size="sm" variant="outline-secondary" @click="refreshDevices">
              Refresh
            </b-button>
          </div>
        </template>
      </b-popover>
    </div>
  </template>

  <script lang="ts">
  import { defineComponent } from 'vue'
  import { BPopover } from 'bootstrap-vue'
  import { Track } from '@/shared/api'

  interface SonosDevice {
    uuid: string
    name: string
    roomName: string
    state: string
    coordinator: boolean
  }

  interface SonosZone {
    coordinator: SonosDevice
    members: SonosDevice[]
  }

  export default defineComponent({
    name: 'SonosButton',
    components: {
      BPopover,
    },
    props: {
      track: {
        type: Object as () => Track,
        required: true
      }
    },
    data() {
      return {
        loading: false,
        error: null as string | null,
        devices: [] as SonosDevice[],
        currentDevice: null as string | null,
        // Felder für die Inline-Konfiguration
        sonosApiUrl: localStorage.getItem('sonos_api_url') || 'http://localhost:5005',
        externalUrl: localStorage.getItem('external_url') || '',
        showConfigForm: false
      }
    },
    computed: {
      sonosApiConfigured(): boolean {
        return this.showConfigForm ? false : !!localStorage.getItem('sonos_api_url')
      }
    },
    methods: {
      toggleDropdown() {
        // Immer wenn das Dropdown geöffnet wird, prüfen wir die Konfiguration
        if (this.sonosApiConfigured) {
          this.loadDevices()
        }
        // Sonst wird automatisch das Konfigurationsformular angezeigt
      },

      showSettings() {
        this.showConfigForm = true
      },

      // Speichern der Einstellungen
      saveSettings() {
        localStorage.setItem('sonos_api_url', this.sonosApiUrl)
        if (this.externalUrl) {
          localStorage.setItem('external_url', this.externalUrl)
        } else {
          localStorage.removeItem('external_url')
        }

        this.error = null
        this.showConfigForm = false
        this.loadDevices()
      },

      async loadDevices() {
        if (!this.sonosApiConfigured) {
          return
        }

        this.loading = true
        this.error = null

        try {
          // Verwende das Sonos-API-direkt über Fetch
          const apiUrl = localStorage.getItem('sonos_api_url')
          const response = await fetch(`${apiUrl}/zones`)

          if (!response.ok) {
            throw new Error(`Failed to connect to Sonos API (${response.status})`)
          }

          const zones: SonosZone[] = await response.json()
          this.devices = []

          // Extrahiere Geräte aus den Zonen
          zones.forEach(zone => {
            // Füge den Koordinator hinzu
            this.devices.push({
              uuid: zone.coordinator.uuid,
              name: zone.coordinator.roomName,
              roomName: zone.coordinator.roomName,
              state: zone.coordinator.state,
              coordinator: true
            })

            // Füge die Mitglieder hinzu (außer den Koordinator)
            zone.members.forEach(member => {
              if (member.uuid !== zone.coordinator.uuid) {
                this.devices.push({
                  uuid: member.uuid,
                  name: member.roomName,
                  roomName: member.roomName,
                  state: member.state,
                  coordinator: false
                })
              }
            })
          })
        } catch (error: any) {
          this.error = error.message || 'Failed to load Sonos devices'
          console.error('Error loading Sonos devices:', error)
        } finally {
          this.loading = false
        }
      },

      async playOnDevice(device: SonosDevice) {
        if (!this.track) {
          return
        }

        try {
          const apiUrl = localStorage.getItem('sonos_api_url')
          const externalUrl = localStorage.getItem('external_url') || window.location.origin

          // Erstelle die Stream-URL für den Track
          const streamUrl = this.track.url || `${externalUrl}/rest/stream?id=${this.track.id}&v=1.15.0`

          // Kodiere die Parameter
          const encodedDeviceName = encodeURIComponent(device.name || device.roomName)
          const encodedUrl = encodeURIComponent(streamUrl)

          // Sende die Anfrage an die Sonos-API
          const response = await fetch(`${apiUrl}/${encodedDeviceName}/setavtransporturi/${encodedUrl}`)

          if (!response.ok) {
            throw new Error(`Failed to play on device (${response.status})`)
          }

          this.currentDevice = device.name
          // Sicherer Zugriff auf das Popover
          const popover = this.$refs.popover as any
          if (popover?.hide) {
            popover.hide()
          }
        } catch (error: any) {
          console.error('Error playing on Sonos device:', error)
          // Zeige optional eine Fehlermeldung an
        }
      },

      refreshDevices() {
        this.loadDevices()
      }
    }
  })
  </script>

  <style scoped>
  .sonos-button {
    display: inline-block;
  }

  /* Stile für das Einstellungsformular */
  .form-group {
    margin-bottom: 10px;
  }

  .form-control {
    background: rgba(255, 255, 255, 0.1);
    border: 1px solid rgba(255, 255, 255, 0.2);
    color: white;
  }

  .form-control:focus {
    background: rgba(255, 255, 255, 0.15);
    border-color: rgba(255, 255, 255, 0.3);
    color: white;
    box-shadow: 0 0 0 0.2rem rgba(255, 255, 255, 0.1);
  }

  label {
    font-size: 12px;
    margin-bottom: 3px;
  }

  small.text-muted {
    font-size: 11px;
    opacity: 0.7;
  }
  </style>
