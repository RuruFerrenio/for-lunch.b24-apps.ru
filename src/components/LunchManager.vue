<script setup lang="ts">
import { ref, computed, onMounted } from 'vue'
import { useToast } from '@bitrix24/b24ui-nuxt/composables/useToast'
import CoffeeIcon from '@bitrix24/b24icons-vue/outline/PowerIcon'
import BriefcaseIcon from '@bitrix24/b24icons-vue/outline/PowerIcon'


type LunchMode = 'start' | 'end'
type WorkdayStatus = 'OPENED' | 'CLOSED' | 'PAUSED' | 'EXPIRED'

interface WorkdayInfo {
  STATUS: WorkdayStatus
  TIME_START?: string
  TIME_FINISH?: string
  TIME_PAUSED?: string
  [key: string]: any
}

interface CurrentUser {
  id: number
  name: string
  lastName: string
  email: string
}

interface BX24Error {
  error?: string
  error_description?: string
  message?: string
}

const props = withDefaults(defineProps<{
  mode: LunchMode
  alertaParameters?: Record<string, any>
  autoCloseDelay?: number
}>(), {
  alertaParameters: () => ({}),
  autoCloseDelay: 2000
})

const toast = useToast()

const isProcessing = ref(false)
const isActionCompleted = ref(false)
const error = ref<string | null>(null)
const statusMessage = ref('')
const workdayInfo = ref<WorkdayInfo | null>(null)
const isBX24Ready = ref(false)

const currentUser = ref<CurrentUser>({
  id: 0,
  name: 'Сотрудник',
  lastName: '',
  email: ''
})

const isStartLunch = computed(() => props.mode === 'start')

const title = computed(() => {
  if (isActionCompleted.value) {
    return isStartLunch.value ? 'Обеденный перерыв начат' : 'Возвращение с обеда'
  }
  return isStartLunch.value ? 'Обеденный перерыв' : 'Возвращение с обеда'
})

const subtitle = computed(() => {
  if (isStartLunch.value) {
    return 'Поставьте рабочий день на паузу,\nчтобы уйти на обед'
  }
  return 'Возобновите рабочий день,\nчтобы продолжить работу'
})

const completionMessage = computed(() => {
  return isStartLunch.value
      ? 'Обеденный перерыв начался'
      : 'Рабочий день возобновлен'
})

const buttonText = computed(() => {
  return isStartLunch.value ? 'На обед!' : 'За работу!'
})

const processingText = computed(() => {
  return isStartLunch.value ? 'Уходим на обед...' : 'Возвращаемся...'
})

const completedText = computed(() => {
  return isStartLunch.value ? 'На обеде' : 'За работой'
})

const buttonClass = computed(() => {
  const isDisabled = isProcessing.value || isActionCompleted.value
  const isEnabled = !isProcessing.value && !isActionCompleted.value

  const classes: Record<string, boolean> = {
    'bg-gray-300 text-gray-600 cursor-not-allowed hover:bg-gray-300': isDisabled,
    'shadow-md hover:shadow-lg': isEnabled
  }

  if (isStartLunch.value) {
    classes['bg-blue-600 hover:bg-blue-700 text-white hover:text-white'] = isEnabled
  } else {
    classes['bg-green-600 hover:bg-green-700 text-white hover:text-white'] = isEnabled
  }

  return classes
})

const statusClass = computed(() => {
  if (error.value) return 'text-red-600 font-medium'
  if (isActionCompleted.value) return 'text-green-600 font-medium'
  return 'text-gray-500'
})

const getStatusText = (status: WorkdayStatus | string): string => {
  const statusMap: Record<string, string> = {
    'OPENED': 'Работает',
    'CLOSED': 'Закрыт',
    'PAUSED': 'На обеде',
    'EXPIRED': 'Истек'
  }
  return statusMap[status] || status
}

const formatDateTime = (dateTimeStr?: string): string => {
  if (!dateTimeStr) return '—'
  try {
    const date = new Date(dateTimeStr)
    return date.toLocaleString('ru-RU', {
      hour: '2-digit',
      minute: '2-digit',
      second: '2-digit',
      day: '2-digit',
      month: '2-digit',
      year: 'numeric'
    })
  } catch {
    return dateTimeStr
  }
}

const loadCurrentUser = async (): Promise<CurrentUser> => {
  if (typeof window === 'undefined' || typeof (window as any).BX24 === 'undefined') {
    return {
      id: 0,
      name: 'Тестовый пользователь',
      lastName: '',
      email: ''
    }
  }

  const BX24 = (window as any).BX24

  try {
    const userData = await new Promise<any>((resolve, reject) => {
      BX24.callMethod('user.current', {}, (result: any) => {
        if (result.error()) {
          reject(result.error())
        } else {
          resolve(result.data())
        }
      })
    })

    const fullName = userData.NAME || ''
    const lastName = userData.LAST_NAME || ''

    let displayName = ''
    if (fullName || lastName) {
      displayName = `${fullName} ${lastName}`.trim()
    } else if (userData.EMAIL) {
      displayName = userData.EMAIL.split('@')[0]
    } else {
      displayName = `Сотрудник ${userData.ID}`
    }

    return {
      id: parseInt(userData.ID) || 0,
      name: displayName,
      lastName: lastName,
      email: userData.EMAIL || ''
    }

  } catch (err) {
    console.warn('Ошибка при получении user.current:', err)

    try {
      const authData = BX24.getAuth()
      if (authData && authData.user_id) {
        return {
          id: parseInt(authData.user_id),
          name: authData.user_name || `Сотрудник ${authData.user_id}`,
          lastName: '',
          email: authData.user_email || ''
        }
      }
    } catch (authError) {
    }

    return {
      id: 0,
      name: 'Сотрудник',
      lastName: '',
      email: ''
    }
  }
}

const getCurrentWorkdayStatus = async (): Promise<WorkdayInfo | null> => {
  if (typeof window === 'undefined' || typeof (window as any).BX24 === 'undefined') return null

  const BX24 = (window as any).BX24

  try {
    const result = await new Promise<WorkdayInfo>((resolve, reject) => {
      BX24.callMethod('timeman.status', {}, (result: any) => {
        if (result.error()) {
          reject(result.error())
        } else {
          resolve(result.data())
        }
      })
    })
    return result
  } catch (error) {
    console.error('Ошибка получения статуса рабочего дня:', error)
    return null
  }
}

const pauseWorkday = async (): Promise<WorkdayInfo> => {
  const BX24 = (window as any).BX24

  const status = await getCurrentWorkdayStatus()

  if (!status || status.STATUS !== 'OPENED') {
    if (status?.STATUS === 'PAUSED') {
      throw new Error('Рабочий день уже на паузе')
    }
    if (status?.STATUS === 'CLOSED') {
      throw new Error('Рабочий день уже завершен. Сначала начните рабочий день')
    }
    throw new Error('Нельзя поставить на паузу - рабочий день не активен')
  }

  const params: Record<string, any> = {}

  if (currentUser.value.id && currentUser.value.id > 0) {
    params.USER_ID = currentUser.value.id
  }

  const result = await new Promise<WorkdayInfo>((resolve, reject) => {
    BX24.callMethod('timeman.pause', params, (result: any) => {
      if (result.error()) {
        reject(result.error())
      } else {
        resolve(result.data())
      }
    })
  })

  return result
}

const resumeWorkday = async (): Promise<WorkdayInfo> => {
  const BX24 = (window as any).BX24

  const status = await getCurrentWorkdayStatus()

  if (!status || status.STATUS !== 'PAUSED') {
    if (status?.STATUS === 'OPENED') {
      throw new Error('Рабочий день не на паузе')
    }
    if (status?.STATUS === 'CLOSED') {
      throw new Error('Рабочий день уже завершен. Сначала начните рабочий день')
    }
    throw new Error('Нельзя возобновить - рабочий день не на паузе')
  }

  const params: Record<string, any> = {}

  if (currentUser.value.id && currentUser.value.id > 0) {
    params.USER_ID = currentUser.value.id
  }

  const result = await new Promise<WorkdayInfo>((resolve, reject) => {
    BX24.callMethod('timeman.resume', params, (result: any) => {
      if (result.error()) {
        reject(result.error())
      } else {
        resolve(result.data())
      }
    })
  })

  return result
}

const closeApplication = (): void => {
  if (typeof window !== 'undefined' && (window as any).BX24 && typeof (window as any).BX24.closeApplication === 'function') {
    (window as any).BX24.closeApplication()
  } else {
    console.error('Функция BX24.closeApplication недоступна')
    if (window.close) {
      window.close()
    }
  }
}

const executeAction = async (): Promise<void> => {
  if (isProcessing.value || isActionCompleted.value) return

  error.value = null
  statusMessage.value = ''

  if (typeof window === 'undefined' || typeof (window as any).BX24 === 'undefined') {
    error.value = 'API Битрикс24 недоступно'
    toast.add({
      description: error.value,
      variant: 'error'
    })
    return
  }

  try {
    isProcessing.value = true
    statusMessage.value = isStartLunch.value ? 'Ставим рабочий день на паузу...' : 'Возобновляем рабочий день...'

    let result: WorkdayInfo
    if (isStartLunch.value) {
      result = await pauseWorkday()
    } else {
      result = await resumeWorkday()
    }

    workdayInfo.value = result
    isActionCompleted.value = true
    statusMessage.value = isStartLunch.value
        ? 'Обеденный перерыв начался'
        : 'Рабочий день возобновлен'

    toast.add({
      description: statusMessage.value,
      variant: 'success'
    })

    setTimeout(() => {
      closeApplication()
    }, props.autoCloseDelay)

  } catch (err) {
    console.error(`Ошибка при ${isStartLunch.value ? 'начале' : 'завершении'} обеда:`, err)

    const bxError = err as BX24Error
    let errorMessage = bxError.error_description || bxError.message || 'Неизвестная ошибка'

    if (bxError.error === 'ACCESS_DENIED' || bxError.error === 'insufficient_scope') {
      errorMessage = 'Недостаточно прав для выполнения операции'
    } else if (bxError.error === 'USER_NOT_FOUND' || bxError.message?.includes('User not found')) {
      errorMessage = 'Пользователь не найден'
    } else if (bxError.error === 'METHOD_NOT_FOUND') {
      errorMessage = 'Метод timeman.pause не поддерживается версией Битрикс24'
    }

    error.value = errorMessage
    statusMessage.value = errorMessage

    toast.add({
      description: errorMessage,
      variant: 'error'
    })
  } finally {
    isProcessing.value = false
  }
}

const initializeComponent = async (): Promise<void> => {
  try {
    const user = await loadCurrentUser()
    currentUser.value = user

    const status = await getCurrentWorkdayStatus()
    if (status) {
      workdayInfo.value = status

      if (isStartLunch.value && status.STATUS === 'PAUSED') {
        isActionCompleted.value = true
        statusMessage.value = 'Уже на обеде'
      }

      if (!isStartLunch.value && status.STATUS === 'OPENED') {
        isActionCompleted.value = true
        statusMessage.value = 'Уже за работой'
      }

      if (!isStartLunch.value && status.STATUS === 'CLOSED') {
        error.value = 'Рабочий день не начат. Сначала начните рабочий день'
        statusMessage.value = error.value
      }

      if (isStartLunch.value && status.STATUS === 'CLOSED') {
        error.value = 'Рабочий день не начат. Сначала начните рабочий день'
        statusMessage.value = error.value
      }
    }
  } catch (error) {
    console.error('Ошибка инициализации компонента:', error)
  }
}

const clearCookies = () => {
  if (typeof document === 'undefined') return

  const cookies = document.cookie.split(';');

  cookies.forEach(cookie => {
    const [name] = cookie.split('=');
    const cookieName = name.trim();

    if (cookieName) {
      document.cookie = `${cookieName}=; expires=Thu, 01 Jan 1970 00:00:00 UTC; path=/; SameSite=None; Secure`;
    }
  });
};

onMounted(() => {
  clearCookies();

  const checkBX24 = () => {
    if (typeof window !== 'undefined' && (window as any).BX24) {
      const BX24 = (window as any).BX24;
      isBX24Ready.value = true;

      if (BX24.init) {
        BX24.init(async () => {
          await initializeComponent();
        });
      } else {
        initializeComponent();
      }
    } else {
      setTimeout(checkBX24, 100);
    }
  };

  checkBX24();
});
</script>

<template>
  <div class="min-h-screen flex flex-col items-center justify-center bg-white p-4">
    <div class="text-center w-full">
      <!-- Иконка -->
      <div class="mb-8 flex justify-center">
        <div class="w-24 h-24 rounded-full flex items-center justify-center"
             :class="isStartLunch ? 'bg-blue-50' : 'bg-green-50'">
          <component
              :is="isStartLunch ? CoffeeIcon : BriefcaseIcon"
              class="w-12 h-12"
              :class="isStartLunch ? 'text-blue-600' : 'text-green-600'"
          />
        </div>
      </div>

      <!-- Заголовок -->
      <h1 class="text-2xl font-bold text-gray-900 mb-4">
        {{ title }}
      </h1>

      <!-- Подзаголовок -->
      <p class="text-gray-600 mb-8 whitespace-pre-line" v-if="!isActionCompleted && !error">
        {{ subtitle }}
      </p>
      <p class="text-gray-600 mb-8" v-else-if="isActionCompleted">
        {{ completionMessage }}
      </p>

      <!-- Кнопка действия -->
      <div class="mb-4" v-if="!isActionCompleted && !error">
        <B24Button
            @click="executeAction"
            :disabled="isProcessing || isActionCompleted || !isBX24Ready"
            variant="primary"
            size="lg"
            class="w-full h-15 rounded-full text-lg font-semibold transition-all duration-300 transform hover:scale-105 active:scale-95"
            :class="buttonClass"
        >
          <div class="flex items-center justify-center">
            <span v-if="!isProcessing && !isActionCompleted">
              {{ buttonText }}
            </span>
            <span v-else-if="isProcessing" class="flex items-center">
              <svg class="w-6 h-6 mr-2 animate-spin" fill="none" stroke="currentColor" viewBox="0 0 24 24">
                <path stroke-linecap="round" stroke-linejoin="round" stroke-width="2"
                      d="M4 4v5h.582m15.356 2A8.001 8.001 0 004.582 9m0 0H9m11 11v-5h-.581m0 0a8.003 8.003 0 01-15.357-2m15.357 2H15" />
              </svg>
              {{ processingText }}
            </span>
            <span v-else-if="isActionCompleted" class="flex items-center">
              <svg class="w-6 h-6 mr-2" fill="none" stroke="currentColor" viewBox="0 0 24 24">
                <path stroke-linecap="round" stroke-linejoin="round" stroke-width="2"
                      d="M5 13l4 4L19 7" />
              </svg>
              {{ completedText }}
            </span>
          </div>
        </B24Button>
      </div>

      <!-- Статус -->
      <div v-if="statusMessage && !error" class="text-sm mb-6" :class="statusClass">
        {{ statusMessage }}
      </div>

      <!-- Сообщение об ошибке -->
      <div v-if="error" class="mt-4 p-3 rounded-lg bg-red-50 text-red-700 border border-red-200">
        <div class="flex items-center justify-center">
          <svg class="w-5 h-5 mr-2 flex-shrink-0" fill="none" stroke="currentColor" viewBox="0 0 24 24">
            <path stroke-linecap="round" stroke-linejoin="round" stroke-width="2"
                  d="M12 9v2m0 4h.01m-6.938 4h13.856c1.54 0 2.502-1.667 1.732-2.5L13.732 4c-.77-.833-1.964-.833-2.732 0L4.346 16.5c-.77.833.192 2.5 1.732 2.5z" />
          </svg>
          <span class="text-sm font-medium">{{ error }}</span>
        </div>
      </div>
    </div>

    <B24NotificationContainer position="top-right" />
  </div>
</template>

<style scoped>
@keyframes spin {
  from {
    transform: rotate(0deg);
  }
  to {
    transform: rotate(360deg);
  }
}

.animate-spin {
  animation: spin 1s linear infinite;
}

.transition-all {
  transition-property: all;
  transition-timing-function: cubic-bezier(0.4, 0, 0.2, 1);
}

.hover\:scale-105:hover {
  transform: scale(1.05);
}

.active\:scale-95:active {
  transform: scale(0.95);
}
</style>