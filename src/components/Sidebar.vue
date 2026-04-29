<script setup lang="ts">
import MailOutIcon from '@bitrix24/b24icons-vue/main/MailOutIcon'
import StarIcon from '@bitrix24/b24icons-vue/outline/AiStarsQuestionIcon'
import ShieldCheckIcon from '@bitrix24/b24icons-vue/outline/ShieldCheckedIcon'
import PowerIcon from '@bitrix24/b24icons-vue/outline/PowerIcon'
import SettingsIcon from '@bitrix24/b24icons-vue/outline/SettingsIcon'
import HomeIcon from '@bitrix24/b24icons-vue/outline/HomeIcon'
import DocumentUpdateIcon from '@bitrix24/b24icons-vue/outline/DocumentUpdateIcon'

import SolutionsSlider from './SolutionsSlider.vue'
import { ref, onMounted, onUnmounted } from 'vue'
import { useRouter, useRoute } from 'vue-router'

// (function(w: Window, d: Document, u: string) {
//   var s = d.createElement('script');
//   s.async = true;
//   s.src = u + '?' + (Date.now() / 60000 | 0);
//   var h = d.getElementsByTagName('script')[0];
//   h.parentNode.insertBefore(s, h);
// })(window, document, 'https://cdn-ru.bitrix24.ru/b37550306/crm/site_button/loader_1_bt7q7g.js');

const router = useRouter()
const route = useRoute()

// Состояние для проверки прав администратора
const isAdmin = ref(false)
let intervalId: number | null = null

// Функция проверки активного маршрута
const isRouteActive = (path: string): boolean => {
  return route.path === path
}

const handleReview = (): void => {
  if (typeof (window as any).BX24 !== 'undefined') {
    (window as any).BX24.init(() => {
      (window as any).BX24.openPath(
          '/marketplace/detail/tekhnogalera.avtomaticheskoe_nachalo_i_zavershenie_rabochego_dnya/',
          function(result: any) {
          }
      );
    });
  } else {
  }
};

// Функция перехода на страницу настроек
const goToSettings = (): void => {
  if (router && isAdmin.value) {
    router.push('/settings')
  }
};

// Функция перехода на главную
const goToMain = (): void => {
  if (router) {
    router.push('/')
  }
};

// Функция перехода на инструкции
const goToInstructions = (): void => {
  if (router) {
    router.push('/instructions')
  }
};

// Функция проверки прав администратора
const checkAdminRights = () => {
  if (typeof (window as any).BX24 !== 'undefined') {
    (window as any).BX24.init(() => {
      const adminStatus = (window as any).BX24.isAdmin()

      if (isAdmin.value !== adminStatus) {
        isAdmin.value = adminStatus
      }
    })
  } else {
    if (isAdmin.value !== false) {
      isAdmin.value = false
    }
  }
}

// Инициализация и запуск периодической проверки
const initialize = () => {
  checkAdminRights()

  if (intervalId === null) {
    intervalId = window.setInterval(() => {
      checkAdminRights()
    }, 5000)
  }
}

// Очистка интервала при размонтировании компонента
const cleanup = () => {
  if (intervalId !== null) {
    clearInterval(intervalId)
    intervalId = null
  }
}

onMounted(() => {
  initialize()
})

onUnmounted(() => {
  cleanup()
})
</script>

<template>
  <div class="lg:sticky lg:top-6 space-y-6">
    <!-- Карточка информации о приложении -->
    <B24Card>
      <div class="p-0 md:p-6">
        <div class="space-y-6">
          <!-- Логотип и название -->
          <div class="flex items-center space-x-3">
            <div class="w-12 h-12 bg-blue-100 rounded-lg flex items-center justify-center">
              <svg xmlns="http://www.w3.org/2000/svg" height="20px" viewBox="0 -960 960 960" width="20px" fill="#009DDB" class="w-6 h-6">
                <path d="M336-240h48v-241q26-4 43-24t17-47v-150.1q0-7.9-5-12.9t-13-5q-8 0-13 5t-5 13.15V-600h-30v-101.85q0-8.15-5-13.15t-13-5q-8 0-13 5t-5 13.15V-600h-30v-101.85q0-8.15-5-13.15t-13-5q-8 0-13 5t-5 12.9V-552q0 27 17 47t43 24v241Zm240 0h48v-246q26-11 43-41.78 17-30.77 17-72.1Q684-650 659.5-685 635-720 600-720t-59.5 35Q516-650 516-599.88q0 41.33 17 72.1Q550-497 576-486v246ZM168-96q-29.7 0-50.85-21.15Q96-138.3 96-168v-624q0-29.7 21.15-50.85Q138.3-864 168-864h624q29.7 0 50.85 21.15Q864-821.7 864-792v624q0 29.7-21.15 50.85Q821.7-96 792-96H168Zm0-72h624v-624H168v624Zm0 0v-624 624Z"/>
              </svg>
            </div>
            <div>
              <h3 class="text-lg font-semibold text-gray-900">На обед!</h3>
              <p class="text-sm text-gray-500">Версия 1.0.0</p>
            </div>
          </div>

          <div class="space-y-3">
            <h4 class="text-sm font-medium text-gray-900">Меню</h4>
            <!-- Панель навигации -->
            <nav class="space-y-2">
              <!-- Главная -->
              <div
                  @click="goToMain"
                  class="flex items-center px-3 py-2 text-sm font-medium rounded-md cursor-pointer"
                  :class="isRouteActive('/') ? 'bg-blue-100 text-blue-700' : 'text-gray-700 hover:bg-gray-100'"
              >
                <HomeIcon :class="isRouteActive('/') ? 'w-5 h-5 mr-3 text-blue-600' : 'w-5 h-5 mr-3 text-gray-500'" />
                Главная
              </div>

              <!-- Настройки - видно только администраторам -->
              <div
                  v-if="isAdmin"
                  @click="goToSettings"
                  class="flex items-center px-3 py-2 text-sm font-medium rounded-md cursor-pointer"
                  :class="isRouteActive('/settings') ? 'bg-blue-100 text-blue-700' : 'text-gray-700 hover:bg-gray-100'"
              >
                <SettingsIcon :class="isRouteActive('/settings') ? 'w-5 h-5 mr-3 text-blue-600' : 'w-5 h-5 mr-3 text-gray-500'" />
                Настройки
              </div>

              <!-- Заглушка для обычных пользователей -->
              <div
                  v-else
                  class="flex items-center px-3 py-2 text-sm font-medium rounded-md text-gray-400 cursor-not-allowed"
                  title="Доступно только администраторам"
              >
                <SettingsIcon class="w-5 h-5 mr-3 text-gray-400" />
                Настройки
              </div>

              <!-- Инструкция -->
              <div
                  @click="goToInstructions"
                  class="flex items-center px-3 py-2 text-sm font-medium rounded-md cursor-pointer"
                  :class="isRouteActive('/instructions') ? 'bg-blue-100 text-blue-700' : 'text-gray-700 hover:bg-gray-100'"
              >
                <DocumentUpdateIcon :class="isRouteActive('/instructions') ? 'w-5 h-5 mr-3 text-blue-600' : 'w-5 h-5 mr-3 text-gray-500'" />
                Инструкция
              </div>

              <!-- Техническая поддержка -->
              <a
                  href="mailto:technogalera@yandex.ru?subject=Поддержка приложения На обед!"
                  class="flex items-center px-3 py-2 text-sm font-medium rounded-md text-gray-700 hover:bg-gray-100"
              >
                <MailOutIcon class="w-5 h-5 mr-3 text-gray-500" />
                Техническая поддержка
              </a>

              <!-- Оставить отзыв -->
              <div
                  @click="handleReview"
                  class="flex items-center px-3 py-2 text-sm font-medium rounded-md text-gray-700 hover:bg-gray-100 cursor-pointer"
              >
                <StarIcon class="w-5 h-5 mr-3 text-gray-500" />
                Оставить отзыв
              </div>
            </nav>
          </div>
        </div>
      </div>
    </B24Card>

    <!-- Слайдер с другими решениями -->
    <SolutionsSlider />
  </div>
</template>

<style scoped>
.cursor-pointer {
  cursor: pointer;
}

.cursor-not-allowed {
  cursor: not-allowed;
}

@media (max-width: 640px) {
  .text-3xl {
    font-size: 1.875rem;
  }

  .text-xl {
    font-size: 1.25rem;
  }
}
</style>