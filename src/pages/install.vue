<script setup lang="ts">
import { ref, computed, onUnmounted } from 'vue'
import ArrowRightLIcon from '@bitrix24/b24icons-vue/outline/ArrowRightLIcon'
import ArrowLeftLIcon from '@bitrix24/b24icons-vue/outline/ArrowLeftLIcon'
import CheckIcon from '@bitrix24/b24icons-vue/main/CheckIcon'
import SettingsIcon from '@bitrix24/b24icons-vue/main/SettingsIcon'
import ShieldIcon from '@bitrix24/b24icons-vue/main/ShieldIcon'
import RefreshIcon from '@bitrix24/b24icons-vue/main/RefreshIcon'
import SuccessIcon from '@bitrix24/b24icons-vue/button/SuccessIcon'
import ErrorIcon from '@bitrix24/b24icons-vue/main/UnavailableIcon'
import LoadingIcon from '@bitrix24/b24icons-vue/animated/LoaderWaitIcon'
import CoffeeIcon from '@bitrix24/b24icons-vue/outline/PowerIcon'
import { Time } from '@internationalized/date'

// Состояние
const currentStep = ref(1)
const totalSteps = 4
const progress = computed(() => Math.round((currentStep.value / totalSteps) * 100))

const autoFinishTimer = ref(null)
const secondsLeft = ref(5)
const isTimerActive = ref(false)

// Статус установки
const isInstalling = ref(false)
const installationComplete = ref(false)
const installedCount = ref(0)
const placementStatus = ref({
  pageBackgroundWorker: null,
  restAppUri: null
})
const settingsStatus = ref(null)

// Выбранные функции
const selectedFeatures = ref({
  lunchStart: true,    // Помощь в начале обеда - по умолчанию включена
  lunchEnd: true,      // Помощь в завершении обеда - по умолчанию включена
  weekendActivity: false // Активность в выходные - по умолчанию выключена
})

// Расширенные настройки (методы)
const configSettings = ref({
  lunchStart: {
    enabled: true,
    method: 'modal'  // По умолчанию модальное окно
  },
  lunchEnd: {
    enabled: true,
    method: 'modal'  // По умолчанию модальное окно
  },
  weekendActivity: {
    enabled: false    // По умолчанию активность в выходные выключена
  },
  defaultLunchTime: {
    startTime: '12:00',  // Время начала обеда по умолчанию
    endTime: '13:00'     // Время завершения обеда по умолчанию
  }
})

// URL обработчиков
const HANDLERS = {
  pageBackgroundWorker: `${window.location.origin}/dist/widgets/background-handler`,
  restAppUri: `${window.location.origin}/dist/`
}

// Конфигурации встроек
const PLACEMENT_CONFIGS = {
  PAGE_BACKGROUND_WORKER: {
    title: 'Фоновая встройка',
    description: 'Автоматически определяет время начала и завершения обеденного перерыва и оповещает об этом сотрудника',
    options: {
      errorHandlerUrl: `${window.location.origin}/dist/widgets/background-error-handler`
    }
  },
  REST_APP_URI: {
    title: 'Встройка для управления обедом из уведомлений',
    description: 'Позволяет сотруднику управлять обеденным перерывом через сообщения в чатах или push-уведомлениях',
    options: {}
  }
}

// Функции для работы с Bitrix24 API
const bitrixAPI = {
  call: (method, params) => {
    return new Promise((resolve, reject) => {
      if (typeof BX24 === 'undefined' || typeof BX24.callMethod === 'undefined') {
        reject(new Error('Библиотека Bitrix24 не доступна'))
        return
      }

      BX24.callMethod(method, params, (result) => {
        if (result.error()) {
          reject(new Error(result.error().getError()))
        } else {
          resolve(result.data())
        }
      })
    })
  },

  installFinish: () => {
    return new Promise((resolve, reject) => {
      if (typeof BX24 === 'undefined') {
        reject(new Error('Библиотека Bitrix24 не доступна'))
        return
      }

      BX24.installFinish()
      resolve()
    })
  },

  setAppOption: async (key, value) => {
    try {
      const result = await bitrixAPI.call('app.option.set', {
        options: { [key]: value }
      })
      return result
    } catch (error) {
      throw error
    }
  },

  getPlacements: () => {
    return new Promise((resolve, reject) => {
      BX24.callMethod('placement.list', {}, (result) => {
        if (result.error()) {
          reject(new Error(result.error().getError()))
        } else {
          resolve(result.data())
        }
      })
    })
  }
}

// Менеджер встроек
const placementManager = {
  getConfig: (placementType) => {
    const config = PLACEMENT_CONFIGS[placementType]
    if (!config) {
      throw new Error(`Неизвестный тип встройки: ${placementType}`)
    }
    return config
  },

  unbind: async (placementType, handler) => {
    try {
      await bitrixAPI.call('placement.unbind', {
        PLACEMENT: placementType,
        HANDLER: handler
      })
      return true
    } catch (error) {
      return null
    }
  },

  bind: async (placementType, handler) => {
    try {
      const config = placementManager.getConfig(placementType)

      const placementConfig = placementType === 'PAGE_BACKGROUND_WORKER'
          ? {
            PLACEMENT: placementType,
            HANDLER: handler,
            OPTIONS: config.options
          }
          : {
            PLACEMENT: placementType,
            HANDLER: handler,
            LANG_ALL: {
              ru: {
                TITLE: config.title,
                DESCRIPTION: config.description,
                GROUP_NAME: 'Управление обеденным перерывом'
              },
              en: {
                TITLE: config.title,
                DESCRIPTION: config.description,
                GROUP_NAME: 'Lunch break management'
              },
              be: {
                TITLE: config.title,
                DESCRIPTION: config.description,
                GROUP_NAME: 'Кіраванне абедзенным перапынкам'
              },
              kk: {
                TITLE: config.title,
                DESCRIPTION: config.description,
                GROUP_NAME: 'Түскі үзілісті басқару'
              }
            },
            OPTIONS: config.options
          }

      await bitrixAPI.call('placement.bind', placementConfig)
      return true
    } catch (error) {
      throw error
    }
  },

  reinstall: async (placementType, handler) => {
    try {
      const placements = await bitrixAPI.getPlacements()

      const existingPlacement = placements.find(p =>
          p.PLACEMENT === placementType && p.HANDLER === handler
      )

      if (existingPlacement) {
        await placementManager.unbind(placementType, handler)
      }

      await placementManager.bind(placementType, handler)

      return true
    } catch (error) {
      throw error
    }
  }
}

// Преобразование Time в строку
const timeToString = (time: Time | string | null): string | null => {
  if (!time) return null
  if (typeof time === 'string') return time
  return `${time.hour.toString().padStart(2, '0')}:${time.minute.toString().padStart(2, '0')}`
}

// Сохранение настроек приложения через app.option.set
const saveSettings = async () => {
  settingsStatus.value = 'loading'
  try {
    const settingsToSave = {
      // Настройки начала обеда
      lunch_start_enabled: selectedFeatures.value.lunchStart ? 'Y' : 'N',
      lunch_start_method: configSettings.value.lunchStart.method,
      // Настройки завершения обеда
      lunch_end_enabled: selectedFeatures.value.lunchEnd ? 'Y' : 'N',
      lunch_end_method: configSettings.value.lunchEnd.method,
      // Настройки активности в выходные
      lunch_weekend_activity_enabled: selectedFeatures.value.weekendActivity ? 'Y' : 'N',
      // Настройки времени обеда по умолчанию
      lunch_default_start_time: configSettings.value.defaultLunchTime.startTime,
      lunch_default_end_time: configSettings.value.defaultLunchTime.endTime,
      // Флаг завершения установки
      installation_completed: 'Y'
    }

    for (const [key, value] of Object.entries(settingsToSave)) {
      await bitrixAPI.setAppOption(key, value)
    }

    settingsStatus.value = 'success'
    installedCount.value++
    return true
  } catch (error) {
    settingsStatus.value = 'error'
    return false
  }
}

// Регистрация фоновой встройки
const registerPageBackgroundWorker = async () => {
  placementStatus.value.pageBackgroundWorker = 'loading'
  try {
    await placementManager.reinstall('PAGE_BACKGROUND_WORKER', HANDLERS.pageBackgroundWorker)
    placementStatus.value.pageBackgroundWorker = 'success'
    installedCount.value++
  } catch (error) {
    placementStatus.value.pageBackgroundWorker = 'error'
  }
}

// Регистрация встройки REST_APP_URI
const registerRestAppUri = async () => {
  placementStatus.value.restAppUri = 'loading'
  try {
    await placementManager.reinstall('REST_APP_URI', HANDLERS.restAppUri)
    placementStatus.value.restAppUri = 'success'
    installedCount.value++
  } catch (error) {
    placementStatus.value.restAppUri = 'error'
  }
}

// Основная функция установки
const startInstallation = async () => {
  if (currentStep.value === 2) {
    currentStep.value = 3
  }

  isInstalling.value = true
  installedCount.value = 0

  placementStatus.value = {
    pageBackgroundWorker: null,
    restAppUri: null
  }
  settingsStatus.value = null

  try {
    await registerPageBackgroundWorker()
    await registerRestAppUri()
    await saveSettings()
    installationComplete.value = true
  } catch (error) {
    // Ошибка при установке
  } finally {
    isInstalling.value = false
  }
}

const startAutoFinishTimer = () => {
  if (currentStep.value === 4) {
    isTimerActive.value = true
    secondsLeft.value = 5

    autoFinishTimer.value = setInterval(() => {
      secondsLeft.value -= 1

      if (secondsLeft.value <= 0) {
        clearInterval(autoFinishTimer.value)
        isTimerActive.value = false
        finishInstallation()
      }
    }, 1000)
  }
}

const cancelAutoFinish = () => {
  if (autoFinishTimer.value) {
    clearInterval(autoFinishTimer.value)
    autoFinishTimer.value = null
    isTimerActive.value = false
  }
}

const hasSelectedFeatures = computed(() => {
  return Object.values(selectedFeatures.value).some(value => value === true)
})

const installationsToProcess = computed(() => {
  return 3 // Две встройки + настройки
})

const installationProgress = computed(() => {
  if (installationsToProcess.value === 0) return 0
  return Math.round((installedCount.value / installationsToProcess.value) * 100)
})

const nextStep = () => {
  if (currentStep.value === 2) {
    startInstallation()
  } else if (currentStep.value < totalSteps) {
    currentStep.value++
    if (currentStep.value === 4) {
      startAutoFinishTimer()
    }
  }
}

const prevStep = () => {
  if (currentStep.value > 1) {
    cancelAutoFinish()
    currentStep.value--
  }
}

const finishInstallation = async () => {
  cancelAutoFinish()
  try {
    await bitrixAPI.installFinish()
  } catch (error) {
    // Ошибка завершения установки
  }
}

onUnmounted(() => {
  if (autoFinishTimer.value) {
    clearInterval(autoFinishTimer.value)
  }
})
</script>

<template>
  <div>
    <!-- Шапка с логотипом и заголовком -->
    <div class="text-center py-8 md:py-12">
      <div class="inline-flex items-center justify-center w-16 h-16 md:w-20 md:h-20 bg-blue-100 rounded-2xl mb-4 md:mb-6">
        <svg
            xmlns="http://www.w3.org/2000/svg"
            height="20px"
            viewBox="0 -960 960 960"
            width="20px"
            fill="#009DDB"
            class="w-8 h-8 md:w-10 md:h-10 text-blue-600"
        >
          <path d="M336-240h48v-241q26-4 43-24t17-47v-150.1q0-7.9-5-12.9t-13-5q-8 0-13 5t-5 13.15V-600h-30v-101.85q0-8.15-5-13.15t-13-5q-8 0-13 5t-5 13.15V-600h-30v-101.85q0-8.15-5-13.15t-13-5q-8 0-13 5t-5 12.9V-552q0 27 17 47t43 24v241Zm240 0h48v-246q26-11 43-41.78 17-30.77 17-72.1Q684-650 659.5-685 635-720 600-720t-59.5 35Q516-650 516-599.88q0 41.33 17 72.1Q550-497 576-486v246ZM168-96q-29.7 0-50.85-21.15Q96-138.3 96-168v-624q0-29.7 21.15-50.85Q138.3-864 168-864h624q29.7 0 50.85 21.15Q864-821.7 864-792v624q0 29.7-21.15 50.85Q821.7-96 792-96H168Zm0-72h624v-624H168v624Zm0 0v-624 624Z"/>
        </svg>
      </div>
      <h1 class="text-2xl md:text-3xl font-bold text-gray-900 mb-2 md:mb-3 px-2">
        Установка приложения "На обед!"
      </h1>
    </div>

    <!-- Прогресс-бар -->
    <div class="container mx-auto px-4 py-4">
      <div class="mx-auto mb-6 md:mb-8">
        <div class="flex items-center justify-between mb-2">
          <span class="text-xs md:text-sm font-medium text-blue-600">Прогресс: {{ progress }}%</span>
          <span class="text-xs md:text-sm text-gray-500">{{ currentStep }}/{{ totalSteps }}</span>
        </div>
        <div class="h-1.5 md:h-2 bg-gray-200 rounded-full overflow-hidden">
          <div class="h-full bg-blue-600 transition-all duration-500 ease-out" :style="{ width: progress + '%' }"></div>
        </div>
      </div>
    </div>

    <!-- Шаг 1: Приветствие -->
    <B24PageSection v-if="currentStep === 1">
      <B24PageCard>
        <div class="flex flex-col md:flex-row md:items-start md:space-x-6">
          <div class="hidden md:flex md:flex-shrink-0 mb-4 md:mb-0">
            <div class="w-16 h-16 bg-blue-100 rounded-xl flex items-center justify-center">
              <ShieldIcon class="w-8 h-8 text-blue-600" />
            </div>
          </div>
          <div class="flex-1">
            <h2 class="text-xl md:text-2xl font-bold text-gray-900 mb-3 md:mb-4">
              Добро пожаловать в систему управления обеденными перерывами
            </h2>
            <p class="text-sm md:text-base text-gray-600 mb-4 md:mb-6">
              Настройте систему, которая поможет сотрудникам своевременно начинать и завершать обеденный перерыв.
            </p>

            <div class="bg-blue-50 rounded-xl p-4 md:p-6 mb-4 md:mb-6">
              <h3 class="text-base md:text-lg font-semibold text-blue-800 mb-3">
                Ключевые возможности системы
              </h3>
              <ul class="space-y-2 md:space-y-3">
                <li class="flex items-start">
                  <CheckIcon class="w-4 h-4 md:w-5 md:h-5 text-green-500 mr-2 md:mr-3 flex-shrink-0 mt-0.5" />
                  <span class="text-sm md:text-base text-gray-700">Помощь в начале и завершении обеденного перерыва</span>
                </li>
                <li class="flex items-start">
                  <CheckIcon class="w-4 h-4 md:w-5 md:h-5 text-green-500 mr-2 md:mr-3 flex-shrink-0 mt-0.5" />
                  <span class="text-sm md:text-base text-gray-700">Гибкая настройка способов уведомлений (модальное окно, автоматический, push, чат)</span>
                </li>
                <li class="flex items-start">
                  <CheckIcon class="w-4 h-4 md:w-5 md:h-5 text-green-500 mr-2 md:mr-3 flex-shrink-0 mt-0.5" />
                  <span class="text-sm md:text-base text-gray-700">Настройка времени обеда по умолчанию</span>
                </li>
              </ul>
            </div>

            <div class="flex justify-end">
              <B24Button @click="nextStep" label="Начать настройку" color="primary" :icon="ArrowRightLIcon" size="lg" icon-position="right" />
            </div>
          </div>
        </div>
      </B24PageCard>
    </B24PageSection>

    <!-- Шаг 2: Настройка функций -->
    <B24PageSection v-else-if="currentStep === 2">
      <B24PageCard>
        <div class="flex flex-col md:flex-row md:items-start md:space-x-6">
          <div class="hidden md:flex md:flex-shrink-0 mb-4 md:mb-0">
            <div class="w-16 h-16 bg-blue-100 rounded-xl flex items-center justify-center">
              <SettingsIcon class="w-8 h-8 text-blue-600" />
            </div>
          </div>
          <div class="flex-1">
            <h2 class="text-xl md:text-2xl font-bold text-gray-900 mb-3 md:mb-4">
              Настройка основных функций системы
            </h2>
            <p class="text-sm md:text-base text-gray-600 mb-4 md:mb-6">
              Выберите, какие функции управления обеденными перерывами вы хотите активировать и настроить.
            </p>

            <div class="space-y-6">
              <!-- Помощь в начале обеда -->
              <B24PageCard>
                <div class="flex items-center justify-between">
                  <div>
                    <h3 class="text-lg font-semibold text-gray-900">Помощь в начале обеда</h3>
                    <p class="text-sm text-gray-500">Автоматическая помощь сотрудникам в своевременном начале обеденного перерыва</p>
                  </div>
                  <B24Switch v-model="selectedFeatures.lunchStart" />
                </div>
                <div v-if="selectedFeatures.lunchStart" class="mt-4 pt-4 border-t">
                  <p class="text-sm font-medium text-gray-700 mb-3">Способ начала обеда:</p>
                  <B24RadioGroup
                      v-model="configSettings.lunchStart.method"
                      :items="[
                        { label: 'Автоматическое начало обеда', value: 'auto', description: 'Обеденный перерыв начинается автоматически' },
                        { label: 'Модальное окно с предложением', value: 'modal', description: 'Показывать окно с предложением начать обед' },
                        { label: 'Сообщение в чате', value: 'chat', description: 'Отправлять уведомление в чат Б24' },
                        { label: 'Push-уведомление', value: 'push', description: 'Отправлять push-уведомление' }
                      ]"
                      orientation="horizontal"
                      variant="card"
                      size="sm"
                      default-value="modal"
                      indicator="end"
                      class="overflow-scroll md:overflow-auto"
                  />
                </div>
              </B24PageCard>

              <!-- Помощь в завершении обеда -->
              <B24PageCard>
                <div class="flex items-center justify-between">
                  <div>
                    <h3 class="text-lg font-semibold text-gray-900">Помощь в завершении обеда</h3>
                    <p class="text-sm text-gray-500">Автоматическая помощь сотрудникам в своевременном завершении обеденного перерыва</p>
                  </div>
                  <B24Switch v-model="selectedFeatures.lunchEnd" />
                </div>
                <div v-if="selectedFeatures.lunchEnd" class="mt-4 pt-4 border-t">
                  <p class="text-sm font-medium text-gray-700 mb-3">Способ завершения обеда:</p>
                  <B24RadioGroup
                      v-model="configSettings.lunchEnd.method"
                      :items="[
                        { label: 'Автоматическое завершение обеда', value: 'auto', description: 'Обеденный перерыв завершается автоматически' },
                        { label: 'Модальное окно с предложением', value: 'modal', description: 'Показывать окно с предложением завершить обед' },
                        { label: 'Сообщение в чате', value: 'chat', description: 'Отправлять уведомление в чат Б24' },
                        { label: 'Push-уведомление', value: 'push', description: 'Отправлять push-уведомление' }
                      ]"
                      orientation="horizontal"
                      variant="card"
                      size="sm"
                      default-value="modal"
                      indicator="end"
                      class="overflow-scroll md:overflow-auto"
                  />
                </div>
              </B24PageCard>

              <!-- Настройка времени обеда по умолчанию -->
              <B24PageCard>
                <div class="flex items-center justify-between">
                  <div>
                    <h3 class="text-lg font-semibold text-gray-900">Время обеда по умолчанию</h3>
                    <p class="text-sm text-gray-500">Установите стандартное время для обеденного перерыва</p>
                  </div>
                </div>
                <div class="mt-4 pt-4 border-t">
                  <div class="grid grid-cols-1 sm:grid-cols-2 gap-4">
                    <div>
                      <label class="block text-sm font-medium text-gray-700 mb-1">Время начала обеда</label>
                      <B24InputTime
                          v-model="configSettings.defaultLunchTime.startTime"
                          :hour-cycle="24"
                          size="md"
                          color="air-primary"
                          placeholder="Выберите время"
                      />
                    </div>
                    <div>
                      <label class="block text-sm font-medium text-gray-700 mb-1">Время завершения обеда</label>
                      <B24InputTime
                          v-model="configSettings.defaultLunchTime.endTime"
                          :hour-cycle="24"
                          size="md"
                          color="air-primary"
                          placeholder="Выберите время"
                      />
                    </div>
                  </div>
                </div>
              </B24PageCard>

              <!-- Активность в выходные -->
              <B24PageCard>
                <div class="flex items-center justify-between">
                  <div>
                    <h3 class="text-lg font-semibold text-gray-900">Активность в выходные</h3>
                    <p class="text-sm text-gray-500">Разрешить уведомления и активность в выходные дни</p>
                  </div>
                  <B24Switch v-model="selectedFeatures.weekendActivity" />
                </div>
              </B24PageCard>
            </div>

            <div class="flex justify-between gap-3 pt-6 mt-6 border-t">
              <B24Button @click="prevStep" label="Назад" variant="outline" size="lg" :icon="ArrowLeftLIcon" />
              <B24Button @click="nextStep" label="Продолжить" color="primary" size="lg" :disabled="!hasSelectedFeatures" :icon="ArrowRightLIcon" icon-position="right" />
            </div>
          </div>
        </div>
      </B24PageCard>
    </B24PageSection>

    <!-- Шаг 3: Установка -->
    <B24PageSection v-else-if="currentStep === 3">
      <B24PageCard>
        <div class="flex flex-col md:flex-row md:items-start md:space-x-6">
          <div class="hidden md:flex md:flex-shrink-0 mb-4 md:mb-0">
            <div class="w-16 h-16 bg-blue-100 rounded-xl flex items-center justify-center">
              <RefreshIcon class="w-8 h-8 text-blue-600" />
            </div>
          </div>
          <div class="flex-1">
            <h2 class="text-xl md:text-2xl font-bold text-gray-900 mb-3 md:mb-4">Установка системы</h2>
            <p class="text-sm md:text-base text-gray-600 mb-4 md:mb-6">Выполняется установка и настройка выбранных компонентов системы управления обеденными перерывами.</p>

            <div class="mb-6 md:mb-8">
              <div class="flex items-center justify-between mb-2">
                <span class="text-xs md:text-sm font-medium text-blue-600">Прогресс: {{ installationProgress }}%</span>
                <span class="text-xs md:text-sm text-gray-500">{{ installedCount }}/{{ installationsToProcess }}</span>
              </div>
              <div class="h-1.5 md:h-2 bg-gray-200 rounded-full overflow-hidden">
                <div class="h-full bg-blue-600 transition-all duration-300 ease-out" :style="{ width: installationProgress + '%' }"></div>
              </div>
            </div>

            <div class="space-y-3 md:space-y-4 mb-6 md:mb-8">
              <!-- Фоновая встройка -->
              <div class="flex items-center">
                <div class="w-6 h-6 md:w-8 md:h-8 flex-shrink-0">
                  <div v-if="placementStatus.pageBackgroundWorker === 'loading'">
                    <LoadingIcon class="animate-spin h-4 w-4 md:h-5 md:w-5 text-blue-600" />
                  </div>
                  <div v-else-if="placementStatus.pageBackgroundWorker === 'success'">
                    <SuccessIcon class="w-4 h-4 md:w-5 md:h-5 text-green-500" />
                  </div>
                  <div v-else-if="placementStatus.pageBackgroundWorker === 'error'">
                    <ErrorIcon class="w-4 h-4 md:w-5 md:h-5 text-red-500" />
                  </div>
                  <div v-else>
                    <div class="w-4 h-4 md:w-5 md:h-5 bg-gray-300 rounded-full"></div>
                  </div>
                </div>
                <div class="ml-2 md:ml-3">
                  <p class="text-xs md:text-sm font-medium text-gray-900">Фоновая встройка</p>
                  <p class="text-xs text-gray-500">Автоматически определяет время начала и завершения обеденного перерыва и оповещает об этом сотрудника</p>
                </div>
              </div>

              <!-- Встройка для управления обедом из уведомлений -->
              <div class="flex items-center">
                <div class="w-6 h-6 md:w-8 md:h-8 flex-shrink-0">
                  <div v-if="placementStatus.restAppUri === 'loading'">
                    <LoadingIcon class="animate-spin h-4 w-4 md:h-5 md:w-5 text-blue-600" />
                  </div>
                  <div v-else-if="placementStatus.restAppUri === 'success'">
                    <SuccessIcon class="w-4 h-4 md:w-5 md:h-5 text-green-500" />
                  </div>
                  <div v-else-if="placementStatus.restAppUri === 'error'">
                    <ErrorIcon class="w-4 h-4 md:w-5 md:h-5 text-red-500" />
                  </div>
                  <div v-else>
                    <div class="w-4 h-4 md:w-5 md:h-5 bg-gray-300 rounded-full"></div>
                  </div>
                </div>
                <div class="ml-2 md:ml-3">
                  <p class="text-xs md:text-sm font-medium text-gray-900">Встройка для управления обедом из уведомлений</p>
                  <p class="text-xs text-gray-500">Позволяет сотруднику управлять обеденным перерывом через сообщения в чатах или push-уведомлениях</p>
                </div>
              </div>

              <!-- Настройка параметров системы -->
              <div class="flex items-center">
                <div class="w-6 h-6 md:w-8 md:h-8 flex-shrink-0">
                  <div v-if="settingsStatus === 'loading'">
                    <LoadingIcon class="animate-spin h-4 w-4 md:h-5 md:w-5 text-blue-600" />
                  </div>
                  <div v-else-if="settingsStatus === 'success'">
                    <SuccessIcon class="w-4 h-4 md:w-5 md:h-5 text-green-500" />
                  </div>
                  <div v-else-if="settingsStatus === 'error'">
                    <ErrorIcon class="w-4 h-4 md:w-5 md:h-5 text-red-500" />
                  </div>
                  <div v-else>
                    <div class="w-4 h-4 md:w-5 md:h-5 bg-gray-300 rounded-full"></div>
                  </div>
                </div>
                <div class="ml-2 md:ml-3">
                  <p class="text-xs md:text-sm font-medium text-gray-900">Настройка параметров системы</p>
                  <p class="text-xs text-gray-500">Сохранение настроек времени обеда и способов уведомлений</p>
                </div>
              </div>
            </div>

            <div class="flex justify-end gap-3 pt-6 border-t">
              <B24Button v-if="!installationComplete" @click="prevStep" label="Назад" size="lg" variant="outline" :disabled="isInstalling" :icon="ArrowLeftLIcon" />
              <B24Button v-if="installationComplete && (placementStatus.pageBackgroundWorker === 'success' || placementStatus.pageBackgroundWorker === 'error') && (placementStatus.restAppUri === 'success' || placementStatus.restAppUri === 'error')" @click="nextStep" label="Продолжить" size="lg" color="primary" :icon="ArrowRightLIcon" icon-position="right" />
            </div>
          </div>
        </div>
      </B24PageCard>
    </B24PageSection>

    <!-- Шаг 4: Завершение -->
    <B24PageSection v-else-if="currentStep === 4">
      <B24PageCard>
        <div class="flex flex-col md:flex-row md:items-start md:space-x-6">
          <div class="hidden md:flex md:flex-shrink-0 mb-4 md:mb-0">
            <div class="w-16 h-16 bg-green-100 rounded-xl flex items-center justify-center">
              <SuccessIcon class="w-8 h-8 text-green-600" />
            </div>
          </div>
          <div class="flex-1">
            <h2 class="text-xl md:text-2xl font-bold text-gray-900 mb-3 md:mb-4">Установка завершена!</h2>

            <div class="space-y-4 md:space-y-6">
              <p class="text-base md:text-lg text-gray-700">Система управления обеденными перерывами успешно установлена и настроена.</p>

              <B24PageCard variant="tinted">
                <h3 class="text-lg font-bold text-gray-900 mb-2">Что дальше?</h3>
                <p class="text-sm text-gray-600 mb-4">Система готова к использованию. Вы можете начать управлять обеденными перерывами сотрудников прямо сейчас.</p>

                <div class="space-y-2">
                  <div v-if="selectedFeatures.lunchStart" class="flex items-start">
                    <CheckIcon class="w-4 h-4 text-green-500 mr-2 flex-shrink-0 mt-0.5" />
                    <span class="text-sm text-gray-700">Помощь в начале обеда (способ: {{ configSettings.lunchStart.method }})</span>
                  </div>
                  <div v-if="selectedFeatures.lunchEnd" class="flex items-start">
                    <CheckIcon class="w-4 h-4 text-green-500 mr-2 flex-shrink-0 mt-0.5" />
                    <span class="text-sm text-gray-700">Помощь в завершении обеда (способ: {{ configSettings.lunchEnd.method }})</span>
                  </div>
                  <div class="flex items-start">
                    <CheckIcon class="w-4 h-4 text-green-500 mr-2 flex-shrink-0 mt-0.5" />
                    <span class="text-sm text-gray-700">Время обеда по умолчанию: {{ configSettings.defaultLunchTime.startTime }} - {{ configSettings.defaultLunchTime.endTime }}</span>
                  </div>
                  <div v-if="selectedFeatures.weekendActivity" class="flex items-start">
                    <CheckIcon class="w-4 h-4 text-green-500 mr-2 flex-shrink-0 mt-0.5" />
                    <span class="text-sm text-gray-700">Активность в выходные дни (включена)</span>
                  </div>
                  <div v-else class="flex items-start">
                    <CheckIcon class="w-4 h-4 text-gray-400 mr-2 flex-shrink-0 mt-0.5" />
                    <span class="text-sm text-gray-500">Активность в выходные дни (выключена)</span>
                  </div>
                </div>

                <div class="mt-4">
                  <B24Link href="mailto:technogalera@yandex.ru?subject=Поддержка приложения На обед!" target="_blank" is-action>
                    Техническая поддержка
                  </B24Link>
                </div>
              </B24PageCard>

              <B24PageCard variant="tinted-alt">
                <h3 class="text-base font-semibold text-gray-900 mb-2">Следующие шаги</h3>
                <ul class="space-y-2">
                  <li class="flex items-start text-gray-700">
                    <ArrowRightLIcon class="w-4 h-4 text-blue-500 mr-2 flex-shrink-0 mt-0.5" />
                    <span class="text-sm">Перейдите в приложение для просмотра статистики обеденных перерывов</span>
                  </li>
                  <li class="flex items-start text-gray-700">
                    <ArrowRightLIcon class="w-4 h-4 text-blue-500 mr-2 flex-shrink-0 mt-0.5" />
                    <span class="text-sm">Настройте дополнительные параметры в разделе "Настройки обеда"</span>
                  </li>
                  <li class="flex items-start text-gray-700">
                    <ArrowRightLIcon class="w-4 h-4 text-blue-500 mr-2 flex-shrink-0 mt-0.5" />
                    <span class="text-sm">Ознакомьте сотрудников с новыми возможностями системы</span>
                  </li>
                </ul>
              </B24PageCard>

              <div class="flex justify-end gap-3 pt-6 border-t">
                <B24Button @click="finishInstallation" size="lg" :label="`Завершить установку${isTimerActive ? ` (${secondsLeft}с)` : ''}`" color="primary" />
              </div>
            </div>
          </div>
        </div>
      </B24PageCard>
    </B24PageSection>
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