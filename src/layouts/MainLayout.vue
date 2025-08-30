<template>
  <q-layout view="lHh Lpr lFf">
    <q-header elevated>
      <q-toolbar class="bg-primary text-white">
        <q-btn
          dense
          flat
          round
          icon="menu"
          @click="drawerOpen = !drawerOpen"
        />
        <q-toolbar-title>Online Kasa</q-toolbar-title>
        <q-space />
        <!-- Ses Ayarları (sağ üst, beyaz) -->
        <div class="row items-center q-gutter-xs text-white">
          <q-toggle
            v-model="soundEnabled"
            label="Ses"
            dense
            dark
            color="white"
            keep-color
            @update:model-value="persistSound"
          />
          <q-slider
            v-model="soundVolume"
            :min="0"
            :max="1"
            :step="0.05"
            dense
            dark
            color="white"
            thumb-color="white"
            track-color="white"
            style="width: 140px"
            @update:model-value="persistSound"
          />
        </div>
      </q-toolbar>
    </q-header>

    <!-- Sol Çekmece: açılışta kapalı; genişlik artırıldı -->
    <q-drawer
      v-model="drawerOpen"
      side="left"
      bordered
      :width="400"
    >
      <product-drawer />
    </q-drawer>

    <q-page-container>
      <router-view />
    </q-page-container>
  </q-layout>

</template>

<script setup>
import { ref, onMounted, watch } from 'vue'
import ProductDrawer from 'src/components/ProductDrawer.vue'

const drawerOpen = ref(false)
const soundEnabled = ref(true)
const soundVolume = ref(0.7)

function loadSound() {
  try {
    const en = localStorage.getItem('sound.enabled')
    const vol = localStorage.getItem('sound.volume')
    if (en != null) soundEnabled.value = en === 'true'
    if (vol != null) {
      const v = Number(vol)
      if (!Number.isNaN(v)) soundVolume.value = Math.min(1, Math.max(0, v))
    }
  } catch (e) { console.debug('loadSound ignore:', e?.name || e) }
}

function persistSound() {
  try {
    localStorage.setItem('sound.enabled', String(soundEnabled.value))
    localStorage.setItem('sound.volume', String(soundVolume.value))
    window.dispatchEvent(new CustomEvent('sound-settings-changed'))
  } catch (e) { console.debug('persistSound ignore:', e?.name || e) }
}

onMounted(loadSound)
watch([soundEnabled, soundVolume], persistSound)
</script>
