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
          @submit.prevent="submitCreate"
          ref="createFormRef"
          class="q-gutter-md"
        >
          <div class="row q-col-gutter-md">
            <div class="col-12 col-sm-4">
              <q-input
                v-model="createForm.barcode"
                label="Barkod"
                dense
                outlined
                :rules="[val => !!val || 'Zorunlu']"
              />
            </div>
            <div class="col-12 col-sm-4">
              <q-input
                v-model="createForm.name"
                label="Ürün Adı"
                dense
                outlined
                :rules="[val => !!val || 'Zorunlu']"
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
                :rules="[val => val > 0 || '> 0 olmalı']"
              />
            </div>
          </div>
          <div class="row">
            <div class="col-12">
              <q-btn
                color="positive"
                type="submit"
                label="Kaydet"
                :loading="loading.create"
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
          :rows="cart"
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
              style="width: 100%; height: 100%; object-fit: cover; border-radius: 4px;"
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
            <div class="absolute-top-left q-ma-sm bg-grey-9 text-white q-pa-xs" style="opacity:.8; border-radius:4px;">Kamerayı barkoda doğrultun</div>
          </div>
        </q-card-section>
      </q-card>
    </q-dialog>
  </q-page>
</template>

<script setup>
import { ref, reactive, computed, onBeforeUnmount, nextTick } from 'vue'
import { useQuasar } from 'quasar'
import { BrowserMultiFormatReader } from '@zxing/browser'

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

// Sepet
const cart = ref([]) // { id, barcode, name, price, qty }
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
let codeReader = null
let detectLock = false

// Kamera listesi ve seçim
const cameras = ref([]) // [{ label, value }]
const selectedCameraId = ref(null)

async function loadCameras() {
  try {
    const host = window.location.hostname
    const isLocalhost = host === 'localhost' || host === '127.0.0.1'
    console.debug('[loadCameras] host:', host, 'isLocalhost:', isLocalhost, 'secure:', window.isSecureContext)

    // 1) İzin durumu (destekleniyorsa) kaydı
    try {
      const perm = await (navigator.permissions?.query?.({ name: 'camera' }))
      if (perm) console.debug('[loadCameras] permission state:', perm.state)
    } catch (e) {
      console.debug('[loadCameras] permissions API not available or failed:', e)
    }

    // 2) Cihazları topla
    let devicesAll = await navigator.mediaDevices.enumerateDevices()
    let devices = devicesAll.filter(d => d.kind === 'videoinput')
    let hasAny = devices.length > 0
    let hasLabels = hasAny && devices.some(d => d.label && d.label.trim().length > 0)
    console.debug('[loadCameras] videoinput count:', devices.length, 'hasLabels:', hasLabels)

    // 3) Label yoksa bir defa izin tetikle
    if (!hasAny || !hasLabels) {
      try {
        const tmp = await navigator.mediaDevices.getUserMedia({ video: true })
        tmp.getTracks().forEach(t => t.stop())
        devicesAll = await navigator.mediaDevices.enumerateDevices()
        devices = devicesAll.filter(d => d.kind === 'videoinput')
        hasAny = devices.length > 0
        hasLabels = hasAny && devices.some(d => d.label && d.label.trim().length > 0)
        console.debug('[loadCameras] after permission, videoinput count:', devices.length, 'hasLabels:', hasLabels)
      } catch (permErr) {
        console.debug('Kamera izni alınamadı:', permErr)
        $q.notify({ type: 'warning', message: 'Kamera erişimine izin veriniz' })
      }
    }

    cameras.value = devices.map((d, i) => ({
      label: d.label || `Kamera ${i + 1}`,
      value: d.deviceId,
    }))

    if (!selectedCameraId.value && cameras.value.length) {
      // Arka kamerayı tercih et
      const back = devices.find(d => /back|rear|environment/i.test(d.label))?.deviceId
      selectedCameraId.value = back || cameras.value[0].value
    }

    if (cameras.value.length === 0) {
      const msg = isLocalhost
        ? 'Kamera bulunamadı. Tarayıcı izinlerini kontrol edin.'
        : 'Güvenli olmayan kaynak. Kamerayı yalnızca http://localhost üzerinde deneyin.'
      $q.notify({ type: 'warning', message: msg })
    }
  } catch (e) {
    console.debug('Kamera listesi alınamadı:', e)
  }
}

// Kamera seçicisi kaldırıldı; otomatik kamera seçimi kullanılacak

function openScanner(mode) {
  scanner.mode = mode
  scanner.open = true
  nextTick(async () => {
    await loadCameras()
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

async function startScanner() {
  if (scanning) return
  codeReader = new BrowserMultiFormatReader()
  scanning = true
  const callback = (result) => {
    if (result) {
      const code = result.getText()
      onCodeDetected(code)
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
    if (selectedCameraId.value) {
      console.debug('[startScanner] starting with deviceId:', selectedCameraId.value)
      await codeReader.decodeFromVideoDevice(selectedCameraId.value, videoEl.value, callback)
      ensureVideoPlaying()
      applyCameraEnhancements()
    } else {
      // Önce arka kamerayı hedefleyen facingMode ile dene
      console.debug('[startScanner] starting with facingMode: environment')
      await codeReader.decodeFromConstraints(
        { video: { facingMode: { ideal: 'environment' } } },
        videoEl.value,
        callback
      )
      // izin alındıysa etiketlerin dolması için listeyi tazele
      loadCameras()
      ensureVideoPlaying()
      applyCameraEnhancements()
    }
  } catch (err) {
    console.debug('[startScanner] primary start failed:', err)
    try {
      // Fallback: cihaz listesinden arka kamerayı bul
      const devices = await BrowserMultiFormatReader.listVideoInputDevices()
      if (!devices || devices.length === 0) {
        throw new Error('Kamera bulunamadı')
      }
      let deviceId = selectedCameraId.value
      if (!deviceId) deviceId = devices.find(d => /back|rear|environment/i.test(d.label))?.deviceId
      if (!deviceId) deviceId = devices[devices.length - 1].deviceId
      selectedCameraId.value = deviceId
      console.debug('[startScanner] fallback starting with deviceId:', deviceId)
      await codeReader.decodeFromVideoDevice(deviceId, videoEl.value, callback)
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
    // kamera/reset esnasında hata oluşsa bile akışı bozmayalım
    console.debug('ZXing reset error:', e)
  }
  try {
    const v = videoEl.value
    const stream = v?.srcObject
    const tracks = stream?.getTracks?.() || []
    tracks.forEach(t => {
      try { t.stop() } catch (e) { console.debug('track.stop() ignore:', e?.name || e) }
    })
    if (v) v.srcObject = null
  } catch (e) {
    console.debug('stop tracks ignore:', e?.name || e)
  }
  scanning = false
}

onBeforeUnmount(stopScanner)

function onCodeDetected(code) {
  if (!code) return
  if (detectLock) return
  detectLock = true
  try {
    if (scanner.mode === 'add') {
      createForm.barcode = code
      $q.notify({ type: 'positive', message: `Barkod okundu: ${code}` })
      // Ürün ekle akışında: popup kapat ve kamerayı durdur
      scanner.open = false
      stopScanner()
    } else {
      // Sepet akışında açık kalsın
      handleCartScan(code)
    }
  } finally {
    // 1-1.5 sn sonra yeni okumalara izin ver
    setTimeout(() => { detectLock = false }, 1200)
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
  getByBarcode: (bc) => `/products/${encodeURIComponent(bc)}`,
}

async function submitCreate() {
  // QForm.validate yoksa gönderimi engelleme
  let valid = true
  const formRef = createFormRef.value
  if (formRef?.validate) {
    valid = await formRef.validate()
  }
  if (!valid) return
  loading.create = true
  try {
    const payload = { barcode: createForm.barcode, name: createForm.name, price: Number(createForm.price) }
    await getApi().post(endpoints.createProduct, payload)
    $q.notify({ type: 'positive', message: 'Ürün kaydedildi', position: 'bottom', timeout: 2000 })
    createForm.barcode = ''
    createForm.name = ''
    createForm.price = null
    // Form doğrulama hatalarını da temizle
    createFormRef.value?.resetValidation?.()
  } catch (e) {
    const msg = e?.response?.data?.message || e.message || 'Kayıt başarısız'
    $q.notify({ type: 'negative', message: msg })
  }
  finally {
    loading.create = false
  }
}

async function handleCartScan(barcode) {
  try {
    const { data } = await getApi().get(endpoints.getByBarcode(barcode))
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

function addToCart(prod) {
  const existing = cart.value.find((x) => x.barcode === prod.barcode)
  if (existing) existing.qty += 1
  else cart.value.push({ ...prod, qty: 1 })
}

function removeFromCart(barcode) {
  cart.value = cart.value.filter((x) => x.barcode !== barcode)
}

function checkout() {
  $q.dialog({ title: 'Satın Alma', message: 'Satın alma başarılı', ok: 'Tamam' })
  cart.value = []
}

function currency(v) {
  const n = Number(v || 0)
  return new Intl.NumberFormat('tr-TR', { style: 'currency', currency: 'TRY' }).format(n)
}
</script>

<style scoped></style>
