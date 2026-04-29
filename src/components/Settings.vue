<script setup lang="ts">
import { ref, onMounted, watch, computed, reactive } from 'vue'
import { useToast } from '@bitrix24/b24ui-nuxt/composables/useToast'
import { Time } from '@internationalized/date'
import ClockIcon from '@bitrix24/b24icons-vue/outline/ClockIcon'
import * as v from 'valibot'
import type { FormSubmitEvent } from '@bitrix24/b24ui-nuxt'

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

// Форма для времени обеда (для валидации)
const lunchForm = reactive<{
  startTime: string | null
  endTime: string | null
}>({
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
  if (!lunchForm.startTime) return '—'
  return lunchForm.startTime
})

const formattedDefaultLunchEnd = computed(() => {
  if (!lunchForm.endTime) return '—'
  return lunchForm.endTime
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

const hasLunchSettings = computed(() => {
  return lunchForm.startTime !== null && lunchForm.endTime !== null
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

async function saveDefaultLunchTimeSettings(event: FormSubmitEvent<LunchSchema>): Promise<void> {
  try {
    isProcessing.value = true
    if (isBitrixLoaded.value && typeof BX24 !== 'undefined') {
      const startTimeStr = lunchForm.startTime || ''
      const endTimeStr = lunchForm.endTime || ''

      await BX24.appOption.set('lunch_default_start_time', startTimeStr)
      await BX24.appOption.set('lunch_default_end_time', endTimeStr)

      // Также сохраняем в formData для синхронизации
      formData.value.defaultLunchTime.startTime = stringToTime(lunchForm.startTime)
      formData.value.defaultLunchTime.endTime = stringToTime(lunchForm.endTime)
    }

    isSettingsSaved.value = true
    showNotification('success', 'Настройки времени обеда сохранены')

    setTimeout(() => {
      isSettingsSaved.value = false
    }, 2000)
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

function parseTimeFromString(timeStr: string): string | null {
  if (!timeStr || timeStr === '' || timeStr === 'null') return null
  try {
    const [hours, minutes] = timeStr.split(':')
    if (hours && minutes && !isNaN(parseInt(hours)) && !isNaN(parseInt(minutes))) {
      return `${parseInt(hours).toString().padStart(2, '0')}:${parseInt(minutes).toString().padStart(2, '0')}`
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
      // Настройки активности в выходные
      weekendActivityEnabled,

      // Настройки общего времени обеда
      lunchDefaultStartTime,
      lunchDefaultEndTime,

      // Настройки начала обеда
      lunchStartEnabled,
      lunchStartMethod,

      // Настройки завершения обеда
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

    // Загрузка общего времени обеда в обе формы
    const startTimeStr = parseTimeFromString(lunchDefaultStartTime)
    const endTimeStr = parseTimeFromString(lunchDefaultEndTime)

    lunchForm.startTime = startTimeStr
    lunchForm.endTime = endTimeStr

    formData.value.defaultLunchTime.startTime = stringToTime(startTimeStr)
    formData.value.defaultLunchTime.endTime = stringToTime(endTimeStr)

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
  <div>
    <!-- Блок 1: Активность в выходные -->
    <B24Card class="mb-8">
      <div class="p-0 md:p-6">
        <div class="space-y-6">
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
      </div>
    </B24Card>

    <!-- Блок 2: Настройки времени обеда -->
    <B24Card class="mb-8">
      <div class="p-0 md:p-6">
        <div class="space-y-6">
          <div class="flex items-center justify-between">
            <div class="flex-1">
              <h3 class="text-lg font-semibold text-gray-900">
                Общее время обеда
              </h3>
              <p class="text-sm text-gray-500 mt-1">
                Настройка времени обеда по умолчанию для всех сотрудников
              </p>
            </div>
          </div>

          <B24Form
              :schema="lunchSchema"
              :state="lunchForm"
              @submit="saveDefaultLunchTimeSettings"
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

              <!-- Отображение длительности обеда -->
              <div v-if="hasLunchSettings" class="mt-4 text-center">
              <span class="inline-flex items-center gap-1 px-3 py-1 bg-amber-50 text-amber-700 rounded-full text-xs font-medium">
                <ClockIcon class="w-3 h-3" />
                {{ getLunchDuration }}
              </span>
              </div>
              <div v-else class="mt-4 text-center text-xs text-gray-400">
                Выберите время начала и окончания обеда
              </div>

              <!-- Время окончания -->
              <div class="min-w-[120px]">
                <B24FormField name="endTime" class="text-center">
                  <B24InputTime
                      :model-value="stringToTime(lunchForm.endTime)"
                      @update:model-value="(val: Time | null) => lunchForm.endTime = timeToString(val)"
                      :hour-cycle="24"
                      size="sm"
                      color="air-primary"
                      highlight
                      tag="Окончание"
                  />
                </B24FormField>
              </div>
            </div>

            <!-- Кнопка сохранения -->
            <div class="mt-4 flex justify-center">
              <B24Button
                  type="submit"
                  size="md"
                  variant="outline"
                  color="air-primary"
                  class="min-w-[120px]"
                  :disabled="isProcessing"
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
    </B24Card>

    <!-- Блок 3: Помощь в начале обеда -->
    <B24Card class="mb-8">
      <div class="p-0 md:p-6">
        <div class="space-y-6">
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
          <div v-if="formData.lunchStart.enabled" class="space-y-4 pt-4 border-t">
            <B24Form
                :state="formData"
                class="space-y-4"
                @submit="saveLunchStartSettings"
            >
              <!-- Способ начала обеда -->
              <B24FormField
                  label="Способ начала обеда"
                  name="lunchStartMethod"
                  :help-text="`Текущий способ: ${getLunchStartMethodText}`"
              >
                <B24RadioGroup
                    :model-value="formData.lunchStart.method"
                    @update:model-value="(val) => formData.lunchStart.method = val"
                    :disabled="isProcessing"
                    :items="[
                        {
                            label: 'Автоматическое начало обеда',
                            value: 'auto',
                            description: 'Обеденный перерыв начинается автоматически в установленное время'
                        },
                        {
                            label: 'Модальное окно с предложением',
                            value: 'modal',
                            description: 'Показывать окно с предложением начать обед'
                        },
                        {
                            label: 'Сообщение в чате',
                            value: 'chat',
                            description: 'Отправлять уведомление в чат Б24'
                        },
                        {
                            label: 'Push-уведомление',
                            value: 'push',
                            description: 'Отправлять push-уведомление'
                        }
                    ]"
                    orientation="horizontal"
                    variant="card"
                    size="sm"
                    default-value="modal"
                    indicator="end"
                    class="overflow-scroll md:overflow-auto"
                />
              </B24FormField>
            </B24Form>

            <!-- Информация о системе помощи в начале обеда -->
            <div class="space-y-4 mt-6">
              <h4 class="text-sm font-medium text-gray-900">
                Как работает помощь в начале обеда
              </h4>
              <div class="space-y-3">
                <div class="flex items-start">
                  <div class="w-6 h-6 bg-blue-100 rounded-full flex items-center justify-center mr-3 flex-shrink-0">
                    <span class="text-xs font-medium text-blue-600">1</span>
                  </div>
                  <div>
                    <p class="text-sm text-gray-700">
                      Система отслеживает текущее время и статус рабочего дня сотрудника.
                    </p>
                  </div>
                </div>
                <div class="flex items-start">
                  <div class="w-6 h-6 bg-blue-100 rounded-full flex items-center justify-center mr-3 flex-shrink-0">
                    <span class="text-xs font-medium text-blue-600">2</span>
                  </div>
                  <div>
                    <p class="text-sm text-gray-700">
                      <span class="font-medium">Автоматическое начало обеда:</span> обеденный перерыв начинается автоматически без участия сотрудника.
                    </p>
                  </div>
                </div>
                <div class="flex items-start">
                  <div class="w-6 h-6 bg-blue-100 rounded-full flex items-center justify-center mr-3 flex-shrink-0">
                    <span class="text-xs font-medium text-blue-600">3</span>
                  </div>
                  <div>
                    <p class="text-sm text-gray-700">
                      <span class="font-medium">Модальное окно:</span> показывается окно с кнопкой "На обед!", пока сотрудник не начнет перерыв.
                    </p>
                  </div>
                </div>
                <div class="flex items-start">
                  <div class="w-6 h-6 bg-blue-100 rounded-full flex items-center justify-center mr-3 flex-shrink-0">
                    <span class="text-xs font-medium text-blue-600">4</span>
                  </div>
                  <div>
                    <p class="text-sm text-gray-700">
                      <span class="font-medium">Сообщение в чате:</span> в чат Битрикс24 отправляется уведомление с предложением начать обед.
                    </p>
                  </div>
                </div>
                <div class="flex items-start">
                  <div class="w-6 h-6 bg-blue-100 rounded-full flex items-center justify-center mr-3 flex-shrink-0">
                    <span class="text-xs font-medium text-blue-600">5</span>
                  </div>
                  <div>
                    <p class="text-sm text-gray-700">
                      <span class="font-medium">Push-уведомление:</span> сотруднику отправляется push-уведомление в Битрикс24.
                    </p>
                  </div>
                </div>
                <div class="flex items-start mt-2 p-3 bg-yellow-50 rounded-lg">
                  <svg class="w-5 h-5 text-yellow-500 mr-2 flex-shrink-0 mt-0.5" fill="none" stroke="currentColor" viewBox="0 0 24 24">
                    <path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M12 9v2m0 4h.01m-6.938 4h13.856c1.54 0 2.502-1.667 1.732-2.5L13.732 4c-.77-.833-1.964-.833-2.732 0L4.346 16.5c-.77.833.192 2.5 1.732 2.5z"/>
                  </svg>
                  <div class="text-sm text-yellow-700">
                    <span class="font-medium">Функция доступна на тарифах "Профессиональный" и выше.</span>
                  </div>
                </div>
              </div>
            </div>
          </div>
        </div>
      </div>
    </B24Card>

    <!-- Блок 4: Помощь в завершении обеда -->
    <B24Card class="mb-8">
      <div class="p-0 md:p-6">
        <div class="space-y-6">
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
          <div v-if="formData.lunchEnd.enabled" class="space-y-4 pt-4 border-t">
            <B24Form
                :state="formData"
                class="space-y-4"
                @submit="saveLunchEndSettings"
            >
              <!-- Способ завершения обеда -->
              <B24FormField
                  label="Способ завершения обеда"
                  name="lunchEndMethod"
                  :help-text="`Текущий способ: ${getLunchEndMethodText}`"
              >
                <B24RadioGroup
                    :model-value="formData.lunchEnd.method"
                    @update:model-value="(val) => formData.lunchEnd.method = val"
                    :disabled="isProcessing"
                    :items="[
                        {
                            label: 'Автоматическое завершение обеда',
                            value: 'auto',
                            description: 'Обеденный перерыв завершается автоматически в установленное время'
                        },
                        {
                            label: 'Модальное окно с предложением',
                            value: 'modal',
                            description: 'Показывать окно с предложением завершить обед'
                        },
                        {
                            label: 'Сообщение в чате',
                            value: 'chat',
                            description: 'Отправлять уведомление в чат Б24'
                        },
                        {
                            label: 'Push-уведомление',
                            value: 'push',
                            description: 'Отправлять push-уведомление'
                        }
                    ]"
                    orientation="horizontal"
                    variant="card"
                    size="sm"
                    default-value="modal"
                    indicator="end"
                    class="overflow-scroll md:overflow-auto"
                />
              </B24FormField>
            </B24Form>

            <!-- Информация о системе помощи в завершении обеда -->
            <div class="space-y-4 mt-6">
              <h4 class="text-sm font-medium text-gray-900">
                Как работает помощь в завершении обеда
              </h4>
              <div class="space-y-3">
                <div class="flex items-start">
                  <div class="w-6 h-6 bg-blue-100 rounded-full flex items-center justify-center mr-3 flex-shrink-0">
                    <span class="text-xs font-medium text-blue-600">1</span>
                  </div>
                  <div>
                    <p class="text-sm text-gray-700">
                      Система отслеживает текущее время и статус обеденного перерыва сотрудника.
                    </p>
                  </div>
                </div>
                <div class="flex items-start">
                  <div class="w-6 h-6 bg-blue-100 rounded-full flex items-center justify-center mr-3 flex-shrink-0">
                    <span class="text-xs font-medium text-blue-600">2</span>
                  </div>
                  <div>
                    <p class="text-sm text-gray-700">
                      <span class="font-medium">Автоматическое завершение обеда:</span> обеденный перерыв завершается автоматически без участия сотрудника.
                    </p>
                  </div>
                </div>
                <div class="flex items-start">
                  <div class="w-6 h-6 bg-blue-100 rounded-full flex items-center justify-center mr-3 flex-shrink-0">
                    <span class="text-xs font-medium text-blue-600">3</span>
                  </div>
                  <div>
                    <p class="text-sm text-gray-700">
                      <span class="font-medium">Модальное окно:</span> показывается окно с кнопкой "За работу!", пока сотрудник не завершит обед.
                    </p>
                  </div>
                </div>
                <div class="flex items-start">
                  <div class="w-6 h-6 bg-blue-100 rounded-full flex items-center justify-center mr-3 flex-shrink-0">
                    <span class="text-xs font-medium text-blue-600">4</span>
                  </div>
                  <div>
                    <p class="text-sm text-gray-700">
                      <span class="font-medium">Сообщение в чате:</span> в чат Битрикс24 отправляется уведомление с предложением завершить обед.
                    </p>
                  </div>
                </div>
                <div class="flex items-start">
                  <div class="w-6 h-6 bg-blue-100 rounded-full flex items-center justify-center mr-3 flex-shrink-0">
                    <span class="text-xs font-medium text-blue-600">5</span>
                  </div>
                  <div>
                    <p class="text-sm text-gray-700">
                      <span class="font-medium">Push-уведомление:</span> сотруднику отправляется push-уведомление в Битрикс24.
                    </p>
                  </div>
                </div>
                <div class="flex items-start mt-2 p-3 bg-yellow-50 rounded-lg">
                  <svg class="w-5 h-5 text-yellow-500 mr-2 flex-shrink-0 mt-0.5" fill="none" stroke="currentColor" viewBox="0 0 24 24">
                    <path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M12 9v2m0 4h.01m-6.938 4h13.856c1.54 0 2.502-1.667 1.732-2.5L13.732 4c-.77-.833-1.964-.833-2.732 0L4.346 16.5c-.77.833.192 2.5 1.732 2.5z"/>
                  </svg>
                  <div class="text-sm text-yellow-700">
                    <span class="font-medium">Функция доступна на тарифах "Профессиональный" и выше.</span>
                  </div>
                </div>
              </div>
            </div>
          </div>
        </div>
      </div>
    </B24Card>
  </div>
</template>