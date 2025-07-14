<script setup lang="ts">
import { computed, onMounted, ref, watch } from 'vue'
import { makeVietQRContent } from '../libs/vietqr';
import banks from './banks.json'
import domtoimage from 'dom-to-image';
import QRCode from "qrcode-svg";
// import { Download } from 'lucide-vue-next'

const amount = ref()
const description = ref('')
const bankId = ref('')
const accountId = ref('')
const accountName = ref('')


// Validation states
const errors = ref({
  bankId: '',
  accountId: '',
  accountName: '',
  amount: '',
  description: ''
})

const selectedBank = computed(() => banks.find(bank => bank.bin === bankId.value))

// Validation functions
const validateBankId = () => {
  if (!bankId.value) {
    errors.value.bankId = 'Vui lòng chọn ngân hàng'
    return false
  }
  errors.value.bankId = ''
  return true
}

const validateAccountId = () => {
  if (!accountId.value) {
    errors.value.accountId = 'Vui lòng nhập số tài khoản'
    return false
  }
  if (accountId.value.length < 6) {
    errors.value.accountId = 'Số tài khoản phải có ít nhất 6 ký tự'
    return false
  }
  if (!/^\d+$/.test(accountId.value)) {
    errors.value.accountId = 'Số tài khoản chỉ được chứa các chữ số'
    return false
  }
  errors.value.accountId = ''
  return true
}

const validateAmount = () => {
  if (amount.value && amount.value.trim()) {
    const numericAmount = parseInt(amount.value.replace(/\D/g, ''))
    if (numericAmount > 500000000) {
      errors.value.amount = 'Số tiền không được vượt quá 500,000,000 VND'
      return false
    }
    if (numericAmount < 1000) {
      errors.value.amount = 'Số tiền phải ít nhất 1,000 VND'
      return false
    }
  }
  errors.value.amount = ''
  return true
}

const validateAccountName = () => {
  if (accountName.value && accountName.value.trim()) {
    if (!/^[a-zA-ZÀÁÂÃÈÉÊÌÍÒÓÔÕÙÚÝàáâãèéêìíòóôõùúýĂăĐđĨĩŨũƠơƯưẠ-ỹ\s]*$/.test(accountName.value)) {
      errors.value.accountName = 'Tên chỉ được chứa chữ cái và dấu cách'
      return false
    }
    if (accountName.value.trim().length < 2) {
      errors.value.accountName = 'Tên phải có ít nhất 2 ký tự'
      return false
    }
  }
  errors.value.accountName = ''
  return true
}

const validateDescription = () => {
  if (description.value && description.value.trim()) {
    if (!/^[a-zA-Z0-9\s]*$/.test(description.value)) {
      errors.value.description = 'Nội dung chỉ được chứa chữ cái, số và dấu cách (không dấu)'
      return false
    }
  }
  errors.value.description = ''
  return true
}

const isFormValid = computed(() => {
  return bankId.value && accountId.value && !errors.value.bankId && !errors.value.accountId && !errors.value.accountName && !errors.value.amount && !errors.value.description
})

// Watch for changes and validate
watch(bankId, validateBankId)
watch(accountId, validateAccountId)
watch(accountName, validateAccountName)
watch(amount, validateAmount)
watch(description, validateDescription)

watch(bankId, () => {
  window.localStorage.setItem('bankId', bankId.value)
})

watch(accountId, () => {
  window.localStorage.setItem('accountId', accountId.value)
})

watch(accountName, () => {
  window.localStorage.setItem('accountName', accountName.value)
})

let debouncingStorage: number = 0

const updateAmount = (event: Event) => {
  const target = event.target as HTMLInputElement

  if (debouncingStorage) {
    clearTimeout(debouncingStorage)
  }

  setTimeout(() => {
    // remove leading zero
    let v = target.value.toString().replace(/^0+/, '')
    
    // remove any non-digit character
    v = v.replace(/\D/g, '')

    // insert comma every 3 digits
    v = v.replace(/\B(?=(\d{3})+(?!\d))/g, ',')

    amount.value = v

    window.localStorage.setItem('amount', v)
  }, 100)
}

watch(description, () => {
  window.localStorage.setItem('content', description.value)
})

onMounted(() => {
  // Initialize form data
  const bankIdFromStorage = window.localStorage.getItem('bankId')
  if (bankIdFromStorage) {
    bankId.value = bankIdFromStorage
  }

  const accountIdFromStorage = window.localStorage.getItem('accountId')
  if (accountIdFromStorage) {
    accountId.value = accountIdFromStorage
  }

  const amountFromStorage = window.localStorage.getItem('amount')
  if (amountFromStorage) {
    amount.value = amountFromStorage
  }

  const contentFromStorage = window.localStorage.getItem('content')
  if (contentFromStorage) {
    description.value = contentFromStorage
  }

  const accountNameFromStorage = window.localStorage.getItem('accountName')
  if (accountNameFromStorage) {
    accountName.value = accountNameFromStorage
  }
})

const qrContent = computed(() => makeVietQRContent({
  bankId: bankId.value,
  accountId: accountId.value,
  amount: parseInt(amount.value?.replace(/\D/g, '')),
  description: description.value.trim(),
}))

const qrSVG = computed(() => {
  const qr = new QRCode({
    content: qrContent.value,
    padding: 5,
    width: 300,
    height: 300,
    color: "#000000",
    background: "#ffffff",
    ecl: "M"
  });
  return qr.svg()
})

const downloadText = ref('Tải xuống mã QR')

const qrNode = ref<HTMLDivElement>()

const downloadImage = async () => {
  if (!qrNode.value) {
    downloadText.value = 'Lỗi: Không tìm thấy mã QR'
    setTimeout(() => {
      downloadText.value = 'Tải xuống mã QR'
    }, 2000);
    return
  }

  if (!isFormValid.value) {
    downloadText.value = 'Vui lòng điền đầy đủ thông tin'
    setTimeout(() => {
      downloadText.value = 'Tải xuống mã QR'
    }, 2000);
    return
  }

  try {
    downloadText.value = 'Đang tạo mã QR...'
    const dataUrl = await domtoimage.toPng(qrNode.value)
    
    // Create download link
    const link = document.createElement('a')
    link.download = `VietQR-${selectedBank.value?.shortName || 'QR'}-${accountId.value || 'code'}.png`
    link.href = dataUrl
    
    // Trigger download
    document.body.appendChild(link)
    link.click()
    document.body.removeChild(link)
    
    downloadText.value = 'Đã tải xuống'
    setTimeout(() => {
      downloadText.value = 'Tải xuống mã QR'
    }, 2000);
  } catch (error) {
    console.error('Download failed:', error)
    downloadText.value = 'Lỗi khi tải xuống'
    setTimeout(() => {
      downloadText.value = 'Tải xuống mã QR'
    }, 2000);
  }
}
</script>

<template>
  <div class="min-h-screen bg-gray-50">
    <!-- Top Gradient Header -->
    <div class="bg-gradient-to-r from-cyan-400 to-green-400 h-24 flex items-center justify-center">
      <div class="flex items-center space-x-3">
        <div class="w-8 h-8 bg-white rounded-full flex items-center justify-center">
          <span class="text-cyan-500 text-lg">🌐</span>
        </div>
        <h1 class="text-white text-3xl font-bold">VietQR</h1>
      </div>
    </div>

    <!-- Main Content -->
    <div class="container mx-auto px-6 py-8 max-w-7xl">
      <div class="grid grid-cols-1 lg:grid-cols-2 gap-8">
        <!-- Left Column - Form -->
        <div class="space-y-6">
          <div class="flex items-center space-x-2 mb-6">
            <span class="text-2xl">⚡</span>
            <h2 class="text-2xl font-bold text-gray-800">生成VietQR</h2>
          </div>
        
          <!-- Bank Selection -->
          <div class="space-y-2">
            <label class="block text-sm font-medium text-gray-700">选择银行：</label>
            <select 
              v-model="bankId"
              class="w-full h-12 px-4 py-2 text-base border border-gray-200 rounded-lg bg-white focus:outline-none focus:ring-2 focus:ring-cyan-500 focus:border-cyan-500"
              :class="errors.bankId ? 'border-red-500' : ''"
            >
              <option value="">Chọn ngân hàng</option>
              <option v-for="bank in banks" :key="bank.bin" :value="bank.bin">
                {{ bank.shortName }} ({{ bank.code }})
              </option>
            </select>
            <p v-if="errors.bankId" class="text-red-500 text-sm">{{ errors.bankId }}</p>
          </div>

          <!-- Account Number -->
          <div class="space-y-2">
            <label class="block text-sm font-medium text-gray-700">银行账号：</label>
            <input 
              v-model="accountId" 
              placeholder="Nhập số tài khoản" 
              maxlength="25"
              type="text"
              class="w-full h-12 px-4 py-2 text-base border border-gray-200 rounded-lg bg-gray-50 focus:outline-none focus:ring-2 focus:ring-cyan-500 focus:border-cyan-500 focus:bg-white"
              :class="errors.accountId ? 'border-red-500' : ''"
            />
            <p v-if="errors.accountId" class="text-red-500 text-sm">{{ errors.accountId }}</p>
          </div>

          <!-- Account Name -->
          <div class="space-y-2">
            <label class="block text-sm font-medium text-gray-700">账户名：</label>
            <input 
              v-model="accountName" 
              placeholder="Tùy chọn, tên người nhận" 
              maxlength="50"
              type="text"
              class="w-full h-12 px-4 py-2 text-base border border-gray-200 rounded-lg bg-gray-50 focus:outline-none focus:ring-2 focus:ring-cyan-500 focus:border-cyan-500 focus:bg-white"
              :class="errors.accountName ? 'border-red-500' : ''"
            />
            <p v-if="errors.accountName" class="text-red-500 text-sm">{{ errors.accountName }}</p>
          </div>

          <!-- Amount -->
          <div class="space-y-2">
            <label class="block text-sm font-medium text-gray-700">金额（VND）：</label>
            <input 
              v-model="amount" 
              @input="updateAmount($event)"
              placeholder="输入金额（可选）" 
              maxlength="16"
              type="text"
              class="w-full h-12 px-4 py-2 text-base border border-gray-200 rounded-lg bg-gray-50 focus:outline-none focus:ring-2 focus:ring-cyan-500 focus:border-cyan-500 focus:bg-white"
              :class="errors.amount ? 'border-red-500' : ''"
            />
            <p v-if="errors.amount" class="text-red-500 text-sm">{{ errors.amount }}</p>
          </div>

          <!-- Description -->
          <div class="space-y-2">
            <label class="block text-sm font-medium text-gray-700">转账备注：</label>
            <textarea 
              v-model="description" 
              placeholder="转账说明（可选）" 
              maxlength="25"
              rows="3"
              class="w-full px-4 py-2 text-base border border-gray-200 rounded-lg bg-gray-50 focus:outline-none focus:ring-2 focus:ring-cyan-500 focus:border-cyan-500 focus:bg-white resize-none"
              :class="errors.description ? 'border-red-500' : ''"
            />
            <p v-if="errors.description" class="text-red-500 text-sm">{{ errors.description }}</p>
          </div>

          <!-- Generate Button -->
          <button 
            @click="downloadImage" 
            :disabled="!isFormValid"
            class="w-full h-14 bg-gradient-to-r from-cyan-500 to-green-500 text-white font-bold text-lg rounded-lg hover:from-cyan-600 hover:to-green-600 focus:outline-none focus:ring-2 focus:ring-cyan-500 focus:ring-offset-2 disabled:opacity-50 disabled:cursor-not-allowed flex items-center justify-center space-x-2"
          >
            <span>❤️</span>
            <span>{{ downloadText }}</span>
          </button>
        </div>

        <!-- Right Column - QR Preview -->
        <div class="lg:flex lg:justify-center">
          <div class="bg-white rounded-lg shadow-sm border border-gray-200 p-6 h-fit">
            <div class="flex items-center space-x-2 mb-4">
              <span class="text-xl">📱</span>
              <h3 class="text-lg font-bold text-gray-800">生成结果</h3>
            </div>
            
            <div class="text-center">
              <div v-if="isFormValid" class="space-y-4">
                <!-- QR Code -->
                <div class="bg-white p-6 rounded-lg border border-gray-100" ref="qrNode">
                  <div v-html="qrSVG" class="qr-image w-64 h-64 mx-auto"></div>
                  <div class="mt-4 space-y-1">
                    <div v-if="accountName.trim()" class="text-gray-800 font-medium">{{ accountName.trim() }}</div>
                    <div class="text-gray-600 text-sm">{{ accountId }}</div>
                    <div v-if="amount" class="text-gray-600 text-sm">{{ amount }} VND</div>
                  </div>
                </div>
              </div>
              <div v-else class="flex items-center justify-center h-80 border-2 border-dashed border-gray-300 rounded-lg">
                <div class="text-center text-gray-500">
                  <div class="text-4xl mb-2">📱</div>
                  <p class="text-sm">请填写完整信息</p>
                  <p class="text-sm">生成QR码</p>
                </div>
              </div>
            </div>
          </div>
        </div>
      </div>
    </div>
  </div>
</template>

<style scoped>
.qr-image {
  image-rendering: pixelated;
}

.qr-image svg {
  width: 100%;
  height: auto;
}
</style>

<style>
/* 强制下拉框滚动 */
[data-radix-select-content] {
  max-height: 320px !important;
  overflow-y: auto !important;
}

/* 确保下拉框选项可见 */
[data-radix-select-viewport] {
  max-height: 320px !important;
  overflow-y: auto !important;
}
</style>
