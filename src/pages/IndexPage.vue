<template>
  <q-page class="q-pa-md column q-gutter-md">
    <!-- Ürün Ekle -->
    <q-card
      flat
      bordered
    >
      <q-card-section class="row items-center q-gutter-sm">
        <div class="text-h6">Ürün Ekle</div>
        <q-space />
        <q-btn
          color="primary"
          icon="qr_code_scanner"
          label="Barkod Tara"
          @click="openScanner('add')"
        />
      </q-card-section>
      <q-separator />
      <q-card-section>
        <q-form
          ref="createFormRef"
          class="q-gutter-md"
        >
          <div class="row q-col-gutter-md">
            <div class="col-12 col-sm-4">
              <q-input
                ref="createFormRef"
                v-model="createForm.barcode"
                label="Barkod"
                outlined
                dense
                hide-bottom-space
                autocomplete="off"
                autocorrect="off"
                autocapitalize="off"
                spellcheck="false"
              />
            </div>
            <div class="col-12 col-sm-4">
              <q-input
                v-model="createForm.name"
                label="Ürün Adı"
                dense
                outlined
                hide-bottom-space
              />
            </div>
            <div class="col-12 col-sm-4">
              <q-input
                v-model.number="createForm.price"
                label="Fiyat"
                dense
                outlined
                type="number"
                step="0.01"
                hide-bottom-space
              />
            </div>
          </div>
          <div class="row">
            <div class="col-12">
              <q-btn
                color="positive"
                type="button"
                label="Kaydet"
                :loading="loading.create"
                :disable="loading.create"
                @click="submitCreate"
              />
            </div>
          </div>
        </q-form>
      </q-card-section>
    </q-card>

    <!-- Sepet / Alışveriş -->
    <q-card
      flat
      bordered
    >
      <q-card-section class="row items-center q-gutter-sm">
        <div class="text-h6">Sepet</div>
        <q-space />
        <q-input
          v-model="searchTerm"
          outlined
          dense
          placeholder="Ürün ara..."
          class="q-mr-sm"
          style="min-width: 200px;"
          clearable
        >
          <template v-slot:prepend>
            <q-icon name="search" />
          </template>
        </q-input>
        <q-btn
          color="primary"
          icon="qr_code"
          label="Barkod Tara"
          @click="openScanner('cart')"
        />
      </q-card-section>
      <q-separator />
      <q-card-section>
        <q-table
          title="Sepet"
          :rows="filteredCart"
          :columns="columns"
          row-key="barcode"
          flat
          :pagination="{ rowsPerPage: 0 }"
          hide-bottom
        >
          <template #body-cell-qty="props">
            <q-td :props="props">
              <q-input
                v-model.number="props.row.qty"
                type="number"
                dense
                outlined
                style="max-width: 90px"
                min="1"
                @update:model-value="recalc()"
              />
            </q-td>
          </template>
          <template #body-cell-total="props">
            <q-td :props="props">
              {{ currency(props.row.price * props.row.qty) }}
            </q-td>
          </template>
          <template #body-cell-actions="props">
            <q-td :props="props">
              <q-btn
                dense
                flat
                color="negative"
                icon="delete"
                @click="removeFromCart(props.row.barcode)"
              />
            </q-td>
          </template>
        </q-table>

        <div class="row q-mt-md items-center">
          <div class="col-12 col-sm-6 text-subtitle1">
            Toplam: <strong>{{ currency(cartTotal) }}</strong>
          </div>
          <div class="col-12 col-sm-6 text-right">
            <q-btn
              color="positive"
              label="Satın Al"
              @click="checkout"
              :disable="cart.length === 0"
            />
          </div>
        </div>
      </q-card-section>
    </q-card>

    <!-- Barkod Okuyucu Dialog -->
    <q-dialog
      v-model="scanner.open"
      persistent
      position="bottom"
      transition-show="scale"
      transition-hide="scale"
    >
      <q-card class="bg-black">
        <q-bar class="bg-grey-9 text-white">
          <div>{{ scanner.mode === 'add' ? 'Ürün Ekle - Barkod Tara' : 'Sepet - Barkod Tara' }}</div>
          <q-space />
          <!-- Kamera Ayarları Menüsü (3 nokta) -->
          <q-btn dense flat round icon="more_vert">
            <q-menu anchor="bottom right" self="top right">
              <q-list style="min-width: 240px">
                <q-item clickable v-ripple @click="refreshCameras">
                  <q-item-section avatar><q-icon name="refresh" /></q-item-section>
                  <q-item-section>Kameraları Yenile</q-item-section>
                </q-item>
                <q-separator />
                <q-item>
                  <q-item-section>Kamera Seç</q-item-section>
                </q-item>
                <template v-if="cameras.length > 0">
                  <q-item 
                    v-for="cam in cameras" 
                    :key="cam.value" 
                    clickable 
                    v-ripple 
                    @click="switchCamera(cam.value)"
                  >
                    <q-item-section avatar>
                      <q-icon :name="selectedCameraId === cam.value ? 'radio_button_checked' : 'radio_button_unchecked'" />
                    </q-item-section>
                    <q-item-section>{{ cam.label || 'Bilinmeyen Kamera' }}</q-item-section>
                  </q-item>
                </template>
                <q-item v-else>
                  <q-item-section class="text-italic text-grey">
                    <div class="row items-center">
                      <q-icon name="info" class="q-mr-sm" size="1.2em" />
                      <span>Kamera bulunamadı</span>
                    </div>
                  </q-item-section>
                </q-item>
                <q-separator />
                <q-item clickable v-ripple @click="mirror = !mirror">
                  <q-item-section avatar><q-icon :name="mirror ? 'toggle_on' : 'toggle_off'" /></q-item-section>
                  <q-item-section>Ayna (Yatay çevir)</q-item-section>
                </q-item>
                <q-item clickable v-ripple @click="rotate180 = !rotate180">
                  <q-item-section avatar><q-icon :name="rotate180 ? 'toggle_on' : 'toggle_off'" /></q-item-section>
                  <q-item-section>Ters Çevir (180°)</q-item-section>
                </q-item>
              </q-list>
            </q-menu>
          </q-btn>
          <q-btn
            dense
            flat
            icon="close"
            v-close-popup
            @click="stopScanner"
          />
        </q-bar>
        <q-card-section class="q-pa-none">
          <div style="position: relative; width: 360px; height: 260px;">
            <video
              ref="videoEl"
              autoplay
              playsinline
              muted
              :style="videoStyle"
            ></video>
            <!-- Orta odak çerçevesi -->
            <div
              class="absolute-center"
              style="width: 70%; height: 70%; max-width: 85%; max-height: 85%; pointer-events:none;"
            >
              <div style="position:absolute; inset:0; border:2px solid rgba(0,255,0,0.8); border-radius:8px;"></div>
              <!-- Köşe çizgileri -->
              <div
                style="position:absolute; top:-2px; left:-2px; width:20%; height:0; border-top:4px solid rgba(0,255,0,0.9);"
              >
              </div>
              <div
                style="position:absolute; top:-2px; right:-2px; width:20%; height:0; border-top:4px solid rgba(0,255,0,0.9);"
              >
              </div>
              <div
                style="position:absolute; bottom:-2px; left:-2px; width:20%; height:0; border-bottom:4px solid rgba(0,255,0,0.9);"
              >
              </div>
              <div
                style="position:absolute; bottom:-2px; right:-2px; width:20%; height:0; border-bottom:4px solid rgba(0,255,0,0.9);"
              >
              </div>
            </div>
          </div>
        </q-card-section>
      </q-card>
    </q-dialog>
  </q-page>
</template>

<script setup>
import { ref, reactive, computed, onBeforeUnmount, nextTick, onMounted } from 'vue'
import { useQuasar } from 'quasar'
import { BrowserMultiFormatReader } from '@zxing/browser'
import { DecodeHintType } from '@zxing/library'

// Axios instance (boot/axios.js -> this.$api)
// Laravel varsayılanı: http://localhost:8000/api
// Gerekirse backend URL'yi src/boot/axios.js içindeki baseURL'den değiştirin.
const $q = useQuasar()
const createFormRef = ref(null)

const createForm = reactive({
  barcode: '',
  name: '',
  price: null,
})

const loading = reactive({ create: false })

// Arama terimi
const searchTerm = ref('')

// Sepet
const cart = ref([]) // { id, barcode, name, price, qty }

// Filtrelenmiş sepet
const filteredCart = computed(() => {
  if (!searchTerm.value) return cart.value
  
  const normalizeText = (text) => {
    return String(text || '')
      .toLocaleLowerCase('tr-TR')
      .normalize('NFD')
      .replace(/[\u0300-\u036f]/g, '')
  }
  
  const term = normalizeText(searchTerm.value)
  
  return cart.value.filter(item => {
    return (
      normalizeText(item.name).includes(term) ||
      normalizeText(item.barcode).includes(term)
    )
  })
})
const columns = [
  { name: 'barcode', label: 'Barkod', field: 'barcode', align: 'left' },
  { name: 'name', label: 'Ürün', field: 'name', align: 'left' },
  { name: 'price', label: 'Birim Fiyat', field: row => currency(row.price), align: 'right' },
  { name: 'qty', label: 'Adet', field: 'qty', align: 'right' },
  { name: 'total', label: 'Toplam', field: 'total', align: 'right' },
  { name: 'actions', label: '', field: 'actions', align: 'right' },
]

const cartTotal = computed(() => cart.value.reduce((sum, i) => sum + i.price * i.qty, 0))
const recalc = () => {
  // trigger recompute; nothing extra needed as we use computed
}

// Barkod Okuyucu
const scanner = reactive({ open: false, mode: 'add' /* 'add' | 'cart' */ })
const videoEl = ref(null)
let scanning = false

// Barkod takip sistemi
const visibleBarcodes = ref(new Set()) // Şu anda kamerada görünen barkodlar
const processedBarcodes = ref(new Set()) // İşlenmiş barkodlar (kameradan çıkana kadar)
const lastScannedBarcode = ref(null) // Son okunan barkod (geçmiş uyumluluk için korundu)
let codeReader = null
// Kamerada son görüldüğü zamanı tut (ms)
const lastSeenAt = ref(new Map())
let detectLock = false
let detectedOnce = false
let lastScannedCode = ''
let lastScanAt = 0
let suppressUntil = 0
let lastSavedCode = ''
// Kayıt sonrası kısa süreli yok sayma penceresi (eski barkodu görse bile işlememe)
let sessionIgnore = { code: '', until: 0 }
// Kamera oturumu başlangıç zamanı
let sessionStartedAt = 0

// Kamera listesi ve seçim
const cameras = ref([]) // [{ label, value }]
const selectedCameraId = ref(null)
const mirror = ref(false)
const rotate180 = ref(false)

const videoStyle = computed(() => {
  const base = {
    width: '100%',
    height: '100%',
    objectFit: 'cover',
    borderRadius: '4px',
  }
  const sx = mirror.value ? -1 : 1
  const rot = rotate180.value ? '180deg' : '0deg'
  return {
    ...base,
    transform: `scaleX(${sx}) rotate(${rot})`,
  }
})

// Ses ayarları (kalıcı)
const sound = reactive({ enabled: true, volume: 0.7 })
function loadSoundSettings() {
  try {
    const en = localStorage.getItem('sound.enabled')
    const vol = localStorage.getItem('sound.volume')
    if (en != null) sound.enabled = en === 'true'
    if (vol != null) {
      const v = Number(vol)
      if (!Number.isNaN(v)) sound.volume = Math.min(1, Math.max(0, v))
    }
  } catch (e) { console.debug('loadSoundSettings ignore:', e?.name || e) }
}
function handleExternalAddToCart(ev) {
  try {
    const p = ev?.detail
    if (!p) return
    const prod = { id: p.id ?? null, barcode: String(p.barcode || ''), name: String(p.name || ''), price: Number(p.price || 0) }
    if (!prod.barcode) return
    // Çekmeceden eklemede sessiz (beep yok)
    addToCart(prod, false)
  } catch (e) { console.debug('add-to-cart listener ignore:', e?.name || e) }
}

onMounted(() => {
  loadSoundSettings()
  try { window.addEventListener('sound-settings-changed', loadSoundSettings) } catch (e) { console.debug('event add ignore:', e?.name || e) }
  try { window.addEventListener('add-to-cart', handleExternalAddToCart) } catch (e) { console.debug('event add ignore:', e?.name || e) }
})

async function loadCameras() {
  try {
    // 1) Önce kamera iznini al
    console.debug('[loadCameras] Requesting camera access...')
    
    // Mevcut tüm kameraları temizle
    cameras.value = []
    
    try {
      // Kameraya erişim izni iste
      const stream = await navigator.mediaDevices.getUserMedia({ 
        video: { 
          width: { ideal: 1280 },
          height: { ideal: 720 },
          facingMode: { ideal: 'environment' }
        } 
      })
      
      // İzni aldıktan sonra cihaz listesini al
      const devicesAll = await navigator.mediaDevices.enumerateDevices()
      const devices = devicesAll.filter(d => d.kind === 'videoinput')
      
      console.debug('[loadCameras] Found video devices:', devices)
      
      // Mevcut kameraları güncelle
      cameras.value = devices.map((d, i) => ({
        label: d.label || `Kamera ${i + 1}`,
        value: d.deviceId,
      }))
      
      // Kamera seçimini güncelle
      if (cameras.value.length > 0) {
        // Önce seçili kamerayı kontrol et
        const currentDeviceExists = devices.some(d => d.deviceId === selectedCameraId.value)
        
        if (!currentDeviceExists) {
          // Arka kamerayı bul veya ilk kamerayı seç
          const backCamera = devices.find(d => /back|rear|environment/i.test(d.label || ''))
          selectedCameraId.value = backCamera ? backCamera.deviceId : devices[0].deviceId
          console.debug('[loadCameras] Selected camera:', selectedCameraId.value)
        }
      } else {
        selectedCameraId.value = null
      }
      
      // Kullanılmayan stream'i temizle
      stream.getTracks().forEach(track => track.stop())
      
      console.debug('[loadCameras] Camera list updated:', cameras.value)
      
    } catch (err) {
      console.error('Kamera erişim hatası:', err)
      $q.notify({
        type: 'negative',
        message: 'Kameraya erişilemiyor. Lütfen tarayıcı izinlerinizi kontrol edin.',
        timeout: 3000
      })
    }
    
  } catch (e) {
    console.error('Kamera yüklenirken hata oluştu:', e)
    $q.notify({
      type: 'negative',
      message: 'Kamera cihazları listelenirken bir hata oluştu.',
      timeout: 3000
    })
  }
}

// Kamera listesi yenile
async function refreshCameras() {
  await loadCameras()
}

// Kamera değiştir ve taramayı güvenle yeniden başlat
async function switchCamera(id) {
  try {
    if (selectedCameraId.value === id) return
    selectedCameraId.value = id
    stopScanner()
    await nextTick()
    await startScanner()
  } catch (e) {
    console.debug('switchCamera failed:', e)
    $q.notify({ type: 'negative', message: 'Kamera değiştirilemedi' })
  }
}

// Kamera seçicisi kaldırıldı; otomatik kamera seçimi kullanılacak

function openScanner(mode) {
  scanner.mode = mode
  scanner.open = true
  // Tüm tarama korumalarını ve durumlarını sıfırla
  detectedOnce = false  // Yeniden açıldığında çoklu taramaya izin vermek için sıfırla
  detectLock = false
  lastScannedCode = ''
  lastScannedBarcode.value = null // Son okunan barkodu temizle
  lastScanAt = 0
  // Yeni açılışta eski reader'a bağlı callback kalmasın
  codeReader = null
  // Add modunda açarken inputu da boşalt (otofill/kalıntı riskini azalt)
  if (mode === 'add') {
    createForm.barcode = ''
  }
  // Bekleme süresi olmasın: baskılama ve başlangıç gecikmesi kaldırıldı
  suppressUntil = 0
  // Kullanıcı etkileşiminde ses altyapısını hazırla (autoplay kısıtları için)
  ensureAudioReady()
  nextTick(async () => {
    await loadCameras()
    // Kamera listesi kontrolü artık loadCameras içinde yapılıyor
    await startScanner()
  })
}

function ensureVideoPlaying() {
  const v = videoEl.value
  if (!v) return
  const info = {
    hasSrcObject: !!v.srcObject,
    readyState: v.readyState,
    paused: v.paused,
    muted: v.muted,
    playsInline: v.playsInline,
  }
  console.debug('[ensureVideoPlaying] video state:', info)
  if (v.paused) {
    v.play().then(() => {
      console.debug('[ensureVideoPlaying] play() resolved')
    }).catch(e => {
      console.debug('[ensureVideoPlaying] play() failed:', e)
    })
  }
}

// Kamera odak/zoom/pozlama ve çözünürlük iyileştirmeleri
function applyCameraEnhancements() {
  try {
    const v = videoEl.value
    const stream = v?.srcObject
    const track = stream?.getVideoTracks?.()[0]
    if (!track) return
    const caps = track.getCapabilities?.() || {}

    const advanced = []
    if (Array.isArray(caps.focusMode) && caps.focusMode.includes('continuous')) {
      advanced.push({ focusMode: 'continuous' })
    }
    if (typeof caps.zoom?.max === 'number') {
      const min = typeof caps.zoom.min === 'number' ? caps.zoom.min : 1
      const target = Math.min(caps.zoom.max, Math.max(min, (caps.zoom.max + min) / 2))
      advanced.push({ zoom: target })
    }
    if (Array.isArray(caps.exposureMode) && caps.exposureMode.includes('continuous')) {
      advanced.push({ exposureMode: 'continuous' })
    }
    if (Array.isArray(caps.whiteBalanceMode) && caps.whiteBalanceMode.includes('continuous')) {
      advanced.push({ whiteBalanceMode: 'continuous' })
    }

    const constraints = {}
    if (caps.frameRate?.max) constraints.frameRate = { ideal: Math.min(30, caps.frameRate.max) }
    if (caps.width?.max && caps.height?.max) {
      constraints.width = { ideal: Math.min(1920, caps.width.max) }
      constraints.height = { ideal: Math.min(1080, caps.height.max) }
    }
    if (advanced.length) constraints.advanced = advanced

    if (Object.keys(constraints).length > 0) {
      track.applyConstraints(constraints)
        .then(() => console.debug('[enhance] applied constraints:', constraints))
        .catch(e => console.debug('[enhance] applyConstraints failed:', e))
    }
  } catch (e) {
    console.debug('[enhance] unexpected error:', e)
  }
}

async function clearBarcodeTracking() {
  // Kamerayı açarken sadece görünürlük ve zaman izlerini temizle
  visibleBarcodes.value.clear()
  lastSeenAt.value.clear()
  // processedBarcodes BİLEREK temizlenmiyor; böylece kamera kapat/aç ile tekrar okuma olmaz
}

async function startScanner() {
  if (scanning) return
  
  // Önceki taramayı temizle
  clearBarcodeTracking()
  sessionStartedAt = Date.now()
  
  // Önceki callback'leri tamamen temizle
  if (codeReader) {
    try {
      codeReader.reset()
    } catch (e) {
      console.debug('codeReader reset ignore:', e)
    }
  }
  // Farklı yön/orientation ve ters renklerde (inverted) daha dayanıklı okuma için ipuçları
  const hints = new Map()
  try {
    hints.set(DecodeHintType.TRY_HARDER, true)
    hints.set(DecodeHintType.ALSO_INVERTED, true)
  } catch (e) {
    console.debug('hints set ignore:', e?.name || e)
  }
  codeReader = new BrowserMultiFormatReader(hints)
  scanning = true
  const callback = (result) => {
    if (result) {
      const code = result.getText()
      onCodeDetected(code)
      // add modunda ilk sonuçtan sonra durdur
      if (scanner.mode === 'add') {
        try { codeReader?.reset?.() } catch (e) { console.debug('codeReader reset failed:', e) }
        stopScanner()
      }
    }
    
    // Periyodik olarak kadrajdan çıkan barkodları zaman aşımı ile temizle
    const now = Date.now()
    const GONE_TIMEOUT = 1200 // ms: bu süre boyunca görülmeyen barkod kadrajdan çıkmış sayılır
    if (now % 300 < 60) { // ~300ms'de bir hafif kontrol
      const expired = []
      lastSeenAt.value.forEach((ts, bc) => {
        if (now - ts > GONE_TIMEOUT) expired.push(bc)
      })
      if (expired.length) {
        expired.forEach(bc => {
          lastSeenAt.value.delete(bc)
          visibleBarcodes.value.delete(bc)
          processedBarcodes.value.delete(bc)
        })
      }
      // Bu oturumda hiç görülmeyen (lastSeenAt'te olmayan) eski processed barkodları da
      // oturum başlangıcından GONE_TIMEOUT sonra temizle (kamera aç-kapatta yeniden okuma olmasın)
      if (sessionStartedAt && (now - sessionStartedAt) > GONE_TIMEOUT) {
        const toDrop = []
        processedBarcodes.value.forEach(bc => {
          if (!lastSeenAt.value.has(bc)) toDrop.push(bc)
        })
        toDrop.forEach(bc => processedBarcodes.value.delete(bc))
      }
    }
  }
  // Yalnızca localhost üzerinde çalışma uyarısı
  const host = window.location.hostname
  const isLocalhost = host === 'localhost' || host === '127.0.0.1'
  if (!isLocalhost) {
    $q.notify({ type: 'warning', message: 'Kamera sadece http://localhost üzerinde çalışacak şekilde yapılandırıldı.' })
    scanning = false
    stopScanner()
    return
  }
  try {
    if (!videoEl.value) {
      console.debug('[startScanner] video element not ready')
      throw new Error('Video elementi hazır değil')
    }
    // Seçilmiş cihaz listede mi kontrol et
    const deviceExists = Array.isArray(cameras.value) && cameras.value.some(c => c.value === selectedCameraId.value)
    const deviceIdToUse = deviceExists ? selectedCameraId.value : null
    // Seçilmiş kamera varsa deviceId, yoksa facingMode ile ilerle
    const constraints = deviceIdToUse
      ? { video: { deviceId: { exact: deviceIdToUse } } }
      : { video: { facingMode: { ideal: 'environment' } } }
    console.debug('[startScanner] using constraints:', constraints)
    await codeReader.decodeFromConstraints(constraints, videoEl.value, callback)
    // izin alındıysa etiketlerin dolması için listeyi tazele
    loadCameras()
    ensureVideoPlaying()
    applyCameraEnhancements()
  } catch (err) {
    console.debug('[startScanner] primary start failed:', err)
    try {
      // Fallback: önce facingMode, o da olmazsa en genel { video: true }
      console.debug('[startScanner] fallback with facingMode environment')
      try {
        await codeReader.decodeFromConstraints(
          { video: { facingMode: { ideal: 'environment' } } },
          videoEl.value,
          callback
        )
      } catch (eFacing) {
        console.debug('[startScanner] facingMode fallback failed, trying { video: true }', eFacing)
        await codeReader.decodeFromConstraints(
          { video: true },
          videoEl.value,
          callback
        )
      }
      loadCameras()
      ensureVideoPlaying()
      applyCameraEnhancements()
    } catch (e2) {
      scanning = false
      console.debug('[startScanner] fallback failed:', e2)
      const msg = e2?.name === 'NotReadableError'
        ? 'Kamera başka bir uygulama tarafından kullanılıyor olabilir.'
        : 'Kamera açılamadı. Lütfen tarayıcıdan kamera izni verin ve sayfayı yenileyin.'
      $q.notify({ type: 'negative', message: msg })
      stopScanner()
    }
  }
}

function stopScanner() {
  try {
    if (codeReader && typeof codeReader.reset === 'function') {
      codeReader.reset()
    }
  } catch (e) {
    console.debug('ZXing reset error:', e)
  }
  try {
    const stream = videoEl.value?.srcObject
    stream?.getTracks?.().forEach(t => t.stop())
  } catch (e) {
    console.debug('stop tracks ignore:', e?.name || e)
  }
  // Video elementini tamamen temizle
  try {
    const v = videoEl.value
    if (v) {
      try { v.pause?.() } catch (e) { console.debug('video pause ignore:', e?.name || e) }
      // Safari/Chrome uyumu için hem property hem attribute temizliği
      v.srcObject = null
      try { v.removeAttribute('srcObject') } catch (e) { console.debug('removeAttribute ignore:', e?.name || e) }
      try { v.load?.() } catch (e) { console.debug('video load ignore:', e?.name || e) }
    }
  } catch (e) {
    console.debug('video clear ignore:', e?.name || e)
  }
  // Güvenlik: kilitleri bırak
  detectLock = false
  detectedOnce = false
  codeReader = null
  scanning = false
}

onBeforeUnmount(() => {
  try { window.removeEventListener('sound-settings-changed', loadSoundSettings) } catch (e) { console.debug('event remove ignore:', e?.name || e) }
  try { window.removeEventListener('add-to-cart', handleExternalAddToCart) } catch (e) { console.debug('event remove ignore:', e?.name || e) }
  stopScanner()
})

// Tarama sıklığını azaltmak için son tarama zamanı
let lastScanTime = 0
const SCAN_INTERVAL = 500 // ms (0.5 saniye)

function onCodeDetected(code) {
  // Scanner kapalıysa ya da aktif tarama yoksa işlem yapma
  if (!scanner.open || !scanning) return
  if (!code) return
  
  const now = Date.now()
  if (now - lastScanTime < SCAN_INTERVAL) return
  lastScanTime = now
  
  if (now < suppressUntil) return
  if (detectLock || detectedOnce) return
  
  const ncode = normalizeCode(code)
  lastScannedCode = ncode // Son okunan barkodu güncelle
  
  // Eğer bu barkod daha önce işlendiyse ve hala kameradaysa tekrar işleme
  if (processedBarcodes.value.has(ncode) && visibleBarcodes.value.has(ncode)) {
    // Yine de son görüldü zamanını tazele
    lastSeenAt.value.set(ncode, now)
    return
  }
  
  // Yeni görünen barkodu işaretle
  visibleBarcodes.value.add(ncode)
  // Son görüldü zamanını güncelle
  lastSeenAt.value.set(ncode, now)
  
  // Eğer bu barkod daha önce işlenmediyse işle
  if (!processedBarcodes.value.has(ncode)) {
    processedBarcodes.value.add(ncode)
  } else {
    return
  }
  
  // 'add' modunda aynı barkodu kısa sürede tekrar tetiklemeyi engelle
  if (scanner.mode === 'add') {
    const nlastSaved = normalizeCode(lastSavedCode)
    const nlastScanned = normalizeCode(lastScannedCode)
    // Kayıt sonrası kısa süreli yok sayma penceresi
    if (sessionIgnore.code && Date.now() < sessionIgnore.until && ncode === sessionIgnore.code) {
      return
    }
    // Son kaydedilen barkodla aynı ise yok say (art arda aynı ürünü yaratmayı önler)
    if (nlastSaved && ncode === nlastSaved) {
      return
    }
    const now = Date.now()
    if (ncode === nlastScanned && (now - lastScanAt) < SCAN_CFG.duplicateWindowMs) {
      return
    }
    lastScannedCode = ncode
    lastScannedBarcode.value = ncode // Son okunan barkodu kaydet
    lastScanAt = now
  } else {
    // cart modunda çoklu okuma var; aynı barkodu çok kısa aralıkta tekrar görürse yok say (farklı barkodlar hemen geçsin)
    const now = Date.now()
    if (ncode === lastScannedCode && (now - lastScanAt) < SCAN_CFG.cartDuplicateWindowMs) {
      return
    }
    lastScannedCode = ncode
    lastScannedBarcode.value = ncode // Son okunan barkodu kaydet
    lastScanAt = now
  }
  
  // Add modunda tek sefer; cart modunda çoklu okuma
  if (scanner.mode === 'add') {
    detectedOnce = true
  }
  
  // Kısa süreli kilit (her iki mod için de) tekrar tetiklemeyi önler
  detectLock = true
  try {
    if (scanner.mode === 'add') {
      // Add modunda taramada bildirim göstermiyoruz; sadece formu doldurup kamerayı kapatıyoruz
      createForm.barcode = ncode
      // Ürün ekle akışında: popup kapat ve kamerayı durdur
      scanner.open = false
      stopScanner()
    } else {
      // Sepet akışında açık kalsın
      handleCartScan(code)
    }
  } finally {
    // add: daha güvenli tek tetik; cart: hızlı tekrar okuma için kısa kilit
    const lockMs = scanner.mode === 'add' ? 1200 : 500
    setTimeout(() => { 
      detectLock = false
      // Add modunda processedBarcodes'ı temizlemiyoruz; sadece görünürlük/zaman izleri
      if (scanner.mode === 'add') {
        visibleBarcodes.value.clear()
        lastSeenAt.value.clear()
      }
    }, lockMs)
    // Sepet modunda siyah ekranı önlemek için oynatmayı zorla
    if (scanner.mode !== 'add') ensureVideoPlaying()
  }
}

// Backend entegrasyonu
// Not: this.$api kullanımı için Options API gerekir; script setup içinde globalProperties erişimi yoktur.
// Bu yüzden import yerine window üzerinden erişeceğiz; Quasar boot/axios, app.config.globalProperties.$api atadı.
function getApi() {
  // Quasar dev sırasında globalProperties doğrudan erişilemez; patern olarak window.__app?.config?.globalProperties?.$api kullanılabilir.
  // Ancak pratikte en temiz yöntem axios instance'ı bu dosyada import etmektir: src/boot/axios.js -> export { api }
  // Oradan import edelim.
  return api
}

import { api } from 'src/boot/axios'

const endpoints = {
  createProduct: '/products',
  getByBarcodePrimary: (bc) => `/products/${encodeURIComponent(bc)}`,
  getByBarcodeAlt1: (bc) => `/products/by-barcode/${encodeURIComponent(bc)}`,
  getByBarcodeAlt2: (bc) => `/products/barcode/${encodeURIComponent(bc)}`,
  getByBarcodeRoot: (bc) => `/${encodeURIComponent(bc)}`,
}

function normalizeCode(v) {
  return String(v ?? '').trim()
}

// Tarama zamanlama ayarları
const SCAN_CFG = {
  warmupAdd: 0,
  warmupCart: 0,
  startDelayAdd: 0,
  duplicateWindowMs: 5000,
  cartDuplicateWindowMs: 800,
  reopenDelayMs: 0
}

async function submitCreate() {
  // Alan başı kırmızı hata göstermemek için manuel doğrulama
  const b = normalizeCode(createForm.barcode)
  const n = normalizeCode(createForm.name)
  const p = Number(createForm.price)
  if (!b || !n || !(p > 0)) {
    $q.notify({ type: 'warning', message: 'Lütfen barkod, ürün adı ve 0’dan büyük fiyat girin.' })
    return
  }
  if (loading.create) return
  loading.create = true
  try {
    const payload = { barcode: b, name: n, price: p }
    await getApi().post(endpoints.createProduct, payload)
    $q.notify({ type: 'positive', message: 'Ürün kaydedildi', position: 'bottom', timeout: 2000 })
    // Barkodla ilişiği tamamen kes: bellekte tutma
    lastSavedCode = ''
    // Kısa süreli olarak aynı kodu yok say (eski barkod kadrajdaysa tekrar tetiklenmesin)
    const savedCode = normalizeCode(payload.barcode)
    sessionIgnore.code = savedCode
    sessionIgnore.until = Date.now() + SCAN_CFG.duplicateWindowMs
    createForm.barcode = ''
    createForm.name = ''
    createForm.price = null
    // Form doğrulama hatalarını da temizle
    createFormRef.value?.resetValidation?.()
    await nextTick()
    // İstek üzerine: Kaydetten sonra kamera popup'ı tekrar açılmasın
  } catch (e) {
    const msg = e?.response?.data?.message || e.message || 'Kayıt başarısız'
    $q.notify({ type: 'negative', message: msg })
  }
  finally {
    loading.create = false
  }
}

async function fetchByBarcodeWithFallback(bc) {
  const apiInst = getApi()
  const paths = [
    endpoints.getByBarcodePrimary(bc),
    endpoints.getByBarcodeAlt1(bc),
    endpoints.getByBarcodeAlt2(bc),
    endpoints.getByBarcodeRoot(bc),
  ]
  let lastErr = null
  for (const p of paths) {
    try {
      const { data } = await apiInst.get(p)
      return data
    } catch (e) {
      lastErr = e
      if (e?.response?.status && e.response.status !== 404) {
        // 404 dışındaki hata geldiğinde hemen bırak
        break
      }
    }
  }
  throw lastErr || new Error('Ürün bulunamadı')
}

async function handleCartScan(barcode) {
  try {
    const data = await fetchByBarcodeWithFallback(barcode)
    const prod = normalizeProduct(data)
    if (!prod) {
      $q.notify({ type: 'warning', message: 'Bu ürün kayıtlı değil' })
      return
    }
    addToCart(prod)
  } catch (e) {
    if (e?.response?.status === 404) {
      $q.notify({ type: 'warning', message: 'Bu ürün kayıtlı değil' })
    } else {
      $q.notify({ type: 'negative', message: e.message || 'Sorgu hatası' })
    }
  }
}

function normalizeProduct(data) {
  // Beklenen alanlar: id, barcode, name, price
  if (!data) return null
  const d = Array.isArray(data) ? data[0] : data
  if (!d) return null
  if (!d.barcode || !d.name || d.price == null) return null
  return { id: d.id ?? null, barcode: String(d.barcode), name: String(d.name), price: Number(d.price) }
}

// Basit beep sesi (Web Audio API)
let audioCtx = null
function ensureAudioReady() {
  try {
    if (!audioCtx) {
      audioCtx = new (window.AudioContext || window.webkitAudioContext)()
    }
    if (audioCtx && audioCtx.state === 'suspended') {
      audioCtx.resume().catch((e) => { console.debug('audioCtx.resume ignore:', e?.name || e) })
    }
  } catch (e) {
    console.debug('ensureAudioReady ignore:', e?.name || e)
  }
}
async function playBeep({ freq = 1000, duration = 200, volume = 0.3 } = {}) {
  try {
    if (!audioCtx) audioCtx = new (window.AudioContext || window.webkitAudioContext)()
    if (audioCtx.state === 'suspended') {
      try { await audioCtx.resume() } catch (e) { console.debug('audioCtx.resume (beep) ignore:', e?.name || e) }
    }
    const effVol = Math.max(0, Math.min(1, (sound.enabled ? sound.volume : 0) * volume))
    if (effVol <= 0) return
    const osc = audioCtx.createOscillator()
    const gain = audioCtx.createGain()
    osc.type = 'square'
    osc.frequency.setValueAtTime(freq, audioCtx.currentTime)
    // kısa bir envelope ile aç/kapa klik sesini azalt
    const now = audioCtx.currentTime
    const attack = 0.005
    const release = 0.04
    gain.gain.setValueAtTime(0, now)
    gain.gain.linearRampToValueAtTime(effVol, now + attack)
    gain.gain.setValueAtTime(effVol, now + (duration / 1000) - release)
    gain.gain.linearRampToValueAtTime(0, now + (duration / 1000))
    osc.connect(gain)
    gain.connect(audioCtx.destination)
    osc.start(now)
    osc.stop(now + duration / 1000)
  } catch {
    // sessizce geç
  }
}

// Satın alma için kısa başarı melodisi (daha yüksek ve belirgin)
function playSuccess() {
  try {
    ensureAudioReady()
    if (!audioCtx) audioCtx = new (window.AudioContext || window.webkitAudioContext)()
    const now = audioCtx.currentTime
    // Master gain (genel ses yüksekliği)
    const master = audioCtx.createGain()
    master.gain.value = 0.9
    master.connect(audioCtx.destination)

    const tones = [
      { f: 880, d: 0.22, v: 0.8, o: 0 },
      { f: 1320, d: 0.28, v: 0.8, o: 0.12 },
      { f: 1760, d: 0.20, v: 0.7, o: 0.28 },
    ]
    tones.forEach(t => {
      const osc = audioCtx.createOscillator()
      const gain = audioCtx.createGain()
      osc.type = 'square'
      osc.frequency.setValueAtTime(t.f, now + t.o)
      // envelope (hızlı attack, kısa release)
      gain.gain.setValueAtTime(0, now + t.o)
      gain.gain.linearRampToValueAtTime(t.v, now + t.o + 0.01)
      gain.gain.linearRampToValueAtTime(0, now + t.o + t.d)
      osc.connect(gain)
      gain.connect(master)
      osc.start(now + t.o)
      osc.stop(now + t.o + t.d)
    })
  } catch (e) {
    console.debug('playSuccess ignore:', e?.name || e)
  }
}

function addToCart(prod, playSound = true) {
  const existing = cart.value.find((x) => x.barcode === prod.barcode)
  if (existing) existing.qty += 1
  else cart.value.push({ ...prod, qty: 1 })
  // Sadece istenirse kısa beep sesi çal
  if (playSound) playBeep()
}

function removeFromCart(barcode) {
  cart.value = cart.value.filter((x) => x.barcode !== barcode)
}

function checkout() {
  // Başarı sesi
  playSuccess()
  cart.value = []
  $q.notify({ type: 'positive', message: 'Satın alma başarılı', position: 'bottom', timeout: 2000 })
}

function currency(v) {
  const n = Number(v || 0)
  return new Intl.NumberFormat('tr-TR', { style: 'currency', currency: 'TRY' }).format(n)
}
</script>

<style scoped></style>
