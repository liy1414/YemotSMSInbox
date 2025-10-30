<script setup>
import { ref, computed } from 'vue';
import { XMarkIcon, CheckIcon } from '@heroicons/vue/24/outline';

const props = defineProps({
  visible: {
    type: Boolean,
    default: false
  }
});

const emit = defineEmits(['close', 'success']);

const step = ref('input'); // 'input', 'verification', 'success'
const callerId = ref('');
const validType = ref('SMS'); // 'SMS' or 'CALL'
const reqId = ref('');
const verificationCode = ref('');
const loading = ref(false);
const error = ref('');
const success = ref('');

const isValidCallerId = computed(() => {
  // בדיקה למספר טלפון ישראלי
  const cleanNumber = callerId.value.replace(/\s/g, '').replace(/-/g, '');
  
  // ביטוי רגולרי מקיף למספרים ישראליים
  const israeliFormat = /^0(5[^7]|[2-4]|[8-9]|7[0-9])[0-9]{7}$/;
  
  const isValid = israeliFormat.test(cleanNumber);
  
  console.log('Phone validation:', callerId.value, 'cleanNumber:', cleanNumber, 'isValid:', isValid);
  return isValid;
});

const sendVerification = async () => {
  if (!isValidCallerId.value) {
    error.value = 'אנא הזן מספר טלפון תקין';
    return;
  }

  loading.value = true;
  error.value = '';

  // ניקוי המספר
  let cleanCallerId = callerId.value.replace(/\s/g, '').replace(/-/g, '');

  try {
    const response = await fetch('https://www.call2all.co.il/ym/api/ValidationCallerId', {
      method: 'POST',
      headers: {
        'Content-Type': 'application/json'
      },
      body: JSON.stringify({
        token: `${encodeURIComponent(token)}`,
        action: 'send',
        callerId: cleanCallerId,
        validType: validType.value
      })
    });

    const data = await response.json();

    if (data.responseStatus === 'OK') {
      reqId.value = data.reqId;
      step.value = 'verification';
      success.value = `קוד אימות נשלח ל${validType.value === 'SMS' ? 'הודעת SMS' : 'שיחת טלפון'} למספר ${callerId.value}`;
    } else {
      error.value = data.message || 'שגיאה בשליחת קוד האימות';
    }
  } catch (err) {
    error.value = 'שגיאה בחיבור לשרת';
    console.error('Error sending verification:', err);
  } finally {
    loading.value = false;
  }
};

const verifyCode = async () => {
  if (!verificationCode.value) {
    error.value = 'אנא הזן את קוד האימות';
    return;
  }

  loading.value = true;
  error.value = '';

  try {
    const response = await fetch('https://www.call2all.co.il/ym/api/ValidationCallerId', {
      method: 'POST',
      headers: {
        'Content-Type': 'application/json'
      },
      body: JSON.stringify({
        token: `${encodeURIComponent(token)}`,
        action: 'valid',
        reId: reqId.value,
        code: verificationCode.value
      })
    });

    const data = await response.json();

    if (data.responseStatus === 'OK' && data.status === true) {
      step.value = 'success';
      success.value = 'הזיהוי יוצא נוסף בהצלחה!';
      setTimeout(() => {
        emit('success');
        closeModal();
      }, 2000);
    } else {
      error.value = data.message || 'קוד האימות שגוי';
    }
  } catch (err) {
    error.value = 'שגיאה בחיבור לשרת';
    console.error('Error verifying code:', err);
  } finally {
    loading.value = false;
  }
};

const closeModal = () => {
  step.value = 'input';
  callerId.value = '';
  validType.value = 'SMS';
  reqId.value = '';
  verificationCode.value = '';
  loading.value = false;
  error.value = '';
  success.value = '';
  emit('close');
};

const goBack = () => {
  step.value = 'input';
  error.value = '';
  success.value = '';
};
</script>

<template>
  <transition name="fade">
    <div v-if="visible" 
      class="fixed inset-0 bg-gray-900 bg-opacity-50 flex justify-center items-center z-50 p-2 sm:p-4"
      style="backdrop-filter: blur(8px); -webkit-backdrop-filter: blur(8px);"
      @click="closeModal">
      
      <div class="bg-white rounded-xl shadow-xl w-full max-w-[98%] sm:max-w-md max-h-[85vh] overflow-hidden mx-1" @click.stop>
        <!-- Header -->
        <div class="flex justify-between items-center p-4 sm:p-6 border-b border-gray-200" dir="rtl">
          <h3 class="text-lg sm:text-xl font-bold text-gray-800">
            {{ step === 'input' ? 'הוסף זיהוי יוצא חדש' : 
               step === 'verification' ? 'אימות זיהוי יוצא' : 'הזיהוי נוסף בהצלחה' }}
          </h3>
          <button @click="closeModal" class="p-1.5 rounded-full hover:bg-gray-100 text-gray-500 transition">
            <XMarkIcon class="w-4 h-4 sm:w-5 sm:h-5" />
          </button>
        </div>
        
        <!-- Content -->
        <div class="p-4 sm:p-6">
          <!-- Step 1: Input -->
          <div v-if="step === 'input'">
            <div class="space-y-4">
              <div>
                <label class="block text-sm font-medium text-gray-700 mb-2" dir="rtl">
                  מספר טלפון
                </label>
                <input 
                  v-model="callerId"
                  type="tel"
                  placeholder="הזן מספר טלפון (למשל: 0501234567)"
                  class="w-full px-3 py-2.5 border border-gray-300 rounded-lg focus:outline-none focus:ring-2 focus:ring-indigo-500 focus:border-indigo-500 text-right"
                  dir="rtl"
                  @input="error = ''"
                />
                <p class="text-xs text-gray-500 mt-1">מספר טלפון ישראלי בפורמט: 0XXXXXXXXX</p>
              </div>

              <div>
                <label class="block text-sm font-medium text-gray-700 mb-2" dir="rtl">
                  סוג אימות
                </label>
                <div class="space-y-2">
                  <label class="flex items-center cursor-pointer">
                    <input 
                      v-model="validType" 
                      type="radio" 
                      value="SMS" 
                      class="mr-2 text-indigo-600 focus:ring-indigo-500"
                    />
                    <span class="text-sm">הודעת SMS</span>
                  </label>
                  <label class="flex items-center cursor-pointer">
                    <input 
                      v-model="validType" 
                      type="radio" 
                      value="CALL" 
                      class="mr-2 text-indigo-600 focus:ring-indigo-500"
                    />
                    <span class="text-sm">שיחת טלפון</span>
                  </label>
                </div>
              </div>

              <div v-if="error" class="p-3 bg-red-50 border border-red-200 text-red-700 rounded-lg text-sm">
                {{ error }}
              </div>

              <button 
                @click="sendVerification"
                :disabled="loading || !isValidCallerId || !callerId.trim()"
                :class="[
                  'w-full py-2.5 px-4 rounded-lg font-medium transition-colors',
                  loading || !isValidCallerId || !callerId.trim()
                    ? 'bg-gray-300 text-gray-500 cursor-not-allowed' 
                    : 'bg-indigo-600 text-white hover:bg-indigo-700'
                ]"
              >
                <span v-if="!loading">שלח קוד אימות</span>
                <span v-else class="flex items-center justify-center">
                  <svg class="animate-spin -ml-1 mr-2 h-4 w-4" xmlns="http://www.w3.org/2000/svg" fill="none" viewBox="0 0 24 24">
                    <circle class="opacity-25" cx="12" cy="12" r="10" stroke="currentColor" stroke-width="4"></circle>
                    <path class="opacity-75" fill="currentColor" d="M4 12a8 8 0 018-8V0C5.373 0 0 5.373 0 12h4zm2 5.291A7.962 7.962 0 014 12H0c0 3.042 1.135 5.824 3 7.938l3-2.647z"></path>
                  </svg>
                  שולח...
                </span>
              </button>
            </div>
          </div>

          <!-- Step 2: Verification -->
          <div v-if="step === 'verification'">
            <div class="space-y-4">
              <div v-if="success" class="p-3 bg-green-50 border border-green-200 text-green-700 rounded-lg text-sm">
                {{ success }}
              </div>

              <div>
                <label class="block text-sm font-medium text-gray-700 mb-2" dir="rtl">
                  קוד אימות
                </label>
                <input 
                  v-model="verificationCode"
                  type="text"
                  placeholder="הזן את קוד האימות"
                  class="w-full px-3 py-2.5 border border-gray-300 rounded-lg focus:outline-none focus:ring-2 focus:ring-indigo-500 focus:border-indigo-500 text-center text-lg tracking-widest"
                  dir="rtl"
                  maxlength="6"
                />
              </div>

              <div v-if="error" class="p-3 bg-red-50 border border-red-200 text-red-700 rounded-lg text-sm">
                {{ error }}
              </div>

              <div class="flex space-x-3 space-x-reverse">
                <button 
                  @click="goBack"
                  class="flex-1 py-2.5 px-4 border border-gray-300 text-gray-700 rounded-lg hover:bg-gray-50 transition-colors"
                >
                  חזור
                </button>
                <button 
                  @click="verifyCode"
                  :disabled="loading || !verificationCode"
                  :class="[
                    'flex-1 py-2.5 px-4 rounded-lg font-medium transition-colors',
                    loading || !verificationCode 
                      ? 'bg-gray-300 text-gray-500 cursor-not-allowed' 
                      : 'bg-indigo-600 text-white hover:bg-indigo-700'
                  ]"
                >
                  <span v-if="!loading">אמת</span>
                  <span v-else class="flex items-center justify-center">
                    <svg class="animate-spin -ml-1 mr-2 h-4 w-4" xmlns="http://www.w3.org/2000/svg" fill="none" viewBox="0 0 24 24">
                      <circle class="opacity-25" cx="12" cy="12" r="10" stroke="currentColor" stroke-width="4"></circle>
                      <path class="opacity-75" fill="currentColor" d="M4 12a8 8 0 018-8V0C5.373 0 0 5.373 0 12h4zm2 5.291A7.962 7.962 0 014 12H0c0 3.042 1.135 5.824 3 7.938l3-2.647z"></path>
                    </svg>
                    מאמת...
                  </span>
                </button>
              </div>
            </div>
          </div>

          <!-- Step 3: Success -->
          <div v-if="step === 'success'" class="text-center py-8">
            <div class="w-16 h-16 bg-green-100 rounded-full flex items-center justify-center mx-auto mb-4">
              <CheckIcon class="w-8 h-8 text-green-600" />
            </div>
            <h4 class="text-lg font-semibold text-gray-900 mb-2">הזיהוי נוסף בהצלחה!</h4>
            <p class="text-gray-600">הזיהוי יוצא {{ callerId }} נוסף למערכת שלך</p>
          </div>
        </div>
      </div>
    </div>
  </transition>
</template>

<style scoped>
.fade-enter-active,
.fade-leave-active {
  transition: opacity 0.2s ease;
}

.fade-enter-from,
.fade-leave-to {
  opacity: 0;
}
</style> 
