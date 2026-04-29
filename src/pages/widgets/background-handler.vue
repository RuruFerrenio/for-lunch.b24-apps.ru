<script setup lang="ts">
import { ref, onMounted, onUnmounted } from 'vue'
import { Time } from '@internationalized/date'

// Типы данных
interface LunchSettings {
  enabled: boolean
  method: 'auto' | 'modal' | 'chat' | 'push'
}

interface WeekendActivitySettings {
  enabled: boolean
}

interface DefaultLunchTime {
  startTime: string
  endTime: string
}

interface UserLunchTime {
  startTime: string | null
  endTime: string | null
}

// Состояние компонента
const isBitrixLoaded = ref(false)
const isProcessing = ref(false)
const currentUserId = ref<number | null>(null)
const applicationOpened = ref(false)
const isTimemanAvailable = ref<boolean | null>(null) // null - не проверено, false - недоступен, true - доступен
const isLoadingUserOptions = ref(false)

// Настройки обеда
const lunchStart = ref<LunchSettings>({
  enabled: false,
  method: 'modal'
})

const lunchEnd = ref<LunchSettings>({
  enabled: false,
  method: 'modal'
})

// Настройка активности в выходные
const weekendActivity = ref<WeekendActivitySettings>({
  enabled: false
})

// Время обеда по умолчанию (из опций приложения)
const defaultLunchTime = ref<DefaultLunchTime>({
  startTime: '12:00',
  endTime: '13:00'
})

// Индивидуальное время обеда пользователя (из user.options)
const userLunchTime = ref<UserLunchTime>({
  startTime: null,
  endTime: null
})

// Флаги отправленных уведомлений
let startNotificationSent = false
let endNotificationSent = false
let lastCheckedDate: string | null = null

// Таймер для периодической проверки
let periodicCheckInterval: ReturnType<typeof setInterval> | null = null
let isPageVisible = true

// Конфигурация модальных окон
const MODAL_CONFIG = {
  WIDTH: 500,
  DYNAMIC_PAGE_PATH: '/marketplace/view/app.69ef1f52649017.42730927/',
  CHECK_INTERVAL_SECONDS: 30 // Интервал проверки в секундах
}

// ==========================================================================
// ФУНКЦИИ ДЛЯ РАБОТЫ С USER.OPTION (индивидуальные настройки пользователя)
// ==========================================================================

/**
 * Сохраняет индивидуальное время обеда пользователя через user.option.set
 */
async function saveUserLunchTimeToUserOptions(startTime: string | null, endTime: string | null): Promise<boolean> {
  if (!isBitrixLoaded.value || typeof BX24 === 'undefined') {
    console.error('BX24 не загружен')
    return false
  }

  return new Promise((resolve) => {
    const options: Record<string, string> = {
      lunch_start_time: startTime || '',
      lunch_end_time: endTime || ''
    }

    BX24.callMethod('user.option.set', {
      options: options
    }, (result: any) => {
      if (result.error()) {
        console.error('Ошибка при сохранении настроек обеда:', result.error())
        resolve(false)
      } else {
        // Обновляем локальное состояние
        userLunchTime.value.startTime = startTime
        userLunchTime.value.endTime = endTime
        resolve(true)
      }
    })
  })
}

/**
 * Загружает индивидуальное время обеда пользователя через user.option.get
 */
async function loadUserLunchTimeFromUserOptions(): Promise<void> {
  if (!isBitrixLoaded.value || typeof BX24 === 'undefined') {
    console.error('BX24 не загружен')
    return
  }

  isLoadingUserOptions.value = true

  return new Promise((resolve) => {
    BX24.callMethod('user.option.get', {
      names: ['lunch_start_time', 'lunch_end_time']
    }, (result: any) => {
      isLoadingUserOptions.value = false

      if (result.error()) {
        console.error('Ошибка при загрузке настроек обеда:', result.error())
        resolve()
        return
      }

      const data = result.data()

      if (data && data.lunch_start_time && data.lunch_start_time !== '') {
        userLunchTime.value.startTime = data.lunch_start_time
      } else {
        userLunchTime.value.startTime = null
      }

      if (data && data.lunch_end_time && data.lunch_end_time !== '') {
        userLunchTime.value.endTime = data.lunch_end_time
      } else {
        userLunchTime.value.endTime = null
      }

      resolve()
    })
  })
}

// Очистка временных флагов
function clearTempFlags(): void {
  if (typeof document === 'undefined') return
  document.cookie = 'open_app_mode=; expires=Thu, 01 Jan 1970 00:00:00 UTC; path=/; SameSite=None; Secure'
  document.cookie = 'modal_type=; expires=Thu, 01 Jan 1970 00:00:00 UTC; path=/; SameSite=None; Secure'
}

// ==========================================================================
// ФУНКЦИИ ДЛЯ РАБОТЫ С LOCALSTORAGE (флаги уведомлений)
// ==========================================================================

function setStoredFlag(key: string, value: string, hours: number): void {
  const data = {
    value: value,
    expires: Date.now() + (hours * 60 * 60 * 1000)
  }
  localStorage.setItem(key, JSON.stringify(data))
}

function getStoredFlag(key: string): string | null {
  const item = localStorage.getItem(key)
  if (!item) return null

  try {
    const data = JSON.parse(item)
    if (Date.now() > data.expires) {
      localStorage.removeItem(key)
      return null
    }
    return data.value
  } catch (e) {
    localStorage.removeItem(key)
    return null
  }
}

function deleteStoredFlag(key: string): void {
  localStorage.removeItem(key)
}

// ==========================================================================
// ФУНКЦИЯ ПРОВЕРКИ ДОСТУПНОСТИ МЕТОДА timeman.status
// ==========================================================================

async function checkTimemanAvailability(): Promise<boolean> {
  if (typeof BX24 === 'undefined') {
    console.warn('⚠️ BX24 не загружен')
    return false
  }

  return new Promise((resolve) => {
    BX24.callMethod('method.get', {
      name: 'timeman.status'
    }, (result: any) => {
      if (result.error()) {
        console.warn('⚠️ Метод method.get вернул ошибку:', result.error())
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
// ФУНКЦИИ ДЛЯ РАБОТЫ С НАСТРОЙКАМИ (опции приложения)
// ==========================================================================

function normalizeBoolean(value: unknown): boolean {
  return value === 'Y' || value === true || value === 1
}

function normalizeMethod(value: unknown): 'auto' | 'modal' | 'chat' | 'push' | null {
  if (value === 'auto' || value === 'modal' || value === 'chat' || value === 'push') {
    return value
  }
  return null
}

async function loadSettings(): Promise<void> {
  if (!isBitrixLoaded.value || typeof BX24 === 'undefined') return

  try {
    const [
      lunchStartEnabled,
      lunchStartMethod,
      lunchEndEnabled,
      lunchEndMethod,
      weekendActivityEnabled,
      defaultLunchStartTime,
      defaultLunchEndTime
    ] = await Promise.all([
      BX24.appOption.get('lunch_start_enabled'),
      BX24.appOption.get('lunch_start_method'),
      BX24.appOption.get('lunch_end_enabled'),
      BX24.appOption.get('lunch_end_method'),
      BX24.appOption.get('lunch_weekend_activity_enabled'),
      BX24.appOption.get('lunch_default_start_time'),
      BX24.appOption.get('lunch_default_end_time')
    ])

    lunchStart.value.enabled = normalizeBoolean(lunchStartEnabled)
    const startMethod = normalizeMethod(lunchStartMethod)
    if (startMethod) {
      lunchStart.value.method = startMethod
    }

    lunchEnd.value.enabled = normalizeBoolean(lunchEndEnabled)
    const endMethod = normalizeMethod(lunchEndMethod)
    if (endMethod) {
      lunchEnd.value.method = endMethod
    }

    weekendActivity.value.enabled = normalizeBoolean(weekendActivityEnabled)

    if (defaultLunchStartTime && typeof defaultLunchStartTime === 'string') {
      defaultLunchTime.value.startTime = defaultLunchStartTime
    }
    if (defaultLunchEndTime && typeof defaultLunchEndTime === 'string') {
      defaultLunchTime.value.endTime = defaultLunchEndTime
    }

  } catch (error) {
    console.error('Ошибка загрузки настроек:', error)
  }
}

// ==========================================================================
// ФУНКЦИИ ДЛЯ РАБОТЫ С ВРЕМЕНЕМ ОБЕДА
// ==========================================================================

// Получение текущего времени обеда (приоритет: индивидуальные настройки > настройки по умолчанию)
function getCurrentLunchStartTime(): string {
  if (userLunchTime.value.startTime) {
    return userLunchTime.value.startTime
  }
  return defaultLunchTime.value.startTime
}

function getCurrentLunchEndTime(): string {
  if (userLunchTime.value.endTime) {
    return userLunchTime.value.endTime
  }
  return defaultLunchTime.value.endTime
}

// Проверка, нужно ли сегодня выполнять действия (с учетом выходных)
function shouldAllowActivity(callback: (allow: boolean) => void): void {
  checkIsWeekend(function(isWeekend: boolean) {
    if (isWeekend && !weekendActivity.value.enabled) {
      callback(false)
    } else {
      callback(true)
    }
  })
}

// Проверка, является ли текущий день выходным
function checkIsWeekend(callback: (isWeekend: boolean) => void): void {
  if (typeof BX24 === 'undefined') {
    callback(false)
    return
  }

  getCurrentUserId(function(userId: number) {
    if (!userId) {
      console.error('Не удалось получить ID пользователя')
      callback(false)
      return
    }

    BX24.callMethod(
        'timeman.settings',
        { USER_ID: userId },
        function(result: any) {
          if (result.error()) {
            console.error('Ошибка получения настроек для проверки выходных:', result.error())
            callback(false)
            return
          }

          const settings = result.data()
          const now = new Date()
          const currentDayOfWeek = now.getDay()

          let isWeekend = false

          if (settings.UF_TM_FREE === true) {
            callback(false)
            return
          }

          if (currentDayOfWeek === 0 || currentDayOfWeek === 6) {
            isWeekend = true
          }

          if (settings.SCHEDULE_ID) {
            isWeekend = (currentDayOfWeek === 0 || currentDayOfWeek === 6)
          }

          callback(isWeekend)
        }
    )
  })
}

// Получение ID текущего пользователя
function getCurrentUserId(callback: (userId: number) => void): void {
  if (typeof BX24 === 'undefined') return

  BX24.callMethod(
      'user.current',
      {},
      function(result: any) {
        if (result.error()) {
          console.error('Ошибка получения текущего пользователя:', result.error())
          return
        }

        const user = result.data()
        callback(user.ID)
      }
  )
}

// ==========================================================================
// ФУНКЦИИ ДЛЯ РАБОТЫ С API БИТРИКС24
// ==========================================================================

// Постановка рабочего дня на паузу (начало обеда)
function pauseWorkday(): void {
  if (!isBitrixLoaded.value || typeof BX24 === 'undefined') return
  if (!isTimemanAvailable.value) {
    console.warn('Метод timeman.status недоступен, пропускаем')
    return
  }

  shouldAllowActivity(function(allow: boolean) {
    if (!allow) return

    isProcessing.value = true
    BX24.callMethod(
        'timeman.pause',
        {},
        function(result: any) {
          isProcessing.value = false

          if (result.error()) {
            console.error('Ошибка при постановке на паузу:', result.error())
          } else {
            // После успешной постановки на паузу очищаем флаг уведомления
            deleteStoredFlag('lunch_start_notification_sent')
            startNotificationSent = false
          }
        }
    )
  })
}

// Возобновление рабочего дня (завершение обеда)
function resumeWorkday(): void {
  if (!isBitrixLoaded.value || typeof BX24 === 'undefined') return
  if (!isTimemanAvailable.value) {
    console.warn('Метод timeman.status недоступен, пропускаем')
    return
  }

  shouldAllowActivity(function(allow: boolean) {
    if (!allow) return

    isProcessing.value = true
    BX24.callMethod(
        'timeman.resume',
        {},
        function(result: any) {
          isProcessing.value = false

          if (result.error()) {
            console.error('Ошибка при возобновлении:', result.error())
          } else {
            // После успешного возобновления очищаем флаг уведомления
            deleteStoredFlag('lunch_end_notification_sent')
            endNotificationSent = false
          }
        }
    )
  })
}

// ==========================================================================
// ФУНКЦИИ ДЛЯ ОТПРАВКИ УВЕДОМЛЕНИЙ
// ==========================================================================

// Отправка push-уведомления
function sendPushNotification(userId: number, mode: 'start' | 'end'): void {
  if (typeof BX24 === 'undefined') return

  shouldAllowActivity(function(allow: boolean) {
    if (!allow) return

    const notificationKey = mode === 'start' ? 'lunch_start_notification_sent' : 'lunch_end_notification_sent'
    const notificationSent = getStoredFlag(notificationKey)

    console.log('notificationSent')
    console.log(notificationSent)

    if (notificationSent === 'true') {
      return
    }

    const modalUrl = `${MODAL_CONFIG.DYNAMIC_PAGE_PATH}`
    const title = mode === 'start' ? '[B]Время обеда![/B]' : '[B]Конец обеда![/B]'
    const message = mode === 'start'
        ? 'Пора сделать перерыв на обед!'
        : 'Обеденный перерыв закончился, пора возвращаться к работе!'

    const buttonText = mode === 'start' ? 'На обед!' : 'За работу!'
    const colorToken = mode === 'start' ? 'primary' : 'success'

    setStoredFlag(notificationKey, 'true', 24)

    BX24.callMethod(
        'im.notify.system.add',
        {
          USER_ID: userId,
          MESSAGE: title,
          MESSAGE_OUT: message,
          TAG: `lunch_${mode}_${Date.now()}`,
          SUB_TAG: `lunch_${mode}`,
          ATTACH: {
            ID: 1,
            COLOR_TOKEN: colorToken,
            BLOCKS: [
              {
                MESSAGE: ``
              },
              {
                LINK: {
                  NAME: buttonText,
                  LINK: modalUrl
                }
              }
            ]
          }
        },
        function(result: any) {
          if (result.error()) {
            console.error(`Ошибка отправки push-уведомления для ${mode === 'start' ? 'начала' : 'завершения'} обеда:`, result.error())
            deleteStoredFlag(notificationKey)
          }
        }
    )
  })
}

// Отправка сообщения в чат
function sendChatNotification(userId: number, mode: 'start' | 'end'): void {
  if (typeof BX24 === 'undefined') return

  shouldAllowActivity(function(allow: boolean) {
    if (!allow) return

    const notificationKey = mode === 'start' ? 'lunch_start_notification_sent' : 'lunch_end_notification_sent'
    const notificationSent = getStoredFlag(notificationKey)

    if (notificationSent === 'true') {
      return
    }

    const modalUrl = `${MODAL_CONFIG.DYNAMIC_PAGE_PATH}`
    const messageText = mode === 'start'
        ? '🍽️ Время сделать перерыв на обед!\n'
        : '💪 Обеденный перерыв закончился, пора возвращаться к работе!\n'

    const buttonText = mode === 'start' ? 'На обед!' : 'За работу!'

    const buttonStyles = mode === 'start'
        ? {
          BG_COLOR_TOKEN: 'primary',
          TEXT_COLOR: '#FFFFFF'
        }
        : {
          BG_COLOR_TOKEN: 'success',
          TEXT_COLOR: '#FFFFFF'
        }

    setStoredFlag(notificationKey, 'true', 24)

    BX24.callMethod(
        'im.message.add',
        {
          DIALOG_ID: userId.toString(),
          MESSAGE: messageText,
          SYSTEM: 'Y',
          KEYBOARD: {
            BUTTONS: [
              {
                TEXT: buttonText,
                LINK: modalUrl,
                BG_COLOR_TOKEN: buttonStyles.BG_COLOR_TOKEN,
                TEXT_COLOR: buttonStyles.TEXT_COLOR,
              }
            ]
          }
        },
        function(result: any) {
          if (result.error()) {
            console.error(`Ошибка отправки сообщения в чат для ${mode === 'start' ? 'начала' : 'завершения'} обеда:`, result.error())
            deleteStoredFlag(notificationKey)
          }
        }
    )
  })
}

// ==========================================================================
// ФУНКЦИИ ДЛЯ ОТКРЫТИЯ МОДАЛЬНОГО ОКНА
// ==========================================================================

function openLunchModal(mode: 'start' | 'end'): void {
  if (!isBitrixLoaded.value || typeof BX24 === 'undefined') return
  if (applicationOpened.value) return
  if (!isTimemanAvailable.value) {
    console.warn('Метод timeman.status недоступен, пропускаем')
    return
  }

  shouldAllowActivity(function(allow: boolean) {
    if (!allow) return

    applicationOpened.value = true

    const modalTitle = mode === 'start' ? 'Обеденный перерыв' : 'Возвращение с обеда'
    const bgColor = mode === 'start' ? 'primary' : 'success'
    const labelText = mode === 'start' ? 'Обед' : 'За работу'

    // Используем document.cookie для временных флагов (они не хранятся долго)
    if (typeof document !== 'undefined') {
      document.cookie = `open_app_mode=modal; path=/; SameSite=None; Secure`
      document.cookie = `modal_type=${mode}; path=/; SameSite=None; Secure`
    }

    const modalSettings = {
      opened: true,
      title: modalTitle,
      label: {
        bgColor: bgColor,
        text: labelText,
        color: '#ffffff',
      },
      width: MODAL_CONFIG.WIDTH,
    }

    BX24.openApplication({}, function() {
      onModalClosed(mode)
    }, modalSettings)
  })
}

function onModalClosed(mode: 'start' | 'end'): void {
  applicationOpened.value = false
  clearTempFlags()
}

// ==========================================================================
// ОСНОВНАЯ ЛОГИКА ПРОВЕРКИ ВРЕМЕНИ ОБЕДА
// ==========================================================================

// Преобразование времени в минуты для сравнения
function timeToMinutes(timeStr: string): number {
  const [hours, minutes] = timeStr.split(':').map(Number)
  return hours * 60 + minutes
}

// Сброс флагов уведомлений при смене дня
function resetDailyFlags(): void {
  const today = new Date().toDateString()
  if (lastCheckedDate !== today) {
    startNotificationSent = false
    endNotificationSent = false
    lastCheckedDate = today
  }
}

// Проверка статуса рабочего дня и выполнение действий
function checkWorkdayStatus(): void {
  if (isTimemanAvailable.value === false) {
    return
  }

  if (!isBitrixLoaded.value || typeof BX24 === 'undefined') return

  if (!isPageVisible) {
    return
  }

  // Сброс флагов при смене дня
  resetDailyFlags()

  getCurrentUserId(function(userId: number) {
    currentUserId.value = userId

    BX24.callMethod(
        'timeman.status',
        { USER_ID: userId },
        function(result: any) {
          if (result.error()) {
            console.error('Ошибка проверки статуса рабочего дня:', result.error())
            return
          }

          const workDayParams = result.data()
          const now = new Date()
          const currentMinutes = now.getHours() * 60 + now.getMinutes()

          const lunchStartTime = getCurrentLunchStartTime()
          const lunchEndTime = getCurrentLunchEndTime()

          const lunchStartMinutes = timeToMinutes(lunchStartTime)
          const lunchEndMinutes = timeToMinutes(lunchEndTime)

          // =============================================
          // ПРОВЕРКА ДЛЯ НАЧАЛА ОБЕДА (постановка на паузу)
          // =============================================
          // Условия:
          // 1. Функция включена
          // 2. Рабочий день открыт (STATUS === 'OPENED')
          // 3. Текущее время >= времени начала обеда
          // 4. Текущее время <= времени окончания обеда + 30 минут (grace period)
          // 5. Еще не отправляли уведомление сегодня
          // 6. Модальное окно еще не открыто
          // 7. Методы timeman доступны

          const isInLunchInterval = currentMinutes >= lunchStartMinutes && currentMinutes <= lunchEndMinutes
          const isWithinGracePeriod = currentMinutes <= lunchEndMinutes + 30 // 30 минут после окончания обеда
          const isValidTimeForStart = currentMinutes >= lunchStartMinutes && isWithinGracePeriod

          if (lunchStart.value.enabled &&
              isTimemanAvailable.value === true &&
              workDayParams.STATUS === 'OPENED' &&
              isValidTimeForStart &&
              !startNotificationSent &&
              !applicationOpened.value) {

            startNotificationSent = true

            console.log('Метод')
            console.log(lunchStart.value.method)
            console.log(lunchEnd.value.method)

            if (lunchStart.value.method === 'modal') {
              openLunchModal('start')
            } else if (lunchStart.value.method === 'auto') {
              pauseWorkday()
            } else if (lunchStart.value.method === 'chat') {
              sendChatNotification(userId, 'start')
            } else if (lunchStart.value.method === 'push') {
              sendPushNotification(userId, 'start')
            }
          }

          // =============================================
          // ПРОВЕРКА ДЛЯ ЗАВЕРШЕНИЯ ОБЕДА (возобновление)
          // =============================================
          // Условия:
          // 1. Функция включена
          // 2. Рабочий день на паузе (STATUS === 'PAUSED')
          // 3. Текущее время >= времени окончания обеда
          // 4. Текущее время <= времени окончания обеда + 60 минут (максимальная задержка)
          // 5. Еще не отправляли уведомление сегодня
          // 6. Модальное окно еще не открыто
          // 7. Методы timeman доступны

          const isAfterLunchEnd = currentMinutes >= lunchEndMinutes
          const isWithinMaxDelay = currentMinutes <= lunchEndMinutes + 60 // 60 минут после окончания обеда
          const isValidTimeForEnd = isAfterLunchEnd && isWithinMaxDelay

          if (lunchEnd.value.enabled &&
              isTimemanAvailable.value === true &&
              workDayParams.STATUS === 'PAUSED' &&
              isValidTimeForEnd &&
              !endNotificationSent &&
              !applicationOpened.value) {

            endNotificationSent = true

            if (lunchEnd.value.method === 'modal') {
              openLunchModal('end')
            } else if (lunchEnd.value.method === 'auto') {
              resumeWorkday()
            } else if (lunchEnd.value.method === 'chat') {
              sendChatNotification(userId, 'end')
            } else if (lunchEnd.value.method === 'push') {
              sendPushNotification(userId, 'end')
            }
          }

          // =============================================
          // ДОПОЛНИТЕЛЬНАЯ ЗАЩИТА ОТ ПОВТОРНЫХ ВЫЗОВОВ
          // =============================================
          // Если рабочий день уже на паузе - сбрасываем флаг начала обеда
          if (workDayParams.STATUS === 'PAUSED') {
            startNotificationSent = true
            setStoredFlag('lunch_start_notification_sent', 'true', 24)
          }

          // Если рабочий день активен - сбрасываем флаг завершения обеда
          if (workDayParams.STATUS === 'OPENED') {
            endNotificationSent = true
            setStoredFlag('lunch_end_notification_sent', 'true', 24)
          }
        }
    )
  })
}

// ==========================================================================
// ТАЙМЕРЫ И ЖИЗНЕННЫЙ ЦИКЛ
// ==========================================================================

function startPeriodicCheck(): void {
  if (periodicCheckInterval) {
    clearInterval(periodicCheckInterval)
  }

  periodicCheckInterval = setInterval(() => {
    checkWorkdayStatus()
  }, MODAL_CONFIG.CHECK_INTERVAL_SECONDS * 1000)
}

function stopPeriodicCheck(): void {
  if (periodicCheckInterval) {
    clearInterval(periodicCheckInterval)
    periodicCheckInterval = null
  }
}

function handleVisibilityChange(): void {
  const wasVisible = isPageVisible
  isPageVisible = !document.hidden

  if (isPageVisible && !wasVisible) {
    if (isTimemanAvailable.value === true) {
      startPeriodicCheck()
      checkWorkdayStatus()
    }
  } else if (!isPageVisible && wasVisible) {
    stopPeriodicCheck()
  }
}

// ==========================================================================
// ПУБЛИЧНЫЕ МЕТОДЫ (для использования извне)
// ==========================================================================

// Обновить индивидуальное время обеда пользователя
async function updateUserLunchTime(startTime: Time | null, endTime: Time | null): Promise<boolean> {
  const startStr = startTime ? `${startTime.hour.toString().padStart(2, '0')}:${startTime.minute.toString().padStart(2, '0')}` : null
  const endStr = endTime ? `${endTime.hour.toString().padStart(2, '0')}:${endTime.minute.toString().padStart(2, '0')}` : null

  const success = await saveUserLunchTimeToUserOptions(startStr, endStr)

  if (success) {
    // Сбрасываем флаги уведомлений после изменения времени
    startNotificationSent = false
    endNotificationSent = false
    deleteStoredFlag('lunch_start_notification_sent')
    deleteStoredFlag('lunch_end_notification_sent')
  }

  return success
}

// Получить текущее время обеда (для отображения)
function getCurrentLunchTimes(): { start: string, end: string } {
  return {
    start: getCurrentLunchStartTime(),
    end: getCurrentLunchEndTime()
  }
}

// Получить индивидуальное время обеда пользователя
function getUserLunchTime(): UserLunchTime {
  return {
    startTime: userLunchTime.value.startTime,
    endTime: userLunchTime.value.endTime
  }
}

// Экспортируем методы для использования в других компонентах
defineExpose({
  updateUserLunchTime,
  getCurrentLunchTimes,
  getUserLunchTime,
  loadUserLunchTimeFromUserOptions
})

// ==========================================================================
// ИНИЦИАЛИЗАЦИЯ
// ==========================================================================

onMounted(async () => {
  // Очищаем временные флаги
  clearTempFlags()

  // Устанавливаем обработчик видимости страницы
  document.addEventListener('visibilitychange', handleVisibilityChange)
  isPageVisible = !document.hidden

  if (typeof BX24 !== 'undefined' && BX24.init) {
    BX24.init(async () => {
      isBitrixLoaded.value = true

      // Проверяем доступность методов timeman
      isTimemanAvailable.value = await checkTimemanAvailability()

      if (!isTimemanAvailable.value) {
        console.warn('Методы timeman не доступны. Приложение будет работать в режиме ожидания.')
        await loadSettings()
        return
      }

      // Загружаем индивидуальные настройки пользователя из user.options
      await loadUserLunchTimeFromUserOptions()

      await loadSettings()

      // Выполняем первую проверку
      checkWorkdayStatus()

      // Запускаем периодическую проверку
      if (isPageVisible) {
        startPeriodicCheck()
      }
    })
  } else {
    console.warn('BX24 не обнаружен')
  }
})

onUnmounted(() => {
  document.removeEventListener('visibilitychange', handleVisibilityChange)
  stopPeriodicCheck()
  clearTempFlags()
})
</script>

<template>
  <!-- Пустой шаблон - модальные окна открываются через BX24.openApplication -->
</template>