<template>
  <div id="app-root">
    <!-- ── Nav ── -->
    <nav class="lmec-nav">
      <div class="nav-inner">
        <div class="nav-brand">
          <a href="https://leventhalmap.org" target="_blank" class="nav-logo">LMEC</a>
          <span class="nav-chevron">⧽</span>
          <span class="nav-title">IIIF Tools</span>
        </div>
        <div class="nav-search">
          <input
            class="nav-input"
            type="text"
            v-model="enteredUrl"
            placeholder="Paste a LMEC, Digital Commonwealth, or external IIIF Manifest URL…"
            @keyup.enter="resolveEnteredUrl"
          />
          <button class="nav-btn" @click="resolveEnteredUrl" :class="{ 'is-loading': state === 'loading' }">
            Open
          </button>
        </div>
      </div>
    </nav>

    <section id="main">
      <!-- ── Error ── -->
      <div class="container is-fluid" v-if="error">
        <div class="notification is-warning is-light">{{ errorMessage }}</div>
      </div>

      <!-- ── Object Info (Commonwealth) ── -->
      <div class="container is-fluid" v-if="manifestLoaded && sourceType === 'commonwealth'">
        <div class="box">
          <h2 class="section-heading">Object Information</h2>
          <div class="url-row" v-for="(field, key) in objectInfoFields" :key="key">
            <a class="url-label button is-link is-small" :href="field.url" target="_blank">{{ field.label }}</a>
            <input class="url-value input is-small" type="text" readonly :value="field.url" />
            <button class="button is-small" @click="copyToClipboard(field.url)">📋</button>
          </div>
        </div>
      </div>

      <!-- ── Canvas Grid ── -->
      <div class="container is-fluid" v-if="manifestLoaded">
        <div class="box">
          <div class="canvas-grid-header">
            <h2 class="section-heading">Images</h2>
            <span class="canvas-count" v-if="allCanvases.length">
              {{ allCanvases.length }} canvas{{ allCanvases.length !== 1 ? 'es' : '' }}
            </span>
          </div>
          <div v-if="allCanvases.length" class="canvas-grid">
            <div
              v-for="(canvas) in pagedCanvases"
              :key="canvas.globalIndex"
              class="canvas-card"
              :class="{ 'is-active': loadedCanvasIndex === canvas.globalIndex }"
            >
              <div class="canvas-card-header">
                <span class="canvas-index">{{ canvas.globalIndex + 1 }}</span>
                <span class="canvas-dims" v-if="canvas.height && canvas.width">{{ canvas.height }}×{{ canvas.width }}</span>
              </div>
              <div class="canvas-id">{{ canvas.shortId }}</div>
              <button class="button is-small is-primary canvas-load-btn" @click="loadCanvasFromGrid(canvas)">Load</button>
            </div>
          </div>
          <p v-else class="has-text-grey">No images found in this manifest.</p>
          <div class="canvas-pagination" v-if="totalPages > 1">
            <button class="button is-small" @click="canvasPage = 0" :disabled="canvasPage === 0">«</button>
            <button class="button is-small" @click="canvasPage--" :disabled="canvasPage === 0">‹</button>
            <span class="pagination-label">Page {{ canvasPage + 1 }} / {{ totalPages }}</span>
            <button class="button is-small" @click="canvasPage++" :disabled="canvasPage >= totalPages - 1">›</button>
            <button class="button is-small" @click="canvasPage = totalPages - 1" :disabled="canvasPage >= totalPages - 1">»</button>
          </div>
        </div>
      </div>

      <!-- ── Image Controls + Map (side by side when image loaded) ── -->
      <div class="container is-fluid" v-if="state === 'imageLoaded' || manifestLoaded">
        <div class="image-map-layout" :class="{ 'has-image': state === 'imageLoaded' }">

          <!-- Left panel: controls -->
          <div class="controls-panel box" v-if="state === 'imageLoaded'">
            <h2 class="section-heading mb-2">{{ loadedImgLabel }}</h2>

            <!-- IIIF endpoint -->
            <div class="compact-row">
              <span class="compact-label">IIIF Endpoint</span>
              <div class="compact-inputs">
                <input class="input is-small" type="text" readonly :value="loadedImgIIIF" />
                <a class="button is-small is-link" :href="loadedImgIIIF" target="_blank">↗</a>
                <button class="button is-small" @click="copyToClipboard(loadedImgIIIF)">📋</button>
              </div>
            </div>

            <!-- Controls row -->
            <div class="inline-controls">
              <div class="inline-control">
                <label class="compact-label">Quality</label>
                <div class="select is-small">
                  <select v-model="imgType">
                    <option>default</option>
                    <option>color</option>
                    <option>gray</option>
                    <option>bitonal</option>
                  </select>
                </div>
              </div>
              <div class="inline-control">
                <label class="compact-label">Resolution</label>
                <div class="select is-small">
                  <select v-model="imgResolutionType">
                    <option>full</option>
                    <option>Maximum height</option>
                    <option>Maximum width</option>
                    <option>Percent</option>
                  </select>
                </div>
              </div>
              <div class="inline-control" v-if="imgResolutionType !== 'full'">
                <label class="compact-label">Value</label>
                <input class="input is-small" type="text" v-model="imgResolutionVariable" style="width:72px" />
              </div>
            </div>

            <!-- Rotation slider -->
            <div class="inline-controls">
              <div class="inline-control rotation-control">
                <label class="compact-label">Rotation</label>
                <div class="rotation-slider-row">
                  <input type="range" min="0" max="359" step="1" v-model.number="imgRotation" class="rotation-slider" />
                  <input
                    class="input is-small rotation-input"
                    type="number"
                    min="0" max="359"
                    :value="imgRotation"
                    @change="imgRotation = Math.min(359, Math.max(0, Math.round(Number(($event.target as HTMLInputElement).value))))"
                    @keyup.enter="($event.target as HTMLInputElement).blur()"
                  />
                  <span class="rotation-deg">°</span>
                  <button class="button is-small" @click="imgRotation = 0" title="Reset rotation">↺</button>
                </div>
              </div>
            </div>

            <!-- Uncropped image URL -->
            <div class="compact-row">
              <span class="compact-label">Uncropped</span>
              <div class="compact-inputs">
                <input class="input is-small" type="text" readonly :value="uncroppedImage" />
                <a class="button is-small is-link" :href="uncroppedImage" target="_blank">↗</a>
                <button class="button is-small" @click="copyToClipboard(uncroppedImage)">📋</button>
              </div>
            </div>

            <!-- Extent fields -->
            <div class="compact-section-label">Extent coordinates</div>
            <div class="compact-row" v-for="(field, key) in extentFields" :key="key">
              <span class="compact-label">{{ field.shortLabel }}</span>
              <div class="compact-inputs">
                <input class="input is-small" type="text" readonly :value="field.value" />
                <button class="button is-small" @click="copyToClipboard(String(field.value))">📋</button>
              </div>
            </div>

            <!-- Cropped image URL -->
            <div class="compact-row">
              <span class="compact-label">Cropped</span>
              <div class="compact-inputs">
                <input class="input is-small" type="text" readonly :value="croppedImage" />
                <a class="button is-small is-link" :href="croppedImage" target="_blank">↗</a>
                <button class="button is-small" @click="copyToClipboard(croppedImage)">📋</button>
              </div>
            </div>

            <p class="crop-hint">Shift + drag on the map to select a crop region</p>
          </div>

          <!-- Right panel: map -->
          <div class="map-panel box" :class="[state === 'imageLoaded' ? '' : 'is-invisible']">
            <div id="ol-map"></div>
          </div>

        </div>
      </div>

    </section>
  </div>
</template>

<script setup lang="ts">
import { ref, computed, onMounted, watch, nextTick } from 'vue'
import Map from 'ol/Map.js'
import View from 'ol/View.js'
import TileLayer from 'ol/layer/Tile.js'
import IIIF from 'ol/source/IIIF'
import IIIFInfo from 'ol/format/IIIFInfo'
import ExtentInteraction from 'ol/interaction/Extent'

// ── State ────────────────────────────────────────────────────────────────────
const state = ref<'initial' | 'loading' | 'imageLoaded' | 'error'>('initial')
const error = ref(false)
const errorMessage = ref<string | null>(null)
const manifestLoaded = ref(false)
const sourceType = ref<string | null>(null)
const manifestUrl = ref<string | null>(null)
const enteredUrl = ref('')
const commonwealthObjectId = ref<string | null>(null)
const manifest = ref<any>(null)
const manifestVersion = ref<2 | 3 | null>(null)
const loadedImg = ref<number[]>([])
const loadedImgLabel = ref('')
const imgType = ref('default')
const extentCoords = ref<number[]>([0, 0, 0, 0])
const imgResolutionType = ref('full')
const imgResolutionVariable = ref(1200)
const canvasPage = ref(0)
const loadedCanvasIndex = ref<number | null>(null)
const imgRotation = ref(0)
const imageApiVersion = ref<2 | 3>(2)

// OL refs
let olMap: Map | null = null
let olLayer: TileLayer<IIIF> | null = null
let olExtent: ExtentInteraction | null = null

// ── Clipboard ────────────────────────────────────────────────────────────────
function copyToClipboard(text: string) {
  navigator.clipboard.writeText(text).catch(() => {
    // fallback
    const el = document.createElement('textarea')
    el.value = text
    document.body.appendChild(el)
    el.select()
    document.execCommand('copy')
    document.body.removeChild(el)
  })
}

// ── Computed: object info ────────────────────────────────────────────────────
const lmecObjectUrl = computed(() =>
  `https://collections.leventhalmap.org/search/${commonwealthObjectId.value}`
)
const dcObjectUrl = computed(() =>
  `https://www.digitalcommonwealth.org/search/${commonwealthObjectId.value}`
)
const arkLink = computed(() => {
  if (!manifest.value?.metadata) return ''
  const idf = manifest.value.metadata.find(
    (d: any) => d.label === 'Identifier' || d.label?.en?.[0] === 'Identifier'
  )
  if (!idf) return ''
  const val = idf.value
  if (!val) return ''
  if (typeof val === 'string') return val
  if (Array.isArray(val)) return val[0]
  if (typeof val === 'object') {
    const first = Object.values(val)[0]
    return Array.isArray(first) ? first[0] : String(first)
  }
  return ''
})

const objectInfoFields = computed(() => [
  { label: 'Digital Commonwealth URL', url: dcObjectUrl.value },
  { label: 'LMEC Collections URL', url: lmecObjectUrl.value },
  { label: 'ARK Permalink', url: arkLink.value },
  { label: 'IIIF Manifest', url: manifestUrl.value ?? '' },
])

// Flatten all canvases into a single list for the grid regardless of manifest version
const allCanvases = computed(() => {
  if (!manifest.value) return []
  const result: any[] = []
  if (manifestVersion.value === 2) {
    const sequences = manifest.value.sequences ?? []
    sequences.forEach((seq: any, si: number) => {
      (seq.canvases ?? []).forEach((canvas: any, ci: number) => {
        (canvas.images ?? []).forEach((_img: any, ii: number) => {
          const id: string = canvas['@id'] ?? ''
          result.push({
            globalIndex: result.length,
            seqIndex: si, canIndex: ci, imgIndex: ii,
            height: canvas.height, width: canvas.width,
            shortId: id.split('/').pop() ?? id,
            fullId: id,
            version: 2,
          })
        })
      })
    })
  } else if (manifestVersion.value === 3) {
    const items = manifest.value.items ?? []
    items.forEach((canvas: any, ci: number) => {
      (canvas.items ?? []).forEach((page: any, pi: number) => {
        (page.items ?? []).forEach((_item: any, ii: number) => {
          const id: string = canvas.id ?? ''
          result.push({
            globalIndex: result.length,
            canIndex: ci, pageIndex: pi, itemIndex: ii,
            height: canvas.height, width: canvas.width,
            shortId: id.split('/').pop() ?? id,
            fullId: id,
            version: 3,
          })
        })
      })
    })
  }
  return result
})

const PAGE_SIZE = 24
const totalPages = computed(() => Math.ceil(allCanvases.value.length / PAGE_SIZE))
const pagedCanvases = computed(() => {
  const start = canvasPage.value * PAGE_SIZE
  return allCanvases.value.slice(start, start + PAGE_SIZE)
})

const extentFields = computed(() => [
  { label: 'Extent (Map Format)', shortLabel: 'Map', value: extentCoords.value },
  { label: 'Extent (Map Format, Rounded)', shortLabel: 'Map (rounded)', value: extentCoordsRounded.value },
  { label: 'Extent (IIIF Format)', shortLabel: 'IIIF', value: extentCoordsIIIF.value },
])

// ── Computed: IIIF image service endpoint ────────────────────────────────────
const loadedImgIIIF = computed(() => {
  if (!manifest.value || loadedImg.value.length === 0) return ''
  return extractServiceId(manifest.value, loadedImg.value)
})

function extractServiceId(m: any, indices: number[]): string {
  try {
    if (manifestVersion.value === 3) {
      const [canIndex, pageIndex, itemIndex] = indices
      const canvas = m.items[canIndex]
      const annotationPage = canvas.items[pageIndex]
      const annotation = annotationPage.items[itemIndex]
      const body = annotation.body
      return getServiceIdFromBody(body)
    } else {
      // v2
      const [seqIndex, canIndex, imgIndex] = indices
      const resource = m.sequences[seqIndex].canvases[canIndex].images[imgIndex].resource
      return getServiceIdV2(resource)
    }
  } catch (e) {
    console.error('Could not extract service ID', e)
    return ''
  }
}

function getServiceIdV2(resource: any): string {
  if (!resource) return ''
  // Direct service on resource
  if (resource.service) {
    const svc = resource.service
    const id = svc['@id'] || svc.id
    if (id) return id
    // Array of services
    if (Array.isArray(svc)) {
      for (const s of svc) {
        const sid = s['@id'] || s.id
        if (sid) return sid
      }
    }
  }
  // Fallback: strip trailing /full/full/0/default.jpg etc from @id
  const rawId: string = resource['@id'] || resource.id || ''
  return stripIIIFSuffix(rawId)
}

function getServiceIdFromBody(body: any): string {
  if (!body) return ''
  // Body may be an array (choice) or a single object
  const target = Array.isArray(body) ? body[0] : body

  const serviceArr = target.service
    ? Array.isArray(target.service) ? target.service : [target.service]
    : []

  for (const svc of serviceArr) {
    const id = svc.id || svc['@id']
    if (id) return id
  }

  // Fallback: use the body id and strip IIIF path suffixes
  const rawId: string = target.id || target['@id'] || ''
  return stripIIIFSuffix(rawId)
}

function stripIIIFSuffix(url: string): string {
  // Remove trailing /full/full/0/default.jpg or similar IIIF image API suffixes
  return url.replace(/\/(full|square|[\d,]+)\/(full|max|[\d,!^pct:]+)\/[\d]+\/(default|color|gray|bitonal)\.\w+$/, '')
}

// ── Computed: image URL helpers ──────────────────────────────────────────────
const resolution = computed(() => {
  // IIIF v3 uses 'max' where v2 used 'full'
  const fullKeyword = imageApiVersion.value === 3 ? 'max' : 'full'
  if (imgResolutionType.value === 'full') return fullKeyword
  if (imgResolutionType.value === 'Maximum height') return `,${imgResolutionVariable.value}`
  if (imgResolutionType.value === 'Maximum width') return `${imgResolutionVariable.value},`
  if (imgResolutionType.value === 'Percent') return `pct:${imgResolutionVariable.value}`
  return fullKeyword
})

const uncroppedImage = computed(() =>
  `${loadedImgIIIF.value}/full/${resolution.value}/${imgRotation.value}/${imgType.value}.jpg`
)

const extentCoordsRounded = computed(() => extentCoords.value.map(d => Math.round(d)))

const extentCoordsIIIF = computed(() => {
  const c = extentCoords.value
  return [
    c[0],
    c[3] * -1,
    c[2] - c[0],
    c[1] * -1 - c[3] * -1,
  ].map(d => Math.round(d))
})

const croppedImage = computed(() => {
  const [x, y, w, h] = extentCoordsIIIF.value
  return `${loadedImgIIIF.value}/${x},${y},${w},${h}/${resolution.value}/${imgRotation.value}/${imgType.value}.jpg`
})

// ── Methods ──────────────────────────────────────────────────────────────────
function resolveEnteredUrl() {
  state.value = 'loading'
  error.value = false
  manifest.value = null
  manifestLoaded.value = false

  const url = enteredUrl.value.trim()

  const reCommonwealth = /search\/(commonwealth:[^/?#]+)/
  const matchCommonwealth = reCommonwealth.exec(url)

  const reArchive = /archive\.org\/details\/([^/?#]+)/
  const matchArchive = reArchive.exec(url)

  // LMEC collections file set pattern
  const reLMEC = /\/2\/(commonwealth:[^/?#]+)/
  const matchLMEC = reLMEC.exec(url)

  if (matchCommonwealth) {
    const objId = `search/${matchCommonwealth[1]}`
    commonwealthObjectId.value = objId
    sourceType.value = 'commonwealth'
    manifestUrl.value = `https://www.digitalcommonwealth.org/${objId}/manifest.json`
    getManifest()
  } else if (matchArchive) {
    sourceType.value = 'external'
    manifestUrl.value = `https://iiif.archivelab.org/iiif/${matchArchive[1]}/manifest.json`
    getManifest()
  } else if (matchLMEC) {
    fetch(`https://collections.leventhalmap.org/search/${matchLMEC[1]}.json`)
      .then(r => r.json())
      .then(data => {
        const uri = data?.response?.document?.is_file_set_of_ssim?.[0]
        if (!uri) throw new Error('Could not find file set URI')
        sourceType.value = 'commonwealth'
        commonwealthObjectId.value = `search/${uri}`
        manifestUrl.value = `https://www.digitalcommonwealth.org/search/${uri}/manifest.json`
        getManifest()
      })
      .catch(e => {
        setError(`Failed to resolve LMEC URL: ${e.message}`)
      })
  } else {
    // Treat as direct manifest URL
    sourceType.value = 'external'
    manifestUrl.value = url
    getManifest()
  }
}

function getManifest() {
  if (!manifestUrl.value) return
  fetch(manifestUrl.value)
    .then(r => {
      if (!r.ok) throw new Error(`HTTP ${r.status}: ${r.statusText}`)
      return r.json()
    })
    .then(d => {
      manifest.value = d
      manifestVersion.value = detectManifestVersion(d)
      manifestLoaded.value = true
      state.value = 'initial'
    })
    .catch(e => {
      console.error(e)
      setError(`Could not load manifest. Check the URL and ensure the server allows cross-origin requests. (${e.message})`)
    })
}

function detectManifestVersion(m: any): 2 | 3 | null {
  const ctx = m['@context']
  if (!ctx) return null
  const ctxStr = Array.isArray(ctx) ? ctx.join(' ') : String(ctx)
  if (ctxStr.includes('iiif.io/api/presentation/3')) return 3
  if (ctxStr.includes('iiif.io/api/presentation/2')) return 2
  // Heuristic fallback
  if (m.sequences) return 2
  if (m.items) return 3
  return null
}

function setError(msg: string) {
  state.value = 'error'
  error.value = true
  errorMessage.value = msg
}

function loadCanvasFromGrid(canvas: any) {
  loadedCanvasIndex.value = canvas.globalIndex
  if (canvas.version === 3) {
    loadImageV3(canvas.canIndex, canvas.pageIndex, canvas.itemIndex)
  } else {
    loadImage(canvas.seqIndex, canvas.canIndex, canvas.imgIndex)
  }
}

function loadImage(seqIndex: number, canIndex: number, imgIndex: number) {
  loadedImg.value = [seqIndex, canIndex, imgIndex]
  loadedImgLabel.value = `Sequence ${seqIndex}, Canvas ${canIndex}, Image ${imgIndex}`
  state.value = 'imageLoaded'
  // Compute service ID directly — don't rely on the computed ref which hasn't flushed yet
  const serviceId = extractServiceId(manifest.value, [seqIndex, canIndex, imgIndex])
  console.log('[loadImage] serviceId:', serviceId)
  if (!serviceId) { alert('Could not find IIIF image service endpoint for this canvas.'); return }
  loadIIIFLayer(serviceId + '/info.json')
}

function loadImageV3(canIndex: number, pageIndex: number, itemIndex: number) {
  loadedImg.value = [canIndex, pageIndex, itemIndex]
  loadedImgLabel.value = `Canvas ${canIndex}, AnnotationPage ${pageIndex}, Item ${itemIndex}`
  state.value = 'imageLoaded'
  const serviceId = extractServiceId(manifest.value, [canIndex, pageIndex, itemIndex])
  console.log('[loadImageV3] serviceId:', serviceId)
  if (!serviceId) { alert('Could not find IIIF image service endpoint for this canvas.'); return }
  loadIIIFLayer(serviceId + '/info.json')
}

/** Normalize an info.json to something OL's IIIFInfo can parse.
 *  OL handles v1/v2 well; v3 needs the @context rewritten to v2. */
function normalizeInfoJson(info: any): any {
  const ctx: string = Array.isArray(info['@context'])
    ? info['@context'].join(' ')
    : String(info['@context'] ?? '')

  if (!ctx.includes('iiif.io/api/image/3')) return info  // already v1/v2, pass through

  // Re-shape v3 → v2 enough for OL to consume
  const out: any = { ...info }

  // @context
  out['@context'] = 'http://iiif.io/api/image/2/context.json'

  // id → @id
  if (info.id && !info['@id']) out['@id'] = info.id

  // type → @type (OL checks for this)
  if (!out['@type']) out['@type'] = 'iiif:ImageApiService'

  // profile: v3 uses an object/array, OL v2 wants a string URI
  if (!out.profile || typeof out.profile !== 'string') {
    out.profile = 'http://iiif.io/api/image/2/level2.json'
  }

  // sizes: v3 uses {width,height}, v2 also uses {width,height} — fine as-is
  // tiles: same shape between v2 and v3 — fine as-is

  console.log('[normalizeInfoJson] rewrote v3 → v2 shim', out)
  return out
}

function loadIIIFLayer(endpoint: string) {
  fetch(endpoint)
    .then(response => {
      if (!response.ok) throw new Error(`HTTP ${response.status} fetching ${endpoint}`)
      return response.json()
    })
    .then(rawInfo => {
      // Detect API version from raw info BEFORE normalizing
      const rawCtx: string = Array.isArray(rawInfo['@context'])
        ? rawInfo['@context'].join(' ')
        : String(rawInfo['@context'] ?? '')
      imageApiVersion.value = rawCtx.includes('iiif.io/api/image/3') ? 3 : 2

      const imageInfo = normalizeInfoJson(rawInfo)
      const options = new IIIFInfo(imageInfo).getTileSourceOptions()
      if (!options || options.version === undefined) {
        console.error('[IIIFInfo] getTileSourceOptions() returned nothing. imageInfo was:', imageInfo)
        alert('Could not parse IIIF image info — see console for details.')
        return
      }
      options.zDirection = -1

      // OL internally uses "2"/"3" but IIIFInfo sometimes returns "version2"/"version3"
      if (options.version === 'version2') options.version = '2'
      if (options.version === 'version3') options.version = '3'

      const iiifTileSource = new IIIF(options)
      olLayer!.setSource(iiifTileSource)

      const tryFit = () => {
        const tileGrid = iiifTileSource.getTileGrid()
        if (!tileGrid) return
        const extent = tileGrid.getExtent()
        olMap!.setView(new View({
          resolutions: tileGrid.getResolutions(),
          extent,
          constrainOnlyCenter: true,
          rotation: (imgRotation.value * Math.PI) / 180,
        }))
        olMap!.getView().fit(extent)
        olMap!.render()
      }

      tryFit()
      iiifTileSource.on('change', () => {
        if (iiifTileSource.getState() === 'ready') tryFit()
      })
    })
    .catch(e => {
      console.error('[loadIIIFLayer] error:', e)
      alert(`Could not load image: ${e.message}`)
    })
}

// ── Lifecycle ────────────────────────────────────────────────────────────────
function initMap() {
  if (olMap) return  // already initialized
  const target = document.getElementById('ol-map')
  if (!target) return

  olLayer = new TileLayer()
  olMap = new Map({
    target: 'ol-map',
    layers: [olLayer],
    view: new View({ center: [0, 0], zoom: 2 }),
  })

  olExtent = new ExtentInteraction()
  olMap.addInteraction(olExtent)
  olExtent.setActive(false)

  window.addEventListener('keydown', (event) => {
    if (event.key === 'Shift') olExtent!.setActive(true)
  })
  window.addEventListener('keyup', (event) => {
    if (event.key === 'Shift') olExtent!.setActive(false)
  })

  olExtent.on('extentchanged', () => {
    const ext = olExtent!.getExtent()
    if (ext) extentCoords.value = ext
  })
}

// Try on mount (works if #ol-map is already in DOM)
onMounted(() => initMap())

// Watch rotation slider and apply to OL map view live
watch(imgRotation, (deg) => {
  if (olMap) {
    olMap.getView().setRotation((deg * Math.PI) / 180)
  }
})

// Also watch manifestLoaded — the v-if wrapper means #ol-map only
// appears in the DOM after a manifest is loaded for the first time
watch(manifestLoaded, async (val) => {
  if (val) {
    await nextTick()
    initMap()
  }
})
</script>

<style>
@import 'ol/ol.css';

/* ── Reset / Root ── */
#app-root {
  min-height: 100vh;
  background: #f5f5f5;
  font-family: 'Segoe UI', system-ui, sans-serif;
}

/* ── Nav ── */
.lmec-nav {
  background: #4a5568;
  padding: 0 1.5rem;
  height: 56px;
  display: flex;
  align-items: center;
  position: sticky;
  top: 0;
  z-index: 100;
  box-shadow: 0 2px 6px rgba(0,0,0,0.25);
}
.nav-inner {
  max-width: 1400px;
  width: 100%;
  margin: 0 auto;
}
.nav-brand {
  display: flex;
  align-items: center;
  gap: 0.5rem;
}
.nav-logo {
  font-weight: 900;
  color: #68d391;
  text-decoration: none;
  font-size: 1rem;
  letter-spacing: 0.05em;
}
.nav-logo:hover { text-decoration: underline; }
.nav-chevron {
  color: #a0aec0;
  font-size: 1rem;
}
.nav-title {
  color: #e2e8f0;
  font-weight: 600;
  font-size: 0.95rem;
}

/* ── Main layout ── */
#main > .container {
  margin-top: 1.25rem;
}

/* ── Canvas grid ── */
.canvas-grid-header {
  display: flex;
  align-items: baseline;
  gap: 0.75rem;
  margin-bottom: 1rem;
}
.canvas-count {
  font-size: 0.8rem;
  color: #718096;
  background: #edf2f7;
  border-radius: 999px;
  padding: 0.1rem 0.6rem;
}

.canvas-grid {
  display: grid;
  grid-template-columns: repeat(auto-fill, minmax(180px, 1fr));
  gap: 0.75rem;
}

.canvas-card {
  background: #fff;
  border: 1px solid #e2e8f0;
  border-radius: 6px;
  padding: 0.6rem 0.75rem;
  display: flex;
  flex-direction: column;
  gap: 0.35rem;
  transition: box-shadow 0.15s, border-color 0.15s;
}
.canvas-card:hover {
  box-shadow: 0 2px 8px rgba(0,0,0,0.1);
  border-color: #a0aec0;
}
.canvas-card.is-active {
  border-color: #3273dc;
  box-shadow: 0 0 0 2px rgba(50, 115, 220, 0.25);
}

.canvas-card-header {
  display: flex;
  justify-content: space-between;
  align-items: center;
}
.canvas-index {
  font-size: 0.7rem;
  font-weight: 700;
  color: #4a5568;
  background: #edf2f7;
  border-radius: 4px;
  padding: 0.1rem 0.4rem;
}
.canvas-dims {
  font-size: 0.65rem;
  color: #a0aec0;
}
.canvas-id {
  font-size: 0.65rem;
  font-family: monospace;
  color: #718096;
  white-space: nowrap;
  overflow: hidden;
  text-overflow: ellipsis;
}
.canvas-load-btn {
  margin-top: 0.25rem;
  width: 100%;
}

/* ── Pagination ── */
.canvas-pagination {
  display: flex;
  align-items: center;
  gap: 0.5rem;
  margin-top: 1rem;
  justify-content: center;
}
.pagination-label {
  font-size: 0.85rem;
  color: #4a5568;
  min-width: 100px;
  text-align: center;
}

/* ── Map ── */
#ol-map {
  height: 500px;
  width: 100%;
  margin-top: 1rem;
  background-color: #1a202c;
  border-radius: 4px;
}
</style>