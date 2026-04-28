<script setup lang="ts">
import { ref, onMounted, onUnmounted, computed, reactive } from 'vue'
import { useToast } from '@bitrix24/b24ui-nuxt/composables/useToast'
import { Time } from '@internationalized/date'
import CoffeeIcon from '@bitrix24/b24icons-vue/outline/PowerIcon'
import BriefcaseIcon from '@bitrix24/b24icons-vue/outline/PowerIcon'
import ClockIcon from '@bitrix24/b24icons-vue/outline/ClockIcon'
import CircleCheckIcon from '@bitrix24/b24icons-vue/outline/CircleCheckIcon'
import InfoCircleIcon from '@bitrix24/b24icons-vue/main/InfoCircleIcon'
import NoteCircleIcon from '@bitrix24/b24icons-vue/main/NoteCircleIcon'
import RefreshIcon from '@bitrix24/b24icons-vue/outline/RefreshIcon'
import RestaurantIcon from '@bitrix24/b24icons-vue/outline/PowerIcon'
import * as v from 'valibot'
import type { FormSubmitEvent } from '@bitrix24/b24ui-nuxt'

type WorkdayStatus = 'OPENED' | 'CLOSED' | 'PAUSED' | 'EXPIRED'

interface WorkdayInfo {
  STATUS: WorkdayStatus
  TIME_START?: string
  TIME_FINISH?: string
  TIME_PAUSED?: string
  DURATION?: string
  TIME_LEAKS?: string
  [key: string]: any
}

interface UserProfile {
  ID: number
  NAME: string
  LAST_NAME: string
  SECOND_NAME?: string
  FULL_NAME: string
  EMAIL: string
  WORK_POSITION: string
  PERSONAL_PHOTO?: string
  IS_ONLINE: 'Y' | 'N'
}

interface LunchFormData {
  startTime: string | null
  endTime: string | null
}

const toast = useToast()

const isBitrixLoaded = ref(false)
const isLoading = ref(true)
const isProcessing = ref(false)
const workdayInfo = ref<WorkdayInfo | null>(null)
const currentUser = ref<UserProfile | null>(null)
const error = ref<string | null>(null)
const isRefreshing = ref(false)

// Настройки обеда для формы
const lunchForm = reactive<LunchFormData>({
  startTime: null,
  endTime: null
})

const isSettingsSaved = ref(false)

// Схема валидации формы
const lunchSchema = v.object({
  startTime: v.pipe(
      v.string(),
      v.minLength(1, 'Укажите время начала обеда')
  ),
  endTime: v.pipe(
      v.string(),
      v.minLength(1, 'Укажите время окончания обеда')
  )
})

type LunchSchema = v.InferOutput<typeof lunchSchema>

// Флаг доступности методов timeman
const isTimemanAvailable = ref<boolean | null>(null)

let refreshInterval: ReturnType<typeof setInterval> | null = null
let isPageVisible = true

// Кэш профилей пользователей
const userProfilesCache = ref<Record<number, UserProfile>>({})

// Вычисляемые свойства для статуса
const statusText = computed(() => {
  if (!workdayInfo.value) return 'Неизвестно'

  const statusMap: Record<WorkdayStatus, string> = {
    'OPENED': 'Рабочий день активен',
    'CLOSED': 'Рабочий день не начат',
    'PAUSED': 'Обеденный перерыв',
    'EXPIRED': 'Рабочий день истек'
  }
  return statusMap[workdayInfo.value.STATUS] || workdayInfo.value.STATUS
})

const statusIcon = computed(() => {
  if (!workdayInfo.value) return InfoCircleIcon

  const iconMap: Record<WorkdayStatus, any> = {
    'OPENED': CircleCheckIcon,
    'CLOSED': InfoCircleIcon,
    'PAUSED': CoffeeIcon,
    'EXPIRED': NoteCircleIcon
  }
  return iconMap[workdayInfo.value.STATUS] || InfoCircleIcon
})

const statusColor = computed(() => {
  if (!workdayInfo.value) return 'text-gray-500'

  const colorMap: Record<WorkdayStatus, string> = {
    'OPENED': 'text-green-600',
    'CLOSED': 'text-gray-500',
    'PAUSED': 'text-blue-600',
    'EXPIRED': 'text-red-600'
  }
  return colorMap[workdayInfo.value.STATUS] || 'text-gray-500'
})

const statusBgColor = computed(() => {
  if (!workdayInfo.value) return 'bg-gray-100'

  const colorMap: Record<WorkdayStatus, string> = {
    'OPENED': 'bg-green-50 border-green-200',
    'CLOSED': 'bg-gray-50 border-gray-200',
    'PAUSED': 'bg-blue-50 border-blue-200',
    'EXPIRED': 'bg-red-50 border-red-200'
  }
  return colorMap[workdayInfo.value.STATUS] || 'bg-gray-100 border-gray-200'
})

const statusChipColor = computed(() => {
  if (!workdayInfo.value) return 'air-secondary-accent'

  const colorMap: Record<WorkdayStatus, string> = {
    'OPENED': 'air-primary-success',
    'CLOSED': 'air-secondary-accent',
    'PAUSED': 'air-primary',
    'EXPIRED': 'air-primary-alert'
  }
  return colorMap[workdayInfo.value.STATUS] || 'air-secondary-accent'
})

const canPauseWorkday = computed(() => {
  return workdayInfo.value && workdayInfo.value.STATUS === 'OPENED'
})

const canResumeWorkday = computed(() => {
  return workdayInfo.value && workdayInfo.value.STATUS === 'PAUSED'
})

const buttonText = computed(() => {
  if (canPauseWorkday.value) return 'На обед!'
  if (canResumeWorkday.value) return 'За работу!'
  return 'Обед'
})

const buttonIcon = computed(() => {
  if (canPauseWorkday.value) return CoffeeIcon
  if (canResumeWorkday.value) return BriefcaseIcon
  return RestaurantIcon
})

const buttonColor = computed(() => {
  if (canPauseWorkday.value) return 'air-primary'
  if (canResumeWorkday.value) return 'air-primary-success'
  return 'air-secondary-accent'
})

// Показывать ли основной контент
const showMainContent = computed(() => {
  return isTimemanAvailable.value === true && !error.value
})

const showUnavailableMessage = computed(() => {
  return isTimemanAvailable.value === false
})

// Форматирование времени обеда для отображения
const formattedLunchStart = computed(() => {
  if (!lunchForm.startTime) return '—'
  return lunchForm.startTime
})

const formattedLunchEnd = computed(() => {
  if (!lunchForm.endTime) return '—'
  return lunchForm.endTime
})

const hasLunchSettings = computed(() => {
  return lunchForm.startTime !== null && lunchForm.endTime !== null
})

// ==========================================================================
// РАБОТА С КУКИ
// ==========================================================================

const setCookie = (name: string, value: string, days: number = 365) => {
  if (typeof document === 'undefined') return

  const expires = new Date()
  expires.setTime(expires.getTime() + days * 24 * 60 * 60 * 1000)
  document.cookie = `${name}=${value}; expires=${expires.toUTCString()}; path=/; SameSite=None; Secure`
}

const getCookie = (name: string): string | null => {
  if (typeof document === 'undefined') return null

  const nameEQ = `${name}=`
  const ca = document.cookie.split(';')
  for (let i = 0; i < ca.length; i++) {
    let c = ca[i]
    while (c.charAt(0) === ' ') c = c.substring(1, c.length)
    if (c.indexOf(nameEQ) === 0) return c.substring(nameEQ.length, c.length)
  }
  return null
}

const saveLunchSettingsToCookies = () => {
  if (lunchForm.startTime) {
    setCookie('lunch_start_time', lunchForm.startTime)
  } else {
    setCookie('lunch_start_time', '', 0)
  }

  if (lunchForm.endTime) {
    setCookie('lunch_end_time', lunchForm.endTime)
  } else {
    setCookie('lunch_end_time', '', 0)
  }
}

const loadLunchSettingsFromCookies = () => {
  const startTimeStr = getCookie('lunch_start_time')
  const endTimeStr = getCookie('lunch_end_time')

  if (startTimeStr && startTimeStr !== '') {
    lunchForm.startTime = startTimeStr
  }

  if (endTimeStr && endTimeStr !== '') {
    lunchForm.endTime = endTimeStr
  }
}

// Преобразование строки времени в объект Time для B24InputTime
const stringToTime = (timeStr: string | null): Time | null => {
  if (!timeStr) return null
  const [hours, minutes] = timeStr.split(':')
  if (hours && minutes && !isNaN(parseInt(hours)) && !isNaN(parseInt(minutes))) {
    return new Time(parseInt(hours), parseInt(minutes), 0)
  }
  return null
}

const timeToString = (time: Time | null): string | null => {
  if (!time) return null
  return `${time.hour.toString().padStart(2, '0')}:${time.minute.toString().padStart(2, '0')}`
}

// Обработчик отправки формы
const handleSaveLunchSettings = (event: FormSubmitEvent<LunchSchema>) => {
  saveLunchSettingsToCookies()
  isSettingsSaved.value = true
  toast.add({
    description: 'Настройки времени обеда сохранены',
    variant: 'success'
  })

  setTimeout(() => {
    isSettingsSaved.value = false
  }, 2000)
}

// ==========================================================================
// ПРОВЕРКА ДОСТУПНОСТИ МЕТОДОВ
// ==========================================================================

const checkTimemanAvailability = async (): Promise<boolean> => {
  if (typeof BX24 === 'undefined') {
    return false
  }

  return new Promise((resolve) => {
    BX24.callMethod('method.get', {
      name: 'timeman.status'
    }, (result: any) => {
      if (result.error()) {
        resolve(false)
        return
      }

      const methodData = result.data()
      const isAvailable = methodData.isExisting && methodData.isAvailable
      resolve(isAvailable)
    })
  })
}

// ==========================================================================
// API ВЫЗОВЫ
// ==========================================================================

const getCurrentUserProfile = async (): Promise<UserProfile | null> => {
  if (!isBitrixLoaded.value || typeof BX24 === 'undefined') {
    return null
  }

  try {
    const result = await new Promise<any>((resolve, reject) => {
      BX24.callMethod('user.current', {}, (result: any) => {
        if (result.error()) {
          reject(result.error())
        } else {
          resolve(result.data())
        }
      })
    })

    const getFullName = (userData: any): string => {
      const parts = []
      if (userData.NAME) parts.push(userData.NAME)
      if (userData.LAST_NAME) parts.push(userData.LAST_NAME)
      if (userData.SECOND_NAME) parts.push(userData.SECOND_NAME)
      return parts.join(' ') || `Пользователь ${userData.ID}`
    }

    const userProfile: UserProfile = {
      ID: parseInt(result.ID),
      NAME: result.NAME || '',
      LAST_NAME: result.LAST_NAME || '',
      SECOND_NAME: result.SECOND_NAME || '',
      FULL_NAME: getFullName(result),
      EMAIL: result.EMAIL || '',
      WORK_POSITION: result.WORK_POSITION || '',
      PERSONAL_PHOTO: result.PERSONAL_PHOTO,
      IS_ONLINE: result.IS_ONLINE || 'N'
    }

    userProfilesCache.value[userProfile.ID] = userProfile
    return userProfile
  } catch (err) {
    return null
  }
}

const getWorkdayStatus = async (): Promise<WorkdayInfo | null> => {
  if (!isTimemanAvailable.value) {
    return null
  }

  if (!isBitrixLoaded.value || typeof BX24 === 'undefined') {
    return null
  }

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
  } catch (err) {
    error.value = 'Не удалось получить статус рабочего дня'
    return null
  }
}

const pauseWorkdayAPI = async (): Promise<void> => {
  if (!isBitrixLoaded.value || typeof BX24 === 'undefined') return
  if (!isTimemanAvailable.value) {
    throw new Error('Методы timeman недоступны')
  }

  return new Promise((resolve, reject) => {
    BX24.callMethod('timeman.pause', {}, (result: any) => {
      if (result.error()) {
        reject(result.error())
      } else {
        resolve(result.data())
      }
    })
  })
}

const resumeWorkdayAPI = async (): Promise<void> => {
  if (!isBitrixLoaded.value || typeof BX24 === 'undefined') return
  if (!isTimemanAvailable.value) {
    throw new Error('Методы timeman недоступны')
  }

  return new Promise((resolve, reject) => {
    BX24.callMethod('timeman.resume', {}, (result: any) => {
      if (result.error()) {
        reject(result.error())
      } else {
        resolve(result.data())
      }
    })
  })
}

// ==========================================================================
// ОСНОВНЫЕ ДЕЙСТВИЯ
// ==========================================================================

const refreshData = async () => {
  if (isRefreshing.value) return
  if (isTimemanAvailable.value === false) return

  isRefreshing.value = true

  try {
    const [user, status] = await Promise.all([
      getCurrentUserProfile(),
      getWorkdayStatus()
    ])

    if (user) currentUser.value = user
    if (status) workdayInfo.value = status

    error.value = null
  } catch (err) {
    error.value = 'Ошибка при обновлении данных'
  } finally {
    isRefreshing.value = false
    isLoading.value = false
  }
}

const handlePauseWorkday = async () => {
  if (isProcessing.value) return
  if (isTimemanAvailable.value === false) {
    toast.add({
      description: 'Функция недоступна на вашем тарифе. Требуется тариф "Профессиональный"',
      variant: 'error'
    })
    return
  }

  isProcessing.value = true
  error.value = null

  try {
    await pauseWorkdayAPI()
    toast.add({
      description: 'Обеденный перерыв начался. Приятного аппетита! 🍽️',
      variant: 'success'
    })
    await refreshData()
  } catch (err: any) {
    const errorMessage = err.error_description || err.message || 'Ошибка при начале обеда'
    error.value = errorMessage
    toast.add({
      description: errorMessage,
      variant: 'error'
    })
  } finally {
    isProcessing.value = false
  }
}

const handleResumeWorkday = async () => {
  if (isProcessing.value) return
  if (isTimemanAvailable.value === false) {
    toast.add({
      description: 'Функция недоступна на вашем тарифе. Требуется тариф "Профессиональный"',
      variant: 'error'
    })
    return
  }

  isProcessing.value = true
  error.value = null

  try {
    await resumeWorkdayAPI()
    toast.add({
      description: 'Обеденный перерыв завершен. Добро пожаловать обратно! 💪',
      variant: 'success'
    })
    await refreshData()
  } catch (err: any) {
    const errorMessage = err.error_description || err.message || 'Ошибка при завершении обеда'
    error.value = errorMessage
    toast.add({
      description: errorMessage,
      variant: 'error'
    })
  } finally {
    isProcessing.value = false
  }
}

const handleMainAction = () => {
  if (canPauseWorkday.value) {
    handlePauseWorkday()
  } else if (canResumeWorkday.value) {
    handleResumeWorkday()
  }
}

// ==========================================================================
// ЖИЗНЕННЫЙ ЦИКЛ
// ==========================================================================

const handleVisibilityChange = () => {
  const wasVisible = isPageVisible
  isPageVisible = !document.hidden

  if (isPageVisible && !wasVisible && isTimemanAvailable.value === true) {
    refreshData()
  }
}

const startAutoRefresh = () => {
  if (refreshInterval) clearInterval(refreshInterval)
  refreshInterval = setInterval(() => {
    if (isPageVisible && isTimemanAvailable.value === true) {
      refreshData()
    }
  }, 30000)
}

const getUserInitials = (user: UserProfile | null): string => {
  if (!user) return '?'
  const firstName = user.NAME || ''
  const lastName = user.LAST_NAME || ''
  if (firstName && lastName) {
    return (firstName[0] + lastName[0]).toUpperCase()
  }
  if (firstName) return firstName.substring(0, 2).toUpperCase()
  if (user.EMAIL) return user.EMAIL.substring(0, 2).toUpperCase()
  return 'С'
}

const getUserPhoto = (user: UserProfile | null): string | null => {
  return user?.PERSONAL_PHOTO || null
}

onMounted(() => {
  // Загружаем настройки обеда из куки
  loadLunchSettingsFromCookies()

  if (typeof BX24 !== 'undefined' && BX24.init) {
    BX24.init(async () => {
      isBitrixLoaded.value = true
      isTimemanAvailable.value = await checkTimemanAvailability()

      if (!isTimemanAvailable.value) {
        isLoading.value = false
        return
      }

      await refreshData()
      startAutoRefresh()
    })
  } else if (typeof BX24 !== 'undefined') {
    isBitrixLoaded.value = true

    checkTimemanAvailability().then((isAvailable) => {
      isTimemanAvailable.value = isAvailable

      if (!isAvailable) {
        isLoading.value = false
        return
      }

      refreshData()
      startAutoRefresh()
    })
  } else {
    isLoading.value = false
    error.value = 'API Битрикс24 не обнаружен. Убедитесь, что приложение запущено в среде Битрикс24.'
  }

  document.addEventListener('visibilitychange', handleVisibilityChange)
  isPageVisible = !document.hidden
})

onUnmounted(() => {
  if (refreshInterval) clearInterval(refreshInterval)
  document.removeEventListener('visibilitychange', handleVisibilityChange)
})
</script>

<template>
  <div class="bg-white rounded-xl shadow-sm border border-gray-200 overflow-hidden">
    <!-- Шапка -->
    <div class="px-6 py-4 border-b border-gray-200 bg-gradient-to-r from-blue-50 to-white">
      <div class="flex items-center justify-between">
        <B24User
            v-if="currentUser && showMainContent"
            :name="currentUser.FULL_NAME"
            :description="currentUser.WORK_POSITION || 'Должность не указана'"
            size="lg"
            :avatar="{
              src: getUserPhoto(currentUser),
              initials: getUserInitials(currentUser)
            }"
            :chip="{
              color: currentUser.IS_ONLINE === 'Y' ? 'air-primary-success' : 'air-secondary-accent',
              position: 'top-right'
            }"
            class="truncate overflow-visible"
        />
        <div v-else-if="!showUnavailableMessage" class="flex items-center gap-3">
          <div class="p-2 bg-blue-100 rounded-full">
            <RestaurantIcon class="w-5 h-5 text-blue-600" />
          </div>
          <span class="font-medium text-gray-700">Загрузка...</span>
        </div>

        <B24Button
            v-if="showMainContent"
            @click="refreshData"
            :disabled="isRefreshing"
            variant="ghost"
            size="sm"
            class="p-2 w-9 h-9"
            :class="{ 'animate-spin': isRefreshing }"
            title="Обновить данные"
            :icon="RefreshIcon"
        >
        </B24Button>
      </div>
      <div v-if="currentUser && showMainContent" class="mt-2 text-xs text-gray-500 pl-12">
        {{ currentUser.EMAIL }}
      </div>
    </div>

    <!-- Содержимое -->
    <div class="p-6">
      <!-- Сообщение о недоступности -->
      <div v-if="showUnavailableMessage" class="text-center py-8">
        <div class="mb-4">
          <svg class="w-16 h-16 mx-auto text-gray-400" fill="none" stroke="currentColor" viewBox="0 0 24 24">
            <path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M12 9v2m0 4h.01m-6.938 4h13.856c1.54 0 2.502-1.667 1.732-3L13.732 4c-.77-1.333-2.694-1.333-3.464 0L3.34 16c-.77 1.333.192 3 1.732 3z"/>
          </svg>
        </div>
        <h3 class="text-lg font-semibold text-gray-900 mb-2">
          Функция недоступна
        </h3>
        <p class="text-gray-600 mb-2">
          Для работы с учетом рабочего времени требуется тариф
        </p>
        <p class="text-lg font-bold text-blue-600 mb-4">
          «Профессиональный»
        </p>
        <p class="text-sm text-gray-500">
          Пожалуйста, обратитесь к администратору вашего Битрикс24<br>
          для перехода на соответствующий тарифный план.
        </p>
      </div>

      <!-- Загрузка -->
      <div v-else-if="isLoading && showMainContent" class="flex items-center justify-center py-8">
        <div class="animate-spin rounded-full h-8 w-8 border-b-2 border-blue-600"></div>
      </div>

      <!-- Ошибка -->
      <div v-else-if="error && showMainContent" class="text-center py-6">
        <div class="text-red-600 mb-3 text-sm">{{ error }}</div>
        <B24Button size="sm" @click="refreshData">Попробовать снова</B24Button>
      </div>

      <!-- Данные -->
      <div v-else-if="showMainContent">
        <!-- Статус -->
        <div class="mb-6">
          <div
              class="flex items-center gap-3 p-3 rounded-lg border"
              :class="statusBgColor"
          >
            <div class="p-1 rounded-full bg-white/50">
              <component :is="statusIcon" class="w-5 h-5" :class="statusColor" />
            </div>
            <div class="flex-1">
              <span class="font-medium" :class="statusColor">{{ statusText }}</span>
            </div>
            <B24Badge
                :color="statusChipColor"
                size="sm"
                variant="soft"
            >
              {{ workdayInfo?.STATUS === 'OPENED' ? 'В работе' :
                workdayInfo?.STATUS === 'PAUSED' ? 'На обеде' :
                    workdayInfo?.STATUS === 'EXPIRED' ? 'Просрочен' : 'Ожидание' }}
            </B24Badge>
          </div>
        </div>

        <!-- Настройки времени обеда - с использованием Form -->
        <div class="mb-6 rounded-xl border border-gray-200 bg-white shadow-sm overflow-hidden">
          <div class="px-4 py-3 bg-gradient-to-r from-orange-50 to-amber-50 border-b border-gray-200">
            <h3 class="text-sm font-medium text-gray-700 flex items-center gap-2">
              <ClockIcon class="w-4 h-4 text-amber-600" />
              Настройки времени обеда
            </h3>
          </div

          <div class="p-4">
            <B24Form
                :schema="lunchSchema"
                :state="lunchForm"
                @submit="handleSaveLunchSettings"
            >
              <!-- Группированный блок времени -->
              <div class="flex items-center gap-2 flex-wrap sm:flex-nowrap">
                <!-- Время начала -->
                <div class="min-w-[120px]">
                  <B24FormField name="startTime" label="Начало" class="text-center">
                    <B24InputTime
                        v-model="lunchForm.startTime"
                        :hour-cycle="24"
                        size="xl"
                        color="air-primary"
                        highlight
                    />
                  </B24FormField>
                </div>

                <!-- Разделитель -->
                <div class="text-gray-400 font-medium text-lg">—</div>

                <!-- Время окончания -->
                <div class="min-w-[120px]">
                  <B24FormField name="endTime" label="Окончание" class="text-center">
                    <B24InputTime
                        v-model="lunchForm.endTime"
                        :hour-cycle="24"
                        size="xl"
                        color="air-primary"
                        highlight
                    />
                  </B24FormField>
                </div>
              </div>

              <!-- Отображение выбранного интервала -->
              <div v-if="hasLunchSettings" class="mt-4 text-center">
                <span class="inline-flex items-center gap-1 px-3 py-1 bg-amber-50 text-amber-700 rounded-full text-xs font-medium">
                  <ClockIcon class="w-3 h-3" />
                  {{ formattedLunchStart }} → {{ formattedLunchEnd }}
                </span>
              </div>
              <div v-else class="mt-4 text-center text-xs text-gray-400">
                Выберите время начала и окончания обеда
              </div>

              <!-- Кнопка сохранения -->
              <div class="mt-4 flex justify-center">
                <B24Button
                    type="submit"
                    size="sm"
                    variant="outline"
                    color="air-primary"
                    class="min-w-[120px]"
                >
                  <template #leading>
                    <svg v-if="isSettingsSaved" class="w-4 h-4" fill="none" stroke="currentColor" viewBox="0 0 24 24">
                      <path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M5 13l4 4L19 7" />
                    </svg>
                  </template>
                  {{ isSettingsSaved ? 'Сохранено' : 'Сохранить настройки' }}
                </B24Button>
              </div>
            </B24Form>
          </div>
        </div>

        <!-- Кнопка действия -->
        <div class="flex flex-col gap-2">
          <B24Button
              v-if="canPauseWorkday || canResumeWorkday"
              @click="handleMainAction"
              :disabled="isProcessing"
              :color="buttonColor"
              size="lg"
              class="w-full"
          >
            <template #leading>
              <svg v-if="isProcessing" class="w-4 h-4 animate-spin" fill="none" stroke="currentColor" viewBox="0 0 24 24">
                <path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M4 4v5h.582m15.356 2A8.001 8.001 0 004.582 9m0 0H9m11 11v-5h-.581m0 0a8.003 8.003 0 01-15.357-2m15.357 2H15"/>
              </svg>
              <component :is="buttonIcon" v-else class="w-4 h-4" />
            </template>
            {{ isProcessing ? 'Обработка...' : buttonText }}
          </B24Button>

          <div v-else class="text-center text-sm text-gray-400 py-2">
            {{ workdayInfo?.STATUS === 'CLOSED' ? 'Сначала начните рабочий день' : '' }}
            {{ workdayInfo?.STATUS === 'EXPIRED' ? 'Рабочий день истек, обратитесь к руководителю' : '' }}
          </div>
        </div>
      </div>
    </div>
  </div>
</template>

<style scoped>
.animate-spin {
  animation: spin 1s linear infinite;
}

@keyframes spin {
  from {
    transform: rotate(0deg);
  }
  to {
    transform: rotate(360deg);
  }
}
</style>