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
      maximized
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
          <div style="position: relative; width: 100%; height: calc(100vh - 48px);">
            <video
              ref="videoEl"
              autoplay
              playsinline
              muted
              style="width: 100%; height: 100%; object-fit: cover;"
            ></video>
            <div
              class="absolute-top-left q-ma-sm bg-grey-9 text-white q-pa-xs"
              style="opacity:.8; border-radius:4px;"
            >
              Kamerayı barkoda doğrultun</div>
            <div
              class="absolute-top-right q-ma-sm bg-grey-9 text-white q-pa-xs"
              style="opacity:.9; border-radius:4px; min-width: 260px;"
            >
              <div class="row items-center q-gutter-xs">
                <div class="col">
                  <q-select
                    dense
                    filled
                    options-dense
                    emit-value
                    map-options
                    v-model="selectedCameraId"
                    :options="cameras"
                    option-label="label"
                    option-value="value"
                    label="Kamera Seç"
                    @update:model-value="val => switchCamera(val)"
                  />
                </div>
                <div class="col-auto">
                  <q-btn
                    dense
                    flat
                    round
                    icon="refresh"
                    @click="loadCameras"
                  />
                </div>
              </div>
            </div>
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

async function switchCamera(deviceId) {
  // seçilen kamerayı ata
  selectedCameraId.value = deviceId
  try {
    // mevcut taramayı durdurup seçilen cihazla tekrar başlat
    console.debug('[switchCamera] switching to deviceId:', deviceId)
    codeReader?.reset()
  } catch (e) {
    console.debug('[switchCamera] Scanner reset error:', e)
  }
  scanning = false
  try {
    await startScanner()
  } catch (e) {
    console.debug('[switchCamera] startScanner error:', e)
    $q.notify({ type: 'negative', message: 'Kamera değiştirilemedi. Başka bir cihaz deneyin.' })
  }
}

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
    codeReader?.reset()
  } catch (e) {
    // kamera/reset esnasında hata oluşsa bile akışı bozmayalım
    console.debug('ZXing reset error:', e)
  }
  scanning = false
}

onBeforeUnmount(stopScanner)

function onCodeDetected(code) {
  if (!code) return
  stopScanner()
  if (scanner.mode === 'add') {
    createForm.barcode = code
    $q.notify({ type: 'positive', message: `Barkod okundu: ${code}` })
  } else {
    handleCartScan(code)
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
  const valid = await createFormRef.value?.validate?.()
  if (!valid) return
  loading.create = true
  try {
    const payload = { barcode: createForm.barcode, name: createForm.name, price: Number(createForm.price) }
    await getApi().post(endpoints.createProduct, payload)
    $q.notify({ type: 'positive', message: 'Ürün kaydedildi' })
    createForm.barcode = ''
    createForm.name = ''
    createForm.price = null
  } catch (e) {
    const msg = e?.response?.data?.message || e.message || 'Kayıt başarısız'
    $q.notify({ type: 'negative', message: msg })
  } finally {
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
