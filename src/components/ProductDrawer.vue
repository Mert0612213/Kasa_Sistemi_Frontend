<template>
  <div class="q-pa-sm column q-gutter-sm">
    <div class="row items-center q-gutter-sm">
      <div class="text-subtitle1">Ürünler</div>
      <q-space />
      <q-input v-model="search" dense outlined clearable debounce="200" placeholder="Ara..." style="min-width: 160px" />
    </div>

    <q-separator />

    <q-list bordered separator class="rounded-borders">
      <template v-if="loading && items.length === 0">
        <div class="q-pa-md text-grey">Yükleniyor...</div>
      </template>
      <template v-else-if="filtered.length === 0">
        <div class="q-pa-md text-grey">Kayıt bulunamadı</div>
      </template>
      <template v-else>
        <q-item v-for="p in filtered" :key="p.id ?? p.barcode" clickable>
          <q-item-section>
            <q-item-label>{{ p.name }}</q-item-label>
            <q-item-label caption>
              {{ p.barcode }} • {{ currency(p.price) }}
            </q-item-label>
          </q-item-section>
          <q-item-section side class="row items-center q-gutter-xs">
            <q-btn flat round dense color="positive" icon="add_shopping_cart" @click.stop="emitAddToCart(p)" />
            <q-btn flat round dense color="secondary" icon="edit" @click.stop="openEdit(p)" />
            <q-btn flat round dense color="negative" icon="delete" :loading="deletingKey === (p.id ?? p.barcode)" @click.stop="confirmDelete(p)" />
          </q-item-section>
        </q-item>
      </template>
    </q-list>

    <!-- Düzenle Dialog -->
    <q-dialog v-model="edit.open">
      <q-card style="min-width: 360px; max-width: 92vw;">
        <q-card-section class="text-h6">Ürünü Düzenle</q-card-section>
        <q-separator />
        <q-card-section class="q-gutter-md">
          <q-input v-model="edit.form.name" label="Ürün Adı" dense outlined />
          <q-input v-model.number="edit.form.price" type="number" step="0.01" label="Fiyat" dense outlined />
          <q-input v-model="edit.form.barcode" label="Barkod" dense outlined />
        </q-card-section>
        <q-card-actions align="right">
          <q-btn flat label="Vazgeç" v-close-popup :disable="edit.saving" />
          <q-btn color="primary" label="Kaydet" :loading="edit.saving" :disable="edit.saving" @click="saveEdit" />
        </q-card-actions>
      </q-card>
    </q-dialog>
  </div>
</template>

<script setup>
import { ref, reactive, computed, onMounted } from 'vue'
import { useQuasar } from 'quasar'
import { api } from 'src/boot/axios'

const $q = useQuasar()
const loading = ref(false)
const items = ref([]) // {id, name, price, barcode}
const search = ref('')

const filtered = computed(() => {
  const s = search.value.trim().toLowerCase()
  if (!s) return items.value
  return items.value.filter(p =>
    (p.name || '').toLowerCase().includes(s) ||
    String(p.barcode || '').toLowerCase().includes(s)
  )
})

function currency(v) {
  const n = Number(v || 0)
  return new Intl.NumberFormat('tr-TR', { style: 'currency', currency: 'TRY' }).format(n)
}

async function loadAll() {
  loading.value = true
  try {
    const { data } = await api.get('/products')
    items.value = Array.isArray(data) ? data.map(n => normalize(n)) : []
  } catch (e) {
    console.debug('loadAll failed:', e)
    $q.notify({ type: 'negative', message: e?.response?.data?.message || e?.message || 'Ürünler alınamadı' })
  } finally {
    loading.value = false
  }
}

function normalize(d) {
  if (!d) return null
  return { id: d.id ?? null, name: String(d.name ?? ''), price: Number(d.price ?? 0), barcode: String(d.barcode ?? '') }
}

const edit = reactive({ open: false, saving: false, form: { id: null, name: '', price: null, barcode: '', originalBarcode: '' } })
function openEdit(p) {
  edit.form.id = p.id ?? null
  edit.form.name = p.name
  edit.form.price = Number(p.price || 0)
  edit.form.barcode = p.barcode
  edit.form.originalBarcode = p.barcode
  edit.open = true
}

async function saveEdit() {
  const f = edit.form
  if (!f.name || !(Number(f.price) > 0) || !f.barcode) {
    $q.notify({ type: 'warning', message: 'Lütfen barkod, ad ve fiyat giriniz.' })
    return
  }
  edit.saving = true
  try {
    // Laravel route: PUT /products/{barcode}
    const pathBarcode = encodeURIComponent(f.originalBarcode || f.barcode)
    await api.put(`/products/${pathBarcode}`, { name: f.name, price: f.price, barcode: f.barcode })
    $q.notify({ type: 'positive', message: 'Ürün güncellendi' })
    edit.open = false
    await loadAll()
  } catch (e) {
    $q.notify({ type: 'negative', message: e?.response?.data?.message || e?.message || 'Güncelleme başarısız' })
  } finally {
    edit.saving = false
  }
}

const deletingKey = ref(null)
function confirmDelete(p) {
  $q.dialog({
    title: 'Sil',
    message: `${p.name} adlı ürünü silmek istiyor musunuz?`,
    ok: { label: 'Sil', color: 'negative' },
    cancel: { label: 'Vazgeç' }
  }).onOk(async () => {
    // Optimistic removal: önce listeden çıkar, sonra API çağır
    const key = (x) => (x.id ?? x.barcode)
    const prev = items.value.slice()
    items.value = items.value.filter(x => key(x) !== key(p))
    try {
      deletingKey.value = key(p)
      if (p.barcode) {
        // Laravel route: DELETE /products/{barcode} (backend'e eklenmeli)
        await api.delete(`/products/${encodeURIComponent(p.barcode)}`)
        $q.notify({ type: 'positive', message: 'Ürün silindi' })
      } else {
        items.value = prev
        $q.notify({ type: 'warning', message: 'Silmek için yeterli bilgi yok.' })
      }
    } catch (e) {
      // Hata: geri al
      items.value = prev
      $q.notify({ type: 'negative', message: e?.response?.data?.message || e?.message || 'Silme başarısız' })
    }
    finally {
      deletingKey.value = null
    }
  })
}

onMounted(loadAll)

function emitAddToCart(p) {
  try {
    const payload = { id: p.id ?? null, name: p.name, barcode: p.barcode, price: Number(p.price || 0) }
    window.dispatchEvent(new CustomEvent('add-to-cart', { detail: payload }))
  } catch (e) {
    console.debug('emitAddToCart ignore:', e?.name || e)
  }
}
</script>

<style scoped>
</style>
