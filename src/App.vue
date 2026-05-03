<template>
  <div id="app" class="container">
    <h2>Foreigner Character Importer for Lotte World Adventure</h2>

    <div class="form-grid">
      <input v-model="name" type="text" maxlength="12" placeholder="Enter IGN" />

      <select v-model="region">
        <option v-for="r in regions" :key="r.value" :value="r.value">
          {{ r.label }}
        </option>
      </select>

      <button @click="searchCharacter">Search</button>
    </div>

    <div v-if="!(imageUrl && regionImageUrl)">
      <p><h4>How to use this page?</h4></p>
      <p><ul>
        <li>Input the character name of the region you're currently playing at.</li>
        <li>Select the game region. CMS and MSN are not supported due to technical limitations.</li>
        <li>Tap Search, and check if your character looks as you would expect on large screen.</li>
        <li>If yes, tap "Generate QR", walk to the QR scanner under the large screen and scan it.</li>
        <li>That's all.</li>
      </ul></p>
      <p>Character doesn't look right? That's because you're using gears, face, hair that unavailable in KMS.</p>
    </div>

    <div v-if="error" class="error">{{ error }}</div>
    <div v-if="loading" class="spinner"></div>

    <table v-if="imageUrl && regionImageUrl" class="image-table">
      <tbody>
        <tr>
          <td class="image-cell">
            <img :src="imageUrl" alt="Your avatar in Lotte World" />
          </td>
          <td class="image-cell">
            <img :src="regionImageUrl" alt="Your avatar in game (for reference)" />
          </td>
        </tr>
        <tr>
          <td class="caption-cell">Your avatar in Lotte World</td>
          <td class="caption-cell">Your avatar in game (for reference)</td>
        </tr>
      </tbody>
    </table>
    <!-- Confirm button -->
    <div v-if="warning" class="warning">{{ warning }}</div>
    <div v-if="!loading && imageUrl && regionImageUrl && !confirmed" class="actions">
      Is previewed image OK for push? If yes, please click Generate QR code.<br />You may change your name in the text
      box before generating the QR code.
      <button class="full-width-btn" @click="confirmImages">Generate QR</button>
    </div>

    <!-- QR code after confirmation -->
    <div v-if="confirmed" class="qr-section">
      <qrcode-vue :value="qrValue" :size="qrSize" />
    </div>
    <div v-if="confirmed" class="regenerate_button">
      <br /><button class="full-width-btn" @click="confirmImages">Regenerate QR</button>
    </div>
  </div>
</template>

<script setup>
import { ref, onMounted, onBeforeUnmount } from 'vue'
import QrcodeVue from 'qrcode.vue'


const qrValue = ref('')
const qrSize = ref(800) // default
const name = ref('')
const region = ref('')
const imageUrl = ref('')
const regionImageUrl = ref('')
const error = ref('')
const loading = ref(false)
const confirmed = ref(false)
const warning = ref('')
const avatarCode = ref('')

// Define regions with display label, actual value, and region-specific prefix
const regions = [
  { label: 'GMS North America', value: 'InkwellMS_NA', prefix: 'https://msavatar1.nexon.net/Character/', suffix: '.png' },
  { label: 'GMS Europe', value: 'InkwellMS_EU', prefix: 'https://msavatar1.nexon.net/Character/', suffix: '.png' },
  { label: 'KMS', value: 'ChangseopMS', prefix: 'https://open.api.nexon.com/static/maplestory/character/look/', suffix: '?.png' },
  { label: 'JMS', value: 'ZipanguMS', prefix: 'https://avatar-maplestory.nexon.co.jp/Character/', suffix: '.png' },
  { label: 'TMS', value: 'TerryMS_Island', prefix: 'https://open.api.nexon.com/static/maplestorytw/character/look/', suffix: '?.png' },
  { label: 'MSEA', value: 'TerryMS_Peninsula', prefix: 'https://open.api.nexon.com/static/maplestorysea/character/look/', suffix: '?.png' }
]

region.value = regions[0].value

async function searchCharacter() {
  error.value = ''
  imageUrl.value = ''
  regionImageUrl.value = ''
  confirmed.value = false
  qrValue.value = ''
  loading.value = true

  if (!name.value.trim() || !region.value.trim()) {
    error.value = 'Please enter valid IGN.'
    return
  }

  try {
    const response = await fetch(import.meta.env.VITE_API_URL, {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({
        ign: name.value,
        region: region.value
      })
    })

    const data = await response.json()

    if (data.status === 'Ok' && data.avatarCode) {
      // First image: global prefix
      imageUrl.value = `${import.meta.env.VITE_IMG_URL}${data.avatarCode}`

      // Second image: region-specific prefix
      const selected = regions.find(r => r.value === region.value)
      if (selected) {
        regionImageUrl.value = `${selected.prefix}${data.avatarCode}${selected.suffix}`
      }
      if (region.value === 'InkwellMS_NA' || region.value === 'InkwellMS_EU') {
        warning.value = '⚠️ Notice: If you import character from GMS, your IGN cannot be displayed in the huge screen due to limitation.'
      }
      avatarCode.value = data.avatarCode
    } else {
      error.value = 'Character not found'
    }
  } catch (err) {
    error.value = 'Error fetching data'
  } finally {
    loading.value = false
  }
}

function confirmImages() {
  confirmed.value = true
  const encodedName = encodeURIComponent(name.value)
  qrValue.value = `01/${avatarCode.value}|${encodedName}|0|0|1|1`
}

function updateQrSize() {
  // Fit to container width (max 100%)
  const container = document.querySelector('.qr-section')
  if (container) {
    qrSize.value = container.offsetWidth
  }
}

onMounted(() => {
  updateQrSize()
  window.addEventListener('resize', updateQrSize)
})

onBeforeUnmount(() => {
  window.removeEventListener('resize', updateQrSize)
})
</script>

<style>
.container {
  font-family: sans-serif;
  padding: 1em;
  max-width: 600px;
  margin: auto;
}

#app {
  background-image: url('back.jpg');
  background-size: cover;
  /* scale to fill */
  background-position: center;
  /* center the image */
  background-repeat: no-repeat;
  /* avoid tiling */
  min-height: 100vh;
  /* full viewport height */
}

.form-grid {
  display: grid;
  grid-template-columns: 1fr;
  gap: 0.5em;
}

.form-grid input,
.form-grid select,
.form-grid button {
  width: 100%;
  padding: 0.5em;
  box-sizing: border-box;
}

@media (min-width: 768px) {
  .form-grid {
    grid-template-columns: 2fr 1fr auto;
    align-items: center;
  }
}

.images {
  display: flex;
  justify-content: center;
  /* center horizontally */
  align-items: center;
  /* align vertically */
  gap: 1em;
  height: 100vh;
  /* full viewport height for vertical centering */
}

.images img {
  image-rendering: pixelated;
  /* crisp edges */
  transform-origin: center;
  /* scale outward from center */
}

.scaled {
  /* integer scaling */
  transform: scale(2);
  /* scale by integer factor */
  transform-origin: center;
  image-rendering: pixelated;
  /* keep crisp edges */
}

.image-table {
  border-collapse: collapse;
  margin: auto;
  /* center the table itself */
}

.image-cell {
  text-align: center;
  /* center content inside each cell */
  vertical-align: middle;
  /* vertically align images */
  padding: 0.5em;
}

.image-cell img {
  display: block;
  margin: auto;
  image-rendering: pixelated;
  /* crisp scaling */
}

.scaled img {
  display: block;
}

.error {
  color: red;
  margin-top: 1em;
}

.spinner {
  margin: 1em auto;
  border: 4px solid #f3f3f3;
  border-top: 4px solid #3498db;
  border-radius: 50%;
  width: 40px;
  height: 40px;
  animation: spin 1s linear infinite;
}

.qr-section {
  margin-top: 1em;
  display: flex;
  justify-content: center;
  width: 100%;
  /* full width */
}

.full-width-btn {
  display: block;
  /* ensures it behaves like a block element */
  width: 100%;
  /* fills the parent container */
  padding: 0.75em;
  /* larger touch-friendly area */
  font-size: 1em;
  /* readable text */
  background-color: #3498db;
  color: white;
  border: none;
  border-radius: 4px;
  cursor: pointer;
}

.full-width-btn:hover {
  background-color: #2980b9;
}


@keyframes spin {
  0% {
    transform: rotate(0deg);
  }

  100% {
    transform: rotate(360deg);
  }
}
</style>
