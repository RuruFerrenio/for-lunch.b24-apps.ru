<script setup lang="ts">
import { ref, computed, onMounted, onUnmounted } from 'vue'
import { markRaw } from 'vue'
import ClockIcon from '@bitrix24/b24icons-vue/outline/ClockIcon'
import CalculatorIcon from '@bitrix24/b24icons-vue/main/CalculatorIcon'
import PowerIcon from '@bitrix24/b24icons-vue/outline/PowerIcon'
import ArrowToTheLeftIcon from '@bitrix24/b24icons-vue/actions/ArrowToTheLeftIcon'
import ArrowToTheRightIcon from '@bitrix24/b24icons-vue/actions/ArrowToTheRightIcon'
import MarketIcon from '@bitrix24/b24icons-vue/outline/MarketIcon'

// Тип для решения
interface Solution {
  id: number
  title: string
  description: string
  iconComponent: any
  color: string
  features: string[]
  link: string
  installed: boolean
  badge: string | null
  badgeClass: string
}

// Состояние развернутых карточек
const expandedFeatures = ref<Record<number, boolean>>({})

// Функция для переключения состояния
const toggleFeatures = (solutionId: number) => {
  expandedFeatures.value[solutionId] = !expandedFeatures.value[solutionId]
}

// Проверка, развернута ли карточка
const isExpanded = (solutionId: number) => {
  return expandedFeatures.value[solutionId] || false
}

// Получение видимых элементов features
const getVisibleFeatures = (solutionId: number) => {
  const solution = solutions.find(s => s.id === solutionId)
  if (!solution) return []

  if (isExpanded(solutionId)) {
    return solution.features
  } else {
    return solution.features.slice(0, 3)
  }
}

// Функция для открытия ссылки решения
const handleReview = (link: string): void => {
  if (typeof (window as any).BX24 !== 'undefined') {
    (window as any).BX24.init(() => {
      (window as any).BX24.openPath(
          link,
          function(result: any) {
          }
      );
    });
  } else {
    // Fallback если BX24 не доступен
    window.open(link, '_blank');
  }
};

// Данные решений (2 элемента)
const solutions: Solution[] = [
  {
    id: 1,
    title: 'Удобное начало и завершение рабочего дня',
    description: 'Забываете начать рабочий день ? Больше не забудете.',
    iconComponent: markRaw(PowerIcon),
    color: 'bg-green-500',
    features: ['Автоматический контроль начала рабочего дня', 'Автоматический контроль завершения рабочего дня', 'Бесшовная интеграция с интерфейсом битрикс24', 'Гибкая настройка'],
    link: '/marketplace/detail/tekhnogalera.avtomaticheskoe_nachalo_i_zavershenie_rabochego_dnya/',
    installed: false,
    badge: 'Новинка',
    badgeClass: 'bg-blue-100 text-blue-800'
  },
  {
    id: 2,
    title: 'Чистое время',
    description: 'Комплексное улучшение учета рабочего времени и не только.',
    iconComponent: markRaw(ClockIcon),
    color: 'bg-blue-500',
    features: ['Журнал посещенных страниц', 'Упрощение процедуры отчета о затраченном времени для сотрудников', 'Пассивная защита от бездействия', 'Контроль начала и завершения рабочего дня', 'Глубокий аудит временных затрат', 'Мгновенные отчеты о занятости', 'Детальная статистика по активности', 'Адаптация под мобильные устройства'],
    link: '/marketplace/detail/tekhnogalera.chistoe_vremya/',
    installed: false,
    badge: 'Новинка',
    badgeClass: 'bg-blue-100 text-blue-800'
  },
]

const currentIndex = ref(0)
const autoplayInterval = ref<number | null>(null)
const totalSlides = computed(() => solutions.length)

// Навигация
const nextSlide = () => {
  if (currentIndex.value < totalSlides.value - 1) {
    currentIndex.value++
  }
}

const prevSlide = () => {
  if (currentIndex.value > 0) {
    currentIndex.value--
  }
}

const goToSlide = (index: number) => {
  if (index >= 0 && index < totalSlides.value) {
    currentIndex.value = index
  }
}

// Автопрокрутка
const startAutoplay = () => {
  stopAutoplay()
  autoplayInterval.value = window.setInterval(() => {
    if (currentIndex.value < totalSlides.value - 1) {
      currentIndex.value++
    } else {
      currentIndex.value = 0
    }
  }, 5000)
}

const stopAutoplay = () => {
  if (autoplayInterval.value) {
    clearInterval(autoplayInterval.value)
    autoplayInterval.value = null
  }
}

onMounted(() => {
  startAutoplay()
})

onUnmounted(() => {
  stopAutoplay()
})
</script>

<template>
  <div>
    <B24Card>
      <div class="p-6">
        <div class="space-y-4">
          <!-- Заголовок слайдера -->
          <div>
            <h3 class="text-lg font-semibold text-gray-900 mb-1">Другие наши решения</h3>
            <p class="text-sm text-gray-500">Расширьте возможности вашего портала</p>
          </div>

          <!-- Контейнер слайдера -->
          <div
              class="relative"
              @mouseenter="stopAutoplay"
              @mouseleave="startAutoplay"
          >
            <div class="overflow-hidden rounded-lg">
              <div
                  class="flex transition-transform duration-500 ease-in-out"
                  :style="{ transform: `translateX(-${currentIndex * 100}%)` }"
              >
                <div
                    v-for="(solution, idx) in solutions"
                    :key="solution.id"
                    class="w-full flex-shrink-0"
                >
                  <div class="p-0">
                    <!-- Иконка решения -->
                    <div class="p-4 pb-0 flex items-center justify-between">
                      <div
                          class="w-12 h-12 rounded-lg flex items-center justify-center"
                          :class="solution.color"
                      >
                        <component :is="solution.iconComponent" class="w-6 h-6 text-white" />
                      </div>
                      <div class="flex items-center space-x-2">
                        <span
                            v-if="solution.badge"
                            class="px-2 py-1 text-xs font-medium rounded-full"
                            :class="solution.badgeClass"
                        >
                          {{ solution.badge }}
                        </span>
                      </div>
                    </div>

                    <!-- Название и описание -->
                    <div class="p-4 pt-3">
                      <h4 class="text-base font-semibold text-gray-900 mb-2">
                        {{ solution.title }}
                      </h4>
                      <p class="text-sm text-gray-600 mb-4">
                        {{ solution.description }}
                      </p>

                      <!-- Особенности с возможностью развернуть -->
                      <div class="space-y-1.5 mb-4">
                        <!-- Отображаем видимые элементы -->
                        <div
                            v-for="(feature, index) in getVisibleFeatures(solution.id)"
                            :key="index"
                            class="flex items-center text-sm text-gray-500"
                        >
                          <svg class="w-3.5 h-3.5 mr-2 text-green-500 flex-shrink-0" fill="none" stroke="currentColor" viewBox="0 0 24 24">
                            <path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M5 13l4 4L19 7" />
                          </svg>
                          <span class="truncate">{{ feature }}</span>
                        </div>

                        <!-- Кнопка "Показать еще" -->
                        <div v-if="solution.features.length > 3" class="mt-2">
                          <button
                              @click="toggleFeatures(solution.id)"
                              class="text-sm text-blue-600 hover:text-blue-800 flex items-center gap-1 transition-colors"
                          >
                            <span>{{ isExpanded(solution.id) ? 'Свернуть' : 'Показать еще' }}</span>
                            <svg
                                class="w-3.5 h-3.5 transition-transform duration-200"
                                :class="{ 'rotate-180': isExpanded(solution.id) }"
                                fill="none"
                                stroke="currentColor"
                                viewBox="0 0 24 24"
                            >
                              <path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M19 9l-7 7-7-7" />
                            </svg>
                          </button>
                        </div>
                      </div>

                      <!-- Кнопка действия -->
                      <div>
                        <B24Button
                            :variant="solution.installed ? 'outline' : 'primary'"
                            @click="handleReview(solution.link)"
                            color="air-primary"
                            class="w-full justify-center"
                        >
                          {{ solution.installed ? 'Открыть' : 'Подробнее' }}
                          <MarketIcon class="w-4 h-4 ml-1" />
                        </B24Button>
                      </div>
                    </div>
                  </div>
                </div>
              </div>
            </div>

            <!-- Индикаторы прогресса -->
            <div class="mt-4 flex items-center justify-between">
              <div class="text-xs text-gray-500">
                {{ currentIndex + 1 }} из {{ totalSlides }}
              </div>
              <div class="flex space-x-1">
                <button
                    v-for="index in totalSlides"
                    :key="index"
                    @click="goToSlide(index - 1)"
                    class="w-1.5 h-1.5 rounded-full transition-all duration-300"
                    :class="currentIndex === index - 1 ? 'bg-blue-600 w-4' : 'bg-gray-300 hover:bg-gray-400'"
                    :aria-label="`Перейти к слайду ${index}`"
                />
              </div>
            </div>
          </div>

          <!-- Кнопки навигации -->
          <div class="flex justify-between gap-2">
            <B24Button
                variant="secondary"
                size="small"
                @click="prevSlide"
                :disabled="currentIndex === 0"
                class="flex-1 justify-center"
            >
              <ArrowToTheLeftIcon class="w-4 h-4 mr-1" />
              Назад
            </B24Button>
            <B24Button
                variant="secondary"
                size="small"
                @click="nextSlide"
                :disabled="currentIndex === totalSlides - 1"
                class="flex-1 justify-center"
            >
              Далее
              <ArrowToTheRightIcon class="w-4 h-4 ml-1" />
            </B24Button>
          </div>
        </div>
      </div>
    </B24Card>
  </div>
</template>

<style scoped>
/* Анимация для перехода слайдов */
.transition-transform {
  transition-timing-function: cubic-bezier(0.4, 0, 0.2, 1);
}

/* Кнопки навигации */
button:disabled {
  cursor: not-allowed;
  opacity: 0.5;
}

button:not(:disabled):hover {
  background-color: rgba(0, 0, 0, 0.05);
}
</style>