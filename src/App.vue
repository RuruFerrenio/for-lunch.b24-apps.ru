<script setup lang="ts">
import { computed, onMounted, ref } from 'vue'
import { useRoute } from 'vue-router'
import Sidebar from './components/Sidebar.vue'
import LunchManager from './components/LunchManager.vue'

const route = useRoute()

// ==========================================================================
// ФУНКЦИИ ДЛЯ РАБОТЫ С КУКИ (УНИФИЦИРОВАННЫЕ)
// ==========================================================================

function getCookie(name: string): string | null {
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

function deleteCookie(name: string): void {
  if (typeof document === 'undefined') return
  document.cookie = `${name}=; expires=Thu, 01 Jan 1970 00:00:00 UTC; path=/; SameSite=None; Secure`
}

// Очистка временных флагов (используем ту же логику, что в основном компоненте)
function clearTempFlags(): void {
  deleteCookie('open_app_mode')
  deleteCookie('modal_type')
}

// ==========================================================================
// СОСТОЯНИЯ КОМПОНЕНТА
// ==========================================================================

const openAppMode = ref<'default' | 'lunch'>('default')
const lunchMode = ref<'start' | 'end' | null>(null)
const isLoading = ref(true)

// Теперь showSidebar зависит от openAppMode
const showSidebar = computed(() => {
  // Если мы в lunch режиме - сайдбар не показываем
  if (openAppMode.value !== 'default') return false

  // В default режиме показываем на определенных страницах
  return route.path === '/' ||
      route.path === '/settings' ||
      route.path === '/instructions'
})

// ==========================================================================
// ИНИЦИАЛИЗАЦИЯ
// ==========================================================================

onMounted(() => {
  // Читаем куки при загрузке (используем те же имена, что в основном компоненте)
  const openAppModeValue = getCookie('open_app_mode')
  const lunchType = getCookie('modal_type')

  if (openAppModeValue === 'modal') {
    openAppMode.value = 'lunch'
    if (lunchType === 'start' || lunchType === 'end') {
      lunchMode.value = lunchType
    }
  } else {
    openAppMode.value = 'default'
  }

  isLoading.value = false

  // Загружаем Bitrix24 API
  const script = document.createElement('script')
  script.src = '//api.bitrix24.com/api/v1/'
  script.async = true
  document.head.appendChild(script)

  // Очищаем временные флаги после загрузки, если они не используются
  // (можно оставить или удалить в зависимости от логики)
  // clearTempFlags()
})

// ==========================================================================
// ОЧИСТКА ПРИ РАЗМОНТИРОВАНИИ
// ==========================================================================

// Обратите внимание: onUnmounted добавлять не нужно, так как куки
// очищаются в основном компоненте LunchManager после закрытия модалки
</script>

<template>
  <div v-if="!isLoading">
    <Suspense>
      <B24App>
        <div class="p-0 md:p-6">
          <div class="mt-0 md:mt-2 grid grid-cols-1 lg:grid-cols-3 gap-6 md:gap-8">
            <!-- Если кука default - показываем основной контент + сайдбар (только на нужных страницах) -->
            <template v-if="openAppMode === 'default'">
              <!-- Контент занимает всю ширину, если сайдбар не показывается -->
              <div :class="showSidebar ? 'lg:col-span-2' : 'lg:col-span-3'">
                <RouterView />
              </div>
              <!-- Сайдбар - показываем только на указанных страницах -->
              <div v-if="showSidebar" class="lg:col-span-1">
                <Sidebar />
              </div>
            </template>

            <!-- Если кука lunch - показываем LunchManager -->
            <div v-else-if="openAppMode === 'lunch'" class="lg:col-span-3">
              <LunchManager
                  v-if="lunchMode"
                  :mode="lunchMode"
                  :auto-close-delay="2000"
                  @closed="openAppMode = 'default'"
              />
              <div v-else class="text-center py-12">
                <p class="text-gray-500">Не указан тип операции для обеда</p>
              </div>
            </div>
          </div>
        </div>
      </B24App>
    </Suspense>
  </div>
  <div v-else class="text-center py-12">
    Загрузка...
  </div>
</template>