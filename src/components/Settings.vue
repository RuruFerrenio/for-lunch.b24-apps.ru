<script setup lang="ts">
import { ref, onMounted, watch, computed } from 'vue'
import { useToast } from '@bitrix24/b24ui-nuxt/composables/useToast'
import { Time } from '@internationalized/date'

const toast = useToast()

// Типы данных
interface LunchSettings {
  enabled: boolean
  method: 'auto' | 'modal' | 'chat' | 'push'
  startTime: Time | null
  endTime: Time | null
}

interface FormData {
  lunchStart: LunchSettings
  lunchEnd: LunchSettings
  weekendActivity: {
    enabled: boolean
  }
  defaultLunchTime: {
    startTime: Time | null
    endTime: Time | null
  }
}

// Инициализация formData
const formData = ref<FormData>({
  lunchStart: {
    enabled: false,
    method: 'modal',
    startTime: null,
    endTime: null
  },
  lunchEnd: {
    enabled: false,
    method: 'modal',
    startTime: null,
    endTime: null
  },
  weekendActivity: {
    enabled: false
  },
  defaultLunchTime: {
    startTime: null,
    endTime: null
  }
})

const isProcessing = ref(false)
const isBitrixLoaded = ref(false)

// Вычисляемые свойства для отображения текста
const getLunchStartMethodText = computed(() => {
  const methods: Record<'auto' | 'modal' | 'chat' | 'push', string> = {
    'auto': 'Автоматическое начало обеда',
    'modal': 'Модальное окно с предложением',
    'chat': 'Сообщение в чате',
    'push': 'Push-уведомление'
  }
  return methods[formData.value.lunchStart.method] || 'модальное окно'
})

const getLunchEndMethodText = computed(() => {
  const methods: Record<'auto' | 'modal' | 'chat' | 'push', string> = {
    'auto': 'Автоматическое завершение обеда',
    'modal': 'Модальное окно с предложением',
    'chat': 'Сообщение в чате',
    'push': 'Push-уведомление'
  }
  return methods[formData.value.lunchEnd.method] || 'модальное окно'
})

const formattedDefaultLunchStart = computed(() => {
  if (!formData.value.defaultLunchTime.startTime) return '—'
  const time = formData.value.defaultLunchTime.startTime
  return `${time.hour.toString().padStart(2, '0')}:${time.minute.toString().padStart(2, '0')}`
})

const formattedDefaultLunchEnd = computed(() => {
  if (!formData.value.defaultLunchTime.endTime) return '—'
  const time = formData.value.defaultLunchTime.endTime
  return `${time.hour.toString().padStart(2, '0')}:${time.minute.toString().padStart(2, '0')}`
})

// Вычисляемая длительность обеда
const getLunchDuration = computed(() => {
  const start = formData.value.defaultLunchTime.startTime
  const end = formData.value.defaultLunchTime.endTime

  if (!start || !end) return '—'

  let totalMinutes = (end.hour * 60 + end.minute) - (start.hour * 60 + start.minute)

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

// Функции для уведомлений
function showNotification(type: 'success' | 'error' | 'warning' | 'info', message: string): void {
  if (typeof toast !== 'undefined') {
    toast.add({
      description: message,
      variant: type
    })
  }
}

// ==========================================================================
// НАСТРОЙКИ АКТИВНОСТИ В ВЫХОДНЫЕ
// ==========================================================================

async function toggleWeekendActivity(newValue: boolean): Promise<void> {
  try {
    isProcessing.value = true
    if (isBitrixLoaded.value && typeof BX24 !== 'undefined') {
      await BX24.appOption.set('lunch_weekend_activity_enabled', newValue ? 'Y' : 'N')
    }
    formData.value.weekendActivity.enabled = newValue
    showNotification('success', newValue ? 'Активность в выходные включена' : 'Активность в выходные выключена')
  } catch {
    showNotification('error', 'Ошибка сохранения настройки')
  } finally {
    isProcessing.value = false
  }
}

// ==========================================================================
// НАСТРОЙКИ ОБЩЕГО ВРЕМЕНИ ОБЕДА
// ==========================================================================

async function saveDefaultLunchTimeSettings(): Promise<void> {
  try {
    isProcessing.value = true
    if (isBitrixLoaded.value && typeof BX24 !== 'undefined') {
      const startTimeStr = formData.value.defaultLunchTime.startTime
          ? `${formData.value.defaultLunchTime.startTime.hour.toString().padStart(2, '0')}:${formData.value.defaultLunchTime.startTime.minute.toString().padStart(2, '0')}`
          : ''
      const endTimeStr = formData.value.defaultLunchTime.endTime
          ? `${formData.value.defaultLunchTime.endTime.hour.toString().padStart(2, '0')}:${formData.value.defaultLunchTime.endTime.minute.toString().padStart(2, '0')}`
          : ''

      await BX24.appOption.set('lunch_default_start_time', startTimeStr)
      await BX24.appOption.set('lunch_default_end_time', endTimeStr)
    }
    showNotification('success', 'Настройки времени обеда сохранены')
  } catch {
    showNotification('error', 'Ошибка сохранения настроек времени обеда')
  } finally {
    isProcessing.value = false
  }
}

// ==========================================================================
// НАСТРОЙКИ ПОМОЩИ В НАЧАЛЕ ОБЕДА
// ==========================================================================

async function toggleLunchStart(newValue: boolean): Promise<void> {
  try {
    isProcessing.value = true
    if (isBitrixLoaded.value && typeof BX24 !== 'undefined') {
      await BX24.appOption.set('lunch_start_enabled', newValue ? 'Y' : 'N')
    }
    formData.value.lunchStart.enabled = newValue
    showNotification('success', newValue ? 'Помощь в начале обеда включена' : 'Помощь в начале обеда выключена')
  } catch {
    showNotification('error', 'Ошибка сохранения настройки')
  } finally {
    isProcessing.value = false
  }
}

async function updateLunchStartMethod(): Promise<void> {
  try {
    isProcessing.value = true
    if (isBitrixLoaded.value && typeof BX24 !== 'undefined') {
      await BX24.appOption.set('lunch_start_method', formData.value.lunchStart.method)
    }
    showNotification('success', `Способ начала обеда: ${getLunchStartMethodText.value}`)
  } catch {
    showNotification('error', 'Ошибка сохранения способа начала обеда')
  } finally {
    isProcessing.value = false
  }
}

async function saveLunchStartSettings(): Promise<void> {
  try {
    isProcessing.value = true
    if (isBitrixLoaded.value && typeof BX24 !== 'undefined') {
      await BX24.appOption.set('lunch_start_enabled', formData.value.lunchStart.enabled ? 'Y' : 'N')
      await BX24.appOption.set('lunch_start_method', formData.value.lunchStart.method)
    }
    showNotification('success', 'Настройки помощи в начале обеда сохранены')
  } catch {
    showNotification('error', 'Ошибка сохранения настроек')
  } finally {
    isProcessing.value = false
  }
}

// ==========================================================================
// НАСТРОЙКИ ПОМОЩИ В ЗАВЕРШЕНИИ ОБЕДА
// ==========================================================================

async function toggleLunchEnd(newValue: boolean): Promise<void> {
  try {
    isProcessing.value = true
    if (isBitrixLoaded.value && typeof BX24 !== 'undefined') {
      await BX24.appOption.set('lunch_end_enabled', newValue ? 'Y' : 'N')
    }
    formData.value.lunchEnd.enabled = newValue
    showNotification('success', newValue ? 'Помощь в завершении обеда включена' : 'Помощь в завершении обеда выключена')
  } catch {
    showNotification('error', 'Ошибка сохранения настройки')
  } finally {
    isProcessing.value = false
  }
}

async function updateLunchEndMethod(): Promise<void> {
  try {
    isProcessing.value = true
    if (isBitrixLoaded.value && typeof BX24 !== 'undefined') {
      await BX24.appOption.set('lunch_end_method', formData.value.lunchEnd.method)
    }
    showNotification('success', `Способ завершения обеда: ${getLunchEndMethodText.value}`)
  } catch {
    showNotification('error', 'Ошибка сохранения способа завершения обеда')
  } finally {
    isProcessing.value = false
  }
}

async function saveLunchEndSettings(): Promise<void> {
  try {
    isProcessing.value = true
    if (isBitrixLoaded.value && typeof BX24 !== 'undefined') {
      await BX24.appOption.set('lunch_end_enabled', formData.value.lunchEnd.enabled ? 'Y' : 'N')
      await BX24.appOption.set('lunch_end_method', formData.value.lunchEnd.method)
    }
    showNotification('success', 'Настройки помощи в завершении обеда сохранены')
  } catch {
    showNotification('error', 'Ошибка сохранения настроек')
  } finally {
    isProcessing.value = false
  }
}

// ==========================================================================
// ЗАГРУЗКА НАСТРОЕК
// ==========================================================================

function normalizeBoolean(value: unknown): boolean {
  return value === 'Y' || value === true || value === 1
}

function normalizeMethod(value: unknown, validMethods: readonly string[]): 'auto' | 'modal' | 'chat' | 'push' | null {
  if (typeof value === 'string' && validMethods.includes(value)) {
    return value as 'auto' | 'modal' | 'chat' | 'push'
  }
  return null
}

function parseTimeFromString(timeStr: string): Time | null {
  if (!timeStr || timeStr === '' || timeStr === 'null') return null
  try {
    const [hours, minutes] = timeStr.split(':')
    if (hours && minutes && !isNaN(parseInt(hours)) && !isNaN(parseInt(minutes))) {
      return new Time(parseInt(hours), parseInt(minutes), 0)
    }
    return null
  } catch (e) {
    console.error('Ошибка парсинга времени:', e)
    return null
  }
}

async function loadSettings(): Promise<void> {
  if (!isBitrixLoaded.value || typeof BX24 === 'undefined') return

  try {
    const [
      weekendActivityEnabled,
      lunchDefaultStartTime,
      lunchDefaultEndTime,
      lunchStartEnabled,
      lunchStartMethod,
      lunchEndEnabled,
      lunchEndMethod
    ] = await Promise.all([
      BX24.appOption.get('lunch_weekend_activity_enabled'),
      BX24.appOption.get('lunch_default_start_time'),
      BX24.appOption.get('lunch_default_end_time'),
      BX24.appOption.get('lunch_start_enabled'),
      BX24.appOption.get('lunch_start_method'),
      BX24.appOption.get('lunch_end_enabled'),
      BX24.appOption.get('lunch_end_method')
    ])

    // Загрузка настроек активности в выходные
    formData.value.weekendActivity.enabled = normalizeBoolean(weekendActivityEnabled)

    // Загрузка общего времени обеда
    const startTime = parseTimeFromString(lunchDefaultStartTime)
    const endTime = parseTimeFromString(lunchDefaultEndTime)

    formData.value.defaultLunchTime.startTime = startTime
    formData.value.defaultLunchTime.endTime = endTime

    // Загрузка настроек начала обеда
    formData.value.lunchStart.enabled = normalizeBoolean(lunchStartEnabled)
    const startMethod = normalizeMethod(lunchStartMethod, ['auto', 'modal', 'chat', 'push'])
    if (startMethod) {
      formData.value.lunchStart.method = startMethod
    }

    // Загрузка настроек завершения обеда
    formData.value.lunchEnd.enabled = normalizeBoolean(lunchEndEnabled)
    const endMethod = normalizeMethod(lunchEndMethod, ['auto', 'modal', 'chat', 'push'])
    if (endMethod) {
      formData.value.lunchEnd.method = endMethod
    }

  } catch (error) {
    console.error('Ошибка загрузки настроек:', error)
  }
}

// ==========================================================================
// ЖИЗНЕННЫЙ ЦИКЛ И НАБЛЮДАТЕЛИ
// ==========================================================================

onMounted(async () => {
  if (typeof BX24 !== 'undefined' && BX24.init) {
    BX24.init(async () => {
      isBitrixLoaded.value = true
      await loadSettings()
    })
  }
})

// Наблюдатели для автоматического сохранения при изменении метода
watch(() => formData.value.lunchStart.method, () => {
  if (isProcessing.value) return
  updateLunchStartMethod()
})

watch(() => formData.value.lunchEnd.method, () => {
  if (isProcessing.value) return
  updateLunchEndMethod()
})
</script>

<template>
  <div class="space-y-6">
    <!-- Блок 1: Активность в выходные -->
    <B24Card class="overflow-hidden">
      <div class="p-6">
        <div class="flex items-center justify-between">
          <div class="flex-1">
            <h3 class="text-lg font-semibold text-gray-900">
              Активность в выходные
            </h3>
            <p class="text-sm text-gray-500 mt-1">
              Разрешить работу приложения для управления обедом в выходные дни
            </p>
          </div>
          <div class="ml-4 flex items-center space-x-4">
            <div class="w-2 h-2 rounded-full"
                 :class="formData.weekendActivity.enabled ? 'bg-green-500' : 'bg-red-500'"></div>
            <B24Switch
                :model-value="formData.weekendActivity.enabled"
                @update:model-value="toggleWeekendActivity"
                :disabled="isProcessing"
            />
          </div>
        </div>
      </div>
    </B24Card>

    <!-- Блок 2: Настройки времени обеда (НОВЫЙ ДИЗАЙН) -->
    <B24Card class="overflow-hidden">
      <div class="p-6">
        <div class="mb-6">
          <h3 class="text-lg font-semibold text-gray-900">
            Время обеда по умолчанию
          </h3>
          <p class="text-sm text-gray-500 mt-1">
            Настройка времени обеда для всех сотрудников
          </p>
        </div>

        <!-- Визуальный индикатор времени -->
        <div class="mb-6 p-4 bg-gradient-to-r from-blue-50 to-purple-50 rounded-xl">
          <div class="flex items-center justify-between flex-wrap gap-4">
            <div class="flex-1 text-center">
              <div class="text-xs text-gray-500 mb-1">Начало обеда</div>
              <div class="text-2xl font-bold text-gray-900">
                {{ formattedDefaultLunchStart }}
              </div>
            </div>
            <div class="text-gray-400 text-xl">→</div>
            <div class="flex-1 text-center">
              <div class="text-xs text-gray-500 mb-1">Окончание обеда</div>
              <div class="text-2xl font-bold text-gray-900">
                {{ formattedDefaultLunchEnd }}
              </div>
            </div>
            <div class="flex-1 text-center">
              <div class="text-xs text-gray-500 mb-1">Длительность</div>
              <div class="text-lg font-semibold text-blue-600">
                {{ getLunchDuration }}
              </div>
            </div>
          </div>
        </div>

        <!-- Ползунки времени -->
        <div class="space-y-6 mb-6">
          <div>
            <B24TimeSlider
                v-model="formData.defaultLunchTime.startTime"
                label="Начало обеда"
                :hour-cycle="24"
                step-minutes="15"
                size="md"
                show-time-label
            />
          </div>
          <div>
            <B24TimeSlider
                v-model="formData.defaultLunchTime.endTime"
                label="Окончание обеда"
                :hour-cycle="24"
                step-minutes="15"
                size="md"
                show-time-label
            />
          </div>
        </div>

        <!-- Кнопка сохранения -->
        <div class="flex items-center justify-between pt-4 border-t">
          <div class="flex items-center text-sm text-gray-500">
            <svg class="w-4 h-4 mr-1 text-blue-500" fill="none" stroke="currentColor" viewBox="0 0 24 24">
              <path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M13 16h-1v-4h-1m1-4h.01M21 12a9 9 0 11-18 0 9 9 0 0118 0z"/>
            </svg>
            <span>Используется для автоматических напоминаний</span>
          </div>
          <B24Button
              @click="saveDefaultLunchTimeSettings"
              :disabled="isProcessing"
              size="md"
              variant="outline"
              color="air-primary"
          >
            <template #leading>
              <svg v-if="!isProcessing" class="w-4 h-4" fill="none" stroke="currentColor" viewBox="0 0 24 24">
                <path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M5 13l4 4L19 7"/>
              </svg>
              <div v-else class="w-4 h-4 border-2 border-white border-t-transparent rounded-full animate-spin"></div>
            </template>
            Сохранить
          </B24Button>
        </div>
      </div>
    </B24Card>

    <!-- Блок 3: Помощь в начале обеда -->
    <B24Card class="overflow-hidden">
      <div class="p-6">
        <div class="flex items-center justify-between">
          <div class="flex-1">
            <h3 class="text-lg font-semibold text-gray-900">
              Помощь в начале обеда
            </h3>
            <p class="text-sm text-gray-500 mt-1">
              Автоматическая помощь сотрудникам в своевременном начале обеденного перерыва
            </p>
          </div>
          <div class="ml-4 flex items-center space-x-4">
            <div class="w-2 h-2 rounded-full"
                 :class="formData.lunchStart.enabled ? 'bg-green-500' : 'bg-red-500'"></div>
            <B24Switch
                :model-value="formData.lunchStart.enabled"
                @update:model-value="toggleLunchStart"
                :disabled="isProcessing"
            />
          </div>
        </div>

        <!-- Настройки помощи в начале обеда -->
        <div v-if="formData.lunchStart.enabled" class="mt-6 pt-6 border-t">
          <B24Form
              :state="formData"
              class="space-y-4"
              @submit="saveLunchStartSettings"
          >
            <B24FormField
                label="Способ начала обеда"
                name="lunchStartMethod"
                :help-text="`Текущий способ: ${getLunchStartMethodText}`"
            >
              <div class="grid grid-cols-1 sm:grid-cols-2 lg:grid-cols-4 gap-3">
                <div
                    v-for="item in [
                        { label: 'Автоматически', value: 'auto', icon: '⚡', description: 'Автоматическое начало обеда' },
                        { label: 'Модальное окно', value: 'modal', icon: '🪟', description: 'Показывать окно с предложением' },
                        { label: 'Сообщение в чате', value: 'chat', icon: '💬', description: 'Отправлять в чат Б24' },
                        { label: 'Push-уведомление', value: 'push', icon: '📱', description: 'Push-уведомление' }
                    ]"
                    :key="item.value"
                    class="relative"
                >
                  <input
                      type="radio"
                      :id="`start-method-${item.value}`"
                      :value="item.value"
                      v-model="formData.lunchStart.method"
                      class="sr-only peer"
                  />
                  <label
                      :for="`start-method-${item.value}`"
                      class="flex flex-col items-center p-4 bg-white border-2 rounded-xl cursor-pointer transition-all peer-checked:border-blue-500 peer-checked:bg-blue-50 hover:border-blue-300"
                  >
                    <span class="text-2xl mb-2">{{ item.icon }}</span>
                    <span class="font-medium text-gray-900 text-sm">{{ item.label }}</span>
                    <span class="text-xs text-gray-500 mt-1 text-center">{{ item.description }}</span>
                  </label>
                </div>
              </div>
            </B24FormField>
          </B24Form>

          <div class="mt-4 p-3 bg-blue-50 rounded-lg">
            <div class="flex items-start">
              <svg class="w-5 h-5 text-blue-500 mr-2 flex-shrink-0 mt-0.5" fill="none" stroke="currentColor" viewBox="0 0 24 24">
                <path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M13 16h-1v-4h-1m1-4h.01M21 12a9 9 0 11-18 0 9 9 0 0118 0z"/>
              </svg>
              <div class="text-sm text-blue-700">
                <span class="font-medium">Функция доступна на тарифах "Профессиональный" и выше.</span>
              </div>
            </div>
          </div>
        </div>
      </div>
    </B24Card>

    <!-- Блок 4: Помощь в завершении обеда -->
    <B24Card class="overflow-hidden">
      <div class="p-6">
        <div class="flex items-center justify-between">
          <div class="flex-1">
            <h3 class="text-lg font-semibold text-gray-900">
              Помощь в завершении обеда
            </h3>
            <p class="text-sm text-gray-500 mt-1">
              Автоматическая помощь сотрудникам в своевременном завершении обеденного перерыва
            </p>
          </div>
          <div class="ml-4 flex items-center space-x-4">
            <div class="w-2 h-2 rounded-full"
                 :class="formData.lunchEnd.enabled ? 'bg-green-500' : 'bg-red-500'"></div>
            <B24Switch
                :model-value="formData.lunchEnd.enabled"
                @update:model-value="toggleLunchEnd"
                :disabled="isProcessing"
            />
          </div>
        </div>

        <!-- Настройки помощи в завершении обеда -->
        <div v-if="formData.lunchEnd.enabled" class="mt-6 pt-6 border-t">
          <B24Form
              :state="formData"
              class="space-y-4"
              @submit="saveLunchEndSettings"
          >
            <B24FormField
                label="Способ завершения обеда"
                name="lunchEndMethod"
                :help-text="`Текущий способ: ${getLunchEndMethodText}`"
            >
              <div class="grid grid-cols-1 sm:grid-cols-2 lg:grid-cols-4 gap-3">
                <div
                    v-for="item in [
                        { label: 'Автоматически', value: 'auto', icon: '⚡', description: 'Автоматическое завершение обеда' },
                        { label: 'Модальное окно', value: 'modal', icon: '🪟', description: 'Показывать окно с предложением' },
                        { label: 'Сообщение в чате', value: 'chat', icon: '💬', description: 'Отправлять в чат Б24' },
                        { label: 'Push-уведомление', value: 'push', icon: '📱', description: 'Push-уведомление' }
                    ]"
                    :key="item.value"
                    class="relative"
                >
                  <input
                      type="radio"
                      :id="`end-method-${item.value}`"
                      :value="item.value"
                      v-model="formData.lunchEnd.method"
                      class="sr-only peer"
                  />
                  <label
                      :for="`end-method-${item.value}`"
                      class="flex flex-col items-center p-4 bg-white border-2 rounded-xl cursor-pointer transition-all peer-checked:border-blue-500 peer-checked:bg-blue-50 hover:border-blue-300"
                  >
                    <span class="text-2xl mb-2">{{ item.icon }}</span>
                    <span class="font-medium text-gray-900 text-sm">{{ item.label }}</span>
                    <span class="text-xs text-gray-500 mt-1 text-center">{{ item.description }}</span>
                  </label>
                </div>
              </div>
            </B24FormField>
          </B24Form>

          <div class="mt-4 p-3 bg-blue-50 rounded-lg">
            <div class="flex items-start">
              <svg class="w-5 h-5 text-blue-500 mr-2 flex-shrink-0 mt-0.5" fill="none" stroke="currentColor" viewBox="0 0 24 24">
                <path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M13 16h-1v-4h-1m1-4h.01M21 12a9 9 0 11-18 0 9 9 0 0118 0z"/>
              </svg>
              <div class="text-sm text-blue-700">
                <span class="font-medium">Функция доступна на тарифах "Профессиональный" и выше.</span>
              </div>
            </div>
          </div>
        </div>
      </div>
    </B24Card>
  </div>
</template>