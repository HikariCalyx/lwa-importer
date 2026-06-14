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
      <p>
      <h4>How to use this page?</h4>
      </p>
      <p>
      <ul>
        <li>Input the character name of the region you're currently playing at.</li>
        <li>Select the game region. </li>
        <li>Tap Search, and check if your character looks as you would expect on large screen.</li>
        <li>If yes, tap "Generate QR", walk to the QR scanner under the big screen and scan it.</li>
        <li>That's all.</li>
      </ul>
      </p>
      <div v-if="isZhCN">
        <p>
        <h4><a href="https://qm.qq.com/q/mFVMcFf4xq" target="_blank">如果你是正在游玩 CMS
            (中国大陆版冒险岛 Online) 的玩家，请加入我们的QQ群：335668600，了解如何将你在 CMS 的角色制作成二维码推送上乐天世界大屏幕。</a></h4>
        </p>
      </div>
    </div>

    <div v-if="error" class="error">{{ error }}</div>
    <div v-if="loading" class="spinner"></div>

    <table v-if="imageUrl && regionImageUrl" class="image-table">
      <tbody>
        <tr>
          <td class="image-cell">
            <img :src="imageUrl" alt="Your avatar in Lotte World" />
            <div class="caption-cell">Your avatar in Lotte World</div>
          </td>
          <td class="image-cell">
            <img :src="regionImageUrl" alt="Your avatar in game (for reference)" />
            <div class="caption-cell">Your avatar in game (for reference)</div>
          </td>
        </tr>
      </tbody>
    </table>

    <!-- Unavailable Gear Table -->
    <div v-if="kmsUnavailableGear && Object.keys(kmsUnavailableGear).length > 0" class="image-table">
      <h3>{{ table_title }}</h3>
      <table class="gear-table">
        <thead>
          <tr>
            <th>{{ table_col_icon }}</th>
            <th>{{ table_col_name }}</th>
          </tr>
        </thead>
        <tbody>
          <tr v-for="(gearName, gearId) in kmsUnavailableGear" :key="gearId">
            <td class="gear-image-cell">
              <img :src="getGearImageUrl(gearId)" :alt="gearName" class="gear-icon" />
            </td>
            <td>{{ gearName }}</td>
          </tr>
        </tbody>
      </table>
    </div>
    <!-- Confirm button -->
    <div v-if="warning" class="warning">{{ warning }}</div>
    <div v-if="!loading && imageUrl && regionImageUrl && !confirmed" class="actions">
      <p>{{ confirmMessage }}</p>
      <div v-html="jmsConfirmMsg"></div>
      <p>If you think it's still OK to push, please click Generate QR code.<br />You may change your name in the text
        box before generating the QR code.</p>
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
import { ref, onMounted, onBeforeUnmount, nextTick } from 'vue'
import QrcodeVue from 'qrcode.vue'

const isZhCN = ref(false)
const qrValue = ref('')
const qrSize = ref(400) // default
const name = ref('')
const region = ref('')
const imageUrl = ref('')
const regionImageUrl = ref('')
const error = ref('')
const loading = ref(false)
const confirmed = ref(false)
const warning = ref('')
const avatarCode = ref('')
const confirmMessage = ref('')
const jmsConfirmMsg = ref('')
const table_title = ref('')
const table_col_icon = ref('')
const table_col_name = ref('')
const kmsUnavailableGear = ref({})

// Define regions with display label, actual value, and region-specific prefix
const regions = [
  { label: 'GMS North America', value: 'InkwellMS_NA', prefix: 'https://msavatar1.nexon.net/Character/', suffix: '.png' },
  { label: 'GMS Europe', value: 'InkwellMS_EU', prefix: 'https://msavatar1.nexon.net/Character/', suffix: '.png' },
  { label: 'KMS', value: 'ChangseopMS', prefix: 'https://open.api.nexon.com/static/maplestory/character/look/', suffix: '?.png' },
  { label: 'JMS', value: 'ZipanguMS', prefix: 'https://avatar-maplestory.nexon.co.jp/Character/', suffix: '.png' },
  { label: 'CMS', value: 'TerryMS_Continent', prefix: 'https://open.api.nexon.com/static/maplestorycn/character/look/', suffix: '?.png' },
  { label: 'TMS', value: 'TerryMS_Island', prefix: 'https://open.api.nexon.com/static/maplestorytw/character/look/', suffix: '?.png' },
  { label: 'MSEA', value: 'TerryMS_Peninsula', prefix: 'https://open.api.nexon.com/static/maplestorysea/character/look/', suffix: '?.png' },
  { label: 'MSN', value: 'TunerMS', prefix: 'https://gamedatahub-static.msu.io/msu/platform/charimages/static/', suffix: '.png' }
]

region.value = regions[0].value

onMounted(() => {
  const lang = navigator.language || navigator.userLanguage
  isZhCN.value = lang.toLowerCase().startsWith('zh-cn')
})

async function searchCharacter() {
  error.value = ''
  imageUrl.value = ''
  regionImageUrl.value = ''
  confirmed.value = false
  qrValue.value = ''
  loading.value = true
  warning.value = ''
  confirmMessage.value = ''
  jmsConfirmMsg.value = ''
  kmsUnavailableGear.value = {}

  if (!name.value.trim() || !region.value.trim()) {
    error.value = 'Please enter valid IGN.'
    loading.value = false
    return
  }

  if (containsChinese(name.value)) {
    if (region.value !== "ZipanguMS" && region.value !== "TerryMS_Island" && region.value !== "TerryMS_Continent") {
      error.value = "This region does not allow IGN contains Hanzi/Kanji. Maybe you're looking for character on JMS/TMS/CMS?"
      loading.value = false
      return
    }
  }

  if (containsKana(name.value)) {
    if (region.value !== "ZipanguMS") {
      error.value = "This region does not allow IGN contains Kana. Maybe you're looking for character on JMS?"
      loading.value = false
      return
    }
  }

  if (containsHangul(name.value)) {
    if (region.value !== "ChangseopMS") {
      error.value = "This region does not allow IGN contains Hangul. Maybe you're looking for character on KMS?"
      loading.value = false
      return
    }
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
        if (region.value === 'TunerMS') {
          regionImageUrl.value = `${selected.prefix}${data.msnAvatarCode}${selected.suffix}`
        } else {
          regionImageUrl.value = `${selected.prefix}${data.avatarCode}${selected.suffix}`
        }
      }

      // Capture unavailable gear data
      if (data.kmsUnavailableGear) {
        kmsUnavailableGear.value = data.kmsUnavailableGear
      }
      if (region.value === 'InkwellMS_NA' || region.value === 'InkwellMS_EU' || region.value === 'TunerMS') {
        warning.value = '⚠️ Notice: If you import character from this region, your IGN cannot be displayed on the huge screen due to limitation.'
        confirmMessage.value = 'Character doesn\'t look right ? That\'s because you\'re using gears, face, hair that unavailable in KMS.'
        jmsConfirmMsg.value = ''
        table_title.value = 'Unavailable Gears in KMS'
        table_col_icon = 'Icon'
        table_col_name = 'Name'
      } else if (region.value === 'TerryMS_Island') {
        warning.value = '⚠️ 注意：如果你的角色名包含中文，可能無法推送。請在上方的文本框修改角色名後生成QR。'
        confirmMessage.value = '角色看起來不對勁？這是因為你的角色正在使用韓版沒有的時裝/裝備/髮型/臉型/皮膚。'
        jmsConfirmMsg.value = ''
        table_title.value = '韓版沒有的裝備'
        table_col_icon = '圖示'
        table_col_name = '名稱'
      } else if (region.value === 'TerryMS_Continent') {
        warning.value = '⚠️ 注意：如果你的角色名包含中文，可能无法推送。请在上方的文本框修改角色名后，生成二维码。'
        confirmMessage.value = '角色看起来不对劲？这是因为你的角色正在使用韩服没有的装备/点装/发型/脸型/皮肤。'
        jmsConfirmMsg.value = ''
        table_title.value = '韩服没有的装备'
        table_col_icon = '图标'
        table_col_name = '名称'
      } else if (region.value === 'ZipanguMS') {
        warning.value = '⚠️ 注：お名前にかなや漢字が含まれている場合、送信されない可能性があります。上のテキストボックスで文字名を変更してから、QRコードを生成してください。'
        confirmMessage.value = 'キャラクターの見た目がおかしい？それは、KMSでは利用できない装備、顔、髪型を使用しているためです。'
        jmsConfirmMsg.value = '<p><a href="https://mushroom-lab.com/my-tools/lotte-world-qr" target="_blank">メイプル研究所のサイトを使ってQRコードを生成することもできます。</a></p>'
        table_title.value = 'KMSは利用できない装備'
        table_col_icon = 'アイコン'
        table_col_name = '名前'
      } else if (region.value === 'TerryMS_Peninsula') {
        warning.value = ''
        confirmMessage.value = 'Character doesn\'t look right ? That\'s because you\'re using gears, face, hair that unavailable in KMS.'
        jmsConfirmMsg.value = ''
        table_title.value = 'Unavailable Gears in KMS'
        table_col_icon = 'Icon'
        table_col_name = 'Name'
      } else {
        warning.value = ''
        confirmMessage.value = ''
        jmsConfirmMsg.value = ''
        table_title.value = ''
        table_col_icon = ''
        table_col_name = ''
      }
      avatarCode.value = data.avatarCode
    } else if (data.code === '0000010D') {
      error.value = data.reason_ja
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
  qrValue.value = `01/${avatarCode.value}|${encodedName}|0|1|1|1`

  nextTick(() => {
    updateQrSize()
    // Extra force repaint for mobile canvas issues
    setTimeout(() => {
      const canvas = document.querySelector('.qr-section canvas')
      if (canvas) {
        canvas.style.display = 'none'
        // Trigger reflow
        void canvas.offsetHeight
        canvas.style.display = 'block'
      }
    }, 50)
  })
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

function containsChinese(str) {
  return /[\u4E00-\u9FFF]/.test(str)
}

function containsKana(str) {
  return /[\u3040-\u309F\u30A0-\u30FF]/.test(str)
}

function containsHangul(str) {
  return /[\uAC00-\uD7AF]/.test(str)
}

function getGearImageUrl(gearID) {
  if (region.value === 'TerryMS_Continent' || region.value === 'TerryMS_Island' || region.value === 'ZipanguMS' || region.value === 'InkwellMS_NA' || region.value === 'InkwellMS_EU' || region.value === 'TunerMS') {
    return `https://open.api.nexon.com/static/maplestorytw/ItemIcon/${gearID}.png`
  } else if (region.value === 'TerryMS_Peninsula') {
    return `https://open.api.nexon.com/static/maplestorysea/ItemIcon/${gearID}.png`
  } else if (region.value === 'ChangseopMS') {
    return `https://open.api.nexon.com/static/maplestory/ItemIcon/${gearID}.png`
  }
  // Fallback to a default icon
  return `https://open.api.nexon.com/static/maplestory/ItemIcon/${gearID}.png`
}

</script>

<style>
.container {
  font-family: sans-serif;
  padding: 1em;
  max-width: 600px;
  margin: auto;
}

body {
  margin: 0;
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

.image-table {
  width: 100%;
  border-collapse: collapse;
  text-align: center;
}

.image-cell {
  vertical-align: middle;
  /* ensures both images align vertically */
  padding: 8px;
}

.image-cell img {
  max-width: 100%;
  height: auto;
  display: block;
  margin: 0 auto;
}

.caption-cell {
  margin-top: 0.5rem;
  font-weight: bold;
  text-align: center;
}

.gear-table {
  width: 100%;
  border-collapse: collapse;
  margin-top: 1rem;
  background-color: #f9f9f9;
  border: 1px solid #ddd;
}

.gear-table thead {
  background-color: #3498db;
  color: white;
}

.gear-table th,
.gear-table td {
  padding: 0.75rem;
  text-align: left;
  border-bottom: 1px solid #ddd;
}

.gear-table tbody tr:hover {
  background-color: #f0f0f0;
}

.gear-table h3 {
  margin: 1rem 0 0.5rem 0;
}

.gear-image-cell {
  text-align: center;
  vertical-align: middle;
}

.gear-icon {
  width: 50px;
  height: 50px;
  object-fit: contain;
  image-rendering: pixelated;
}

@media (max-width: 600px) {
  .gear-icon {
    width: 32px;
    height: 32px;
  }
}

/* Mobile: stack image + caption together */
@media (max-width: 600px) {

  .image-table,
  .image-table tbody,
  .image-table tr,
  .image-table td {
    display: block;
    width: 100%;
  }

  .image-cell {
    margin-bottom: 1.5rem;
  }

  /* Keep gear table in single line on mobile */
  .gear-table {
    display: table;
    width: 100%;
  }

  .gear-table tbody,
  .gear-table tr {
    display: table-row;
  }

  .gear-table td,
  .gear-table th {
    display: table-cell;
    padding: 0.75rem 0.5rem;
  }

  .gear-image-cell {
    width: 40px;
    padding: 0.5rem 0.25rem;
  }

  .gear-table td {
    padding: 0.5rem 0.4rem;
    font-size: 0.9rem;
  }
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
