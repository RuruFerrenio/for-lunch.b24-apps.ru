<script setup lang="ts">
import { ref, onMounted, onUnmounted, computed, reactive } from 'vue'
import { useToast } from '@bitrix24/b24ui-nuxt/composables/useToast'
import { Time } from '@internationalized/date'
import ClockIcon from '@bitrix24/b24icons-vue/outline/ClockIcon'
import RefreshIcon from '@bitrix24/b24icons-vue/outline/RefreshIcon'
import * as v from 'valibot'
import type { FormSubmitEvent } from '@bitrix24/b24ui-nuxt'

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
const currentUser = ref<UserProfile | null>(null)
const error = ref<string | null>(null)
const isRefreshing = ref(false)

// Настройки обеда для формы
const lunchForm = reactive<LunchFormData>({
  startTime: null,
  endTime: null
})

const isSettingsSaved = ref(false)
const isLoadingSettings = ref(false)

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

// Вычисляемая длительность обеда
const getLunchDuration = computed(() => {
  if (!lunchForm.startTime || !lunchForm.endTime) {
    return '—'
  }

  const [startHours, startMinutes] = lunchForm.startTime.split(':').map(Number)
  const [endHours, endMinutes] = lunchForm.endTime.split(':').map(Number)

  let totalMinutes = (endHours * 60 + endMinutes) - (startHours * 60 + startMinutes)

  // Если время окончания меньше времени начала (переход через полночь)
  if (totalMinutes < 0) {
    totalMinutes += 24 * 60
  }

  const hours = Math.floor(totalMinutes / 60)
  const minutes = totalMinutes % 60

  if (hours === 0) {
    return `${minutes} мин`
  } else if (minutes === 0) {
    return `${hours} ч`
  } else {
    return `${hours} ч ${minutes} мин`
  }
})

// ==========================================================================
// РАБОТА С USER.OPTION.SET/GET
// ==========================================================================

/**
 * Сохраняет настройки обеда через user.option.set
 */
const saveLunchSettingsToUserOptions = async (): Promise<boolean> => {
  if (!isBitrixLoaded.value || typeof BX24 === 'undefined') {
    console.error('BX24 не загружен')
    return false
  }

  return new Promise((resolve) => {
    const options: Record<string, string> = {}

    if (lunchForm.startTime) {
      options['lunch_start_time'] = lunchForm.startTime
    } else {
      options['lunch_start_time'] = ''
    }

    if (lunchForm.endTime) {
      options['lunch_end_time'] = lunchForm.endTime
    } else {
      options['lunch_end_time'] = ''
    }

    BX24.callMethod('user.option.set', {
      options: options
    }, (result: any) => {
      if (result.error()) {
        console.error('Ошибка при сохранении настроек:', result.error())
        resolve(false)
      } else {
        resolve(true)
      }
    })
  })
}

/**
 * Загружает настройки обеда через user.option.get
 */
const loadLunchSettingsFromUserOptions = async (): Promise<void> => {
  if (!isBitrixLoaded.value || typeof BX24 === 'undefined') {
    console.error('BX24 не загружен')
    return
  }

  isLoadingSettings.value = true

  return new Promise((resolve) => {
    BX24.callMethod('user.option.get', {
      names: ['lunch_start_time', 'lunch_end_time']
    }, (result: any) => {
      if (result.error()) {
        console.error('Ошибка при загрузке настроек:', result.error())
        isLoadingSettings.value = false
        resolve()
        return
      }

      const data = result.data()

      if (data && data.lunch_start_time && data.lunch_start_time !== '') {
        lunchForm.startTime = data.lunch_start_time
      }

      if (data && data.lunch_end_time && data.lunch_end_time !== '') {
        lunchForm.endTime = data.lunch_end_time
      }

      isLoadingSettings.value = false
      resolve()
    })
  })
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
const handleSaveLunchSettings = async (event: FormSubmitEvent<LunchSchema>) => {
  const success = await saveLunchSettingsToUserOptions()

  if (success) {
    isSettingsSaved.value = true
    toast.add({
      description: 'Настройки времени обеда сохранены',
      variant: 'success'
    })

    setTimeout(() => {
      isSettingsSaved.value = false
    }, 2000)
  } else {
    toast.add({
      description: 'Ошибка при сохранении настроек',
      variant: 'error'
    })
  }
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

// ==========================================================================
// ОСНОВНЫЕ ДЕЙСТВИЯ
// ==========================================================================

const refreshData = async () => {
  if (isRefreshing.value) return
  if (isTimemanAvailable.value === false) return

  isRefreshing.value = true

  try {
    const user = await getCurrentUserProfile()
    if (user) currentUser.value = user
    error.value = null
  } catch (err) {
    error.value = 'Ошибка при обновлении данных'
  } finally {
    isRefreshing.value = false
    isLoading.value = false
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

onMounted(async () => {
  if (typeof BX24 !== 'undefined' && BX24.init) {
    BX24.init(async () => {
      isBitrixLoaded.value = true
      isTimemanAvailable.value = await checkTimemanAvailability()

      if (!isTimemanAvailable.value) {
        isLoading.value = false
        return
      }

      // Загружаем настройки обеда из user.options
      await loadLunchSettingsFromUserOptions()
      await refreshData()
      startAutoRefresh()
    })
  } else if (typeof BX24 !== 'undefined') {
    isBitrixLoaded.value = true

    checkTimemanAvailability().then(async (isAvailable) => {
      isTimemanAvailable.value = isAvailable

      if (!isAvailable) {
        isLoading.value = false
        return
      }

      // Загружаем настройки обеда из user.options
      await loadLunchSettingsFromUserOptions()
      await refreshData()
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
            <ClockIcon class="w-5 h-5 text-blue-600" />
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
        <!-- Настройки времени обеда - с использованием Form -->
        <div class="rounded-xl border border-gray-200 bg-white shadow-sm overflow-hidden">
          <div class="px-4 py-3 bg-gradient-to-r from-orange-50 to-amber-50 border-b border-gray-200">
            <h3 class="text-sm font-medium text-gray-700 flex items-center gap-2">
              <ClockIcon class="w-4 h-4 text-amber-600" />
              Настройки времени обеда
            </h3>
          </div>

          <div class="p-4">
            <!-- Индикатор загрузки настроек -->
            <div v-if="isLoadingSettings" class="flex justify-center items-center py-4">
              <div class="animate-spin rounded-full h-5 w-5 border-b-2 border-blue-600"></div>
              <span class="ml-2 text-sm text-gray-500">Загрузка настроек...</span>
            </div>

            <B24Form
                v-else
                :schema="lunchSchema"
                :state="lunchForm"
                @submit="handleSaveLunchSettings"
            >
              <!-- Группированный блок времени -->
              <div class="flex items-center justify-center gap-3 flex-wrap sm:flex-nowrap">
                <!-- Время начала -->
                <div class="min-w-[120px]">
                  <B24FormField name="startTime" class="text-center">
                    <B24InputTime
                        :model-value="stringToTime(lunchForm.startTime)"
                        @update:model-value="(val: Time | null) => lunchForm.startTime = timeToString(val)"
                        :hour-cycle="24"
                        size="xl"
                        color="air-primary"
                        highlight
                        tag="Начало"
                    />
                  </B24FormField>
                </div>

                <!-- Разделитель -->
                <div class="text-gray-400 font-medium text-lg">—</div>

                <!-- Время окончания -->
                <div class="min-w-[120px]">
                  <B24FormField name="endTime" class="text-center">
                    <B24InputTime
                        :model-value="stringToTime(lunchForm.endTime)"
                        @update:model-value="(val: Time | null) => lunchForm.endTime = timeToString(val)"
                        :hour-cycle="24"
                        size="xl"
                        color="air-primary"
                        highlight
                        tag="Окончание"
                    />
                  </B24FormField>
                </div>
              </div>

              <!-- Отображение длительности обеда -->
              <div v-if="hasLunchSettings" class="mt-4 text-center">
                <span class="inline-flex items-center gap-1 px-3 py-1 bg-amber-50 text-amber-700 rounded-full text-xs font-medium">
                  <ClockIcon class="w-3 h-3" />
                  Длительность: {{ getLunchDuration }}
                </span>
              </div>
              <div v-else class="mt-4 text-center text-xs text-gray-400">
                Выберите время начала и окончания обеда
              </div>

              <!-- Кнопка сохранения -->
              <div class="mt-4 flex justify-center">
                <B24Button
                    type="submit"
                    size="md"
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