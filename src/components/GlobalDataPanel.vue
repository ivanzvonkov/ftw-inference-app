<script setup lang="ts">
import { computed } from 'vue'
import { mdiInformationOutline } from '@mdi/js'
import useSettings from '../composables/useSettings'
import { get_layer_year } from '../composables/useSettings'
import useMap from '../composables/useMap'
import { getEffectiveCloudlessYear } from '../layers/S2-Cloudless-Layer'
import useAreaOfInterest from '../composables/useAreaOfInterest'
import type { PlaceResult } from '../composables/useAreaOfInterest'
import useNotifier from '../composables/useNotifier'
import useDownloadGrid from '../composables/useDownloadGrid'
import { transformExtent } from 'ol/proj'
import GeocodingSearch from './GeocodingSearch.vue'

const { settings, defaultSettings } = useSettings()
const { map } = useMap()

const basemapYearMismatch = computed(
  () => settings.value.year !== getEffectiveCloudlessYear(settings.value.year),
)
const thresholdMayHideFields = computed(() => settings.value.threshold >= 90)
const resetThreshold = () => {
  settings.value.threshold = defaultSettings.threshold
}
const { fitToExtent } = useAreaOfInterest()
const { showError } = useNotifier()
const { downloadFormat } = useDownloadGrid()

const handleLocationSelected = (place: PlaceResult) => {
  if (!map.value) return
  if (!place.boundingbox) {
    showError('Selected place does not have a bounding box.')
    return
  }
  const [south, north, west, east] = place.boundingbox.map(Number)
  const extent = transformExtent([west, south, east, north], 'EPSG:4326', 'EPSG:3857')
  fitToExtent(map.value, extent, 0, 15)
}

const layerSelection = computed({
  get: () => settings.value.layerName,
  set: (name: string) => {
    settings.value.layerName = name
    settings.value.year = get_layer_year(name)
    settings.value.downloads = false
  }
})
const showSouthAmerica = true
const isGlobal = computed(() => (settings.value.layerName === "Global2024" || settings.value.layerName === "Global2025"))
const isSouthAmerica = computed(() => (settings.value.layerName === "SouthAmerica2024"))
const infoText = computed(() => {
  switch (layerSelection.value) {
    case 'Global2025':
    case 'Global2024':
      return `Global field boundaries generated with the
        <a href="https://arxiv.org/abs/2603.27101" target="_blank" rel="noopener">FTW v3: CC-BY, B7</a> model.`
    case 'SouthAmerica2024':
      return `South American field boundaries generated with customized model and MapBiomas.`
    default:
      return ''
  }
})

</script>

<template>
  <div class="settings">
    <GeocodingSearch @location-selected="handleLocationSelected" />
    <v-radio-group v-model="layerSelection" :messages="[infoText]">
      <v-radio label="Global Fields 2025" value="Global2025"></v-radio>
      <v-radio label="Global Fields 2024" value="Global2024"></v-radio>
      <template v-if="showSouthAmerica">
        <v-radio label="South America Fields 2024" value="SouthAmerica2024"></v-radio>
      </template>
      <template #message="{ message }">
      <span v-html="message"></span>
    </template>
    </v-radio-group>

    <br/>
    <br/>

    <v-row class="d-flex justify-center w-100 mx-auto mb-0">
      <v-col>
        <div class="text-body-small">
            Opacity
        </div>
        <v-slider
          v-model.number="settings.opacity"
          :min="0"
          :max="100"
          :step="1"
          color="teal"
          track-color="grey-darken-2"
          thumb-color="teal"
          hide-details
        />
        <br/>
        <template v-if="isGlobal">
          <div class="text-body-small">
            Confidence Threshold {{ settings.threshold }}%
          </div>
          <v-slider
            v-model.number="settings.threshold"
            :min="0"
            :max="100"
            :step="1"
            color="teal"
            track-color="grey-darken-2"
            thumb-color="teal"
            hide-details
          />
          <v-alert
            v-if="thresholdMayHideFields"
            type="warning"
            variant="tonal"
            density="compact"
            class="mt-2 note"
          >
            A high confidence threshold may hide most or all fields in this area.
            <v-btn @click="resetThreshold" size="small" class="mt-2"> Reset threshold </v-btn>
          </v-alert>
          <br/>
        </template>

        <div class="group group-with-help">
          <div class="text-body-small">Download</div>
          <template v-if="isGlobal">
              <v-menu open-on-hover :close-on-content-click="false" max-width="400">
                <template #activator="{ props }">
                  <v-icon :icon="mdiInformationOutline" size="small" v-bind="props"></v-icon>
                </template>
                <v-sheet class="pa-3 text-body-2">
                  <p class="pb-2">
                    After activation, click a tile to download the agricultural field boundary
                    predictions for that 1° cell as a GeoParquet or GeoJSON file. The confidence
                    threshold and opacity settings do not affect the downloaded data; all fields
                    predicted by the model for the selected cell and year will be included.
                  </p>
                  <p>
                    You can also download the entire dataset in various variants from
                    <a href="https://source.coop/ftw/global-data/" target="_blank" rel="noopener"
                      >our Source Cooperative repository</a
                    >.
                  </p>
                </v-sheet>
              </v-menu>
          </template>
        </div>
        <template v-if="isGlobal">
          <v-switch
            v-model="settings.downloads"
            color="teal"
            density="compact"
            hide-details
            label="Show download grid"
            class="mb-1"
          />
          <template v-if="settings.downloads">
            <v-radio-group v-model="downloadFormat" density="compact" hide-details inline>
              <v-radio label="GeoParquet" value="parquet"></v-radio>
              <v-radio label="GeoJSON" value="json"></v-radio>
            </v-radio-group>
            <p class="text-caption text-medium-emphasis mt-2 mb-1">
              Click any grid cell on the map to download.
            </p>
          </template>
          
        </template>
        <template v-if="isSouthAmerica">
          <div class="text-body-small">Fields can be downloaded from <a href="source.coop">source.coop</a>.</div>
        </template>
      </v-col>
    </v-row>
  </div>
</template>

<style scoped>
.group {
  font-weight: 500;
  font-size: 1.1rem;
}
.group-with-help {
  display: flex;
  align-items: center;
  gap: 0.25rem;
}
.group-with-help h3 {
  margin: 0;
  font-weight: inherit;
  font-size: inherit;
}
.group.legend {
  margin-top: 1rem;
  border-top: 1px solid rgba(136, 136, 136, 0.65);
  padding-top: 1rem;
}
:deep(.v-selection-control-group--inline) {
  gap: 1rem;
}
</style>
