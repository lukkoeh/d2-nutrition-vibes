<script setup lang="ts">
import { ref, computed, onMounted } from 'vue'
import { useRoute, useRouter } from 'vue-router'
import { useUIStore } from '@/stores/useUIStore'
import { useTourStore } from '@/tour/stores/useTourStore'

interface Props {
  collapsed?: boolean
}

const props = withDefaults(defineProps<Props>(), {
  collapsed: false
})

const emit = defineEmits<{
  toggle: []
}>()

const route = useRoute()
const router = useRouter()
const uiStore = useUIStore()
const tourStore = useTourStore()

const isNewUser = ref(false)

onMounted(() => {
  const hasCompletedTour = localStorage.getItem('tourCompleted') === 'true'
  const hasSeenTour = localStorage.getItem('tourSeen') === 'true'
  isNewUser.value = !hasCompletedTour && !hasSeenTour
})

const navigationItems = [
  {
    label: 'Dashboard',
    panel: 'dashboard',
    icon: 'home',
    description: 'Übersicht und Hauptsteuerung'
  },
  {
    label: 'Zeitreihen',
    panel: 'timeseries',
    icon: 'chart-line',
    description: 'Zeitreihenanalyse und Trends'
  },
  {
    label: 'Simulation',
    panel: 'simulation',
    icon: 'cog',
    description: 'Szenario-Simulationen'
  },
  {
    label: 'ML Prognosen',
    panel: 'ml-predictions',
    icon: 'brain',
    description: 'Machine Learning Vorhersagen'
  },
]

const sidebarClasses = computed(() => [
  'fixed',
  'top-0',
  'left-0',
  'h-screen',
  'bg-white',
  'dark:bg-gray-800',
  'border-r',
  'border-gray-200',
  'dark:border-gray-700',
  'transition-all',
  'duration-300',
  'ease-in-out',
  'z-40', // Higher z-index to overlay content
  'shadow-lg', // Add shadow for overlay effect
  // Responsive width handling
  props.collapsed 
    ? 'w-16' 
    : 'w-64'
].join(' '))

const isActive = (panelName: string) => uiStore.currentPanel === panelName

const navigateTo = (panelName: string) => {
  uiStore.setCurrentPanel(panelName)
  uiStore.showPanel(panelName)
  router.push({ path: '/', query: { pnl: panelName } })
}

const getIconSvg = (iconName: string) => {
  const icons: Record<string, string> = {
    home: 'M3 12l2-2m0 0l7-7 7 7M5 10v10a1 1 0 001 1h3m10-11l2 2m-2-2v10a1 1 0 01-1 1h-3m-6 0a1 1 0 001-1v-4a1 1 0 011-1h2a1 1 0 011 1v4a1 1 0 001 1m-6 0h6',
    'chart-line': 'M7 12l3-3 3 3 4-4M8 21l4-4 4 4M3 4h18M4 4h16v12a1 1 0 01-1 1H5a1 1 0 01-1-1V4z',
    cog: 'M12 6V4m0 2a2 2 0 100 4m0-4a2 2 0 110 4m-6 8a2 2 0 100-4m0 4a2 2 0 100 4m0-4v2m0-6V4m6 6v10m6-2a2 2 0 100-4m0 4a2 2 0 100 4m0-4v2m0-6V4',
    brain: 'M9.75 17L9 20l-1 1h8l-1-1-.75-3M3 13h18M5 17h14a2 2 0 002-2V9a2 2 0 00-2-2H5a2 2 0 00-2 2v6a2 2 0 002 2z'
  }
  return icons[iconName] || icons.home
}

const toggleDarkModeWithNotification = () => {
  uiStore.toggleDarkMode()
}

const startTour = () => {
  localStorage.setItem('tourSeen', 'true')
  isNewUser.value = false
  tourStore.startTour('main')
}

const resumeTour = () => {
  tourStore.resumeTour()
}
</script>

<template>
  <aside :class="sidebarClasses">
    <!-- Sidebar Header -->
    <div class="flex items-center justify-between p-4 border-b border-gray-200 dark:border-gray-700 h-16">
      <div class="flex items-center flex-1">
        <div v-if="!collapsed" class="flex items-center justify-center w-full">
          <!-- Light mode logo -->
          <img 
            v-if="!uiStore.darkMode"
            src="/logolight.png" 
            alt="OpenFoodMap Logo" 
            class="h-8 object-contain max-w-full"
          />
          <!-- Dark mode logo -->
          <img 
            v-else
            src="/logodark.png" 
            alt="OpenFoodMap Logo" 
            class="h-8 object-contain max-w-full"
          />
        </div>
        
        <!-- Logo for collapsed state -->
        <div v-if="collapsed" class="flex items-center justify-center w-full">
          <!-- Light mode logo -->
          <img 
            v-if="!uiStore.darkMode"
            src="/logolight.png" 
            alt="OpenFoodMap Logo" 
            class="h-8 w-8 object-contain"
          />
          <!-- Dark mode logo -->
          <img 
            v-else
            src="/logodark.png" 
            alt="OpenFoodMap Logo" 
            class="h-8 w-8 object-contain"
          />
        </div>
      </div>
      
      <!-- Toggle Button -->
      <button
        class="p-2 rounded-lg hover:bg-gray-100 dark:hover:bg-gray-700 transition-colors"
        :title="collapsed ? 'Sidebar erweitern' : 'Sidebar einklappen'"
        @click="emit('toggle')"
      >
        <svg 
          class="w-5 h-5 text-gray-600 dark:text-gray-400 transition-transform"
          :class="{ 'rotate-180': collapsed }"
          fill="none" 
          stroke="currentColor" 
          viewBox="0 0 24 24"
        >
          <path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M11 19l-7-7 7-7m8 14l-7-7 7-7" />
        </svg>
      </button>
    </div>
    
    <!-- Navigation Menu -->
    <nav class="flex-1 p-2 space-y-1 overflow-y-auto">
      <div
        v-for="item in navigationItems"
        :key="item.panel"
        class="relative"
      >
        <button
          :class="[
            'w-full flex items-center rounded-lg transition-all duration-200 group',
            collapsed ? 'px-2 py-2 justify-center' : 'px-3 py-2 text-sm font-medium',
            isActive(item.panel)
              ? 'bg-primary-100 dark:bg-primary-900/20 text-primary-700 dark:text-primary-300'
              : 'text-gray-700 dark:text-gray-300 hover:bg-gray-100 dark:hover:bg-gray-700 hover:text-gray-900 dark:hover:text-gray-100'
          ]"
          :title="collapsed ? item.label : ''"
          @click="navigateTo(item.panel)"
        >
          <!-- Icon -->
          <svg 
            class="flex-shrink-0"
            :class="collapsed ? 'w-5 h-5' : 'w-4 h-4 mr-3'"
            fill="none" 
            stroke="currentColor" 
            viewBox="0 0 24 24"
          >
            <path 
              stroke-linecap="round" 
              stroke-linejoin="round" 
              stroke-width="2" 
              :d="getIconSvg(item.icon)"
            />
          </svg>
          
          <!-- Label -->
          <span 
            v-if="!collapsed"
            class="truncate text-sm"
          >
            {{ item.label }}
          </span>
          
          <!-- Active Indicator -->
          <div
            v-if="isActive(item.panel) && !collapsed"
            class="ml-auto w-1.5 h-1.5 bg-primary-500 rounded-full"
          />
        </button>
        
        <!-- Tooltip for collapsed state -->
        <div
          v-if="collapsed"
          class="absolute left-full top-1/2 transform -translate-y-1/2 ml-2 px-2 py-1 bg-gray-900 dark:bg-gray-50 text-white dark:text-gray-900 text-xs rounded opacity-0 pointer-events-none group-hover:opacity-100 transition-opacity z-50 whitespace-nowrap"
        >
          {{ item.label }}
          <div class="absolute left-0 top-1/2 transform -translate-x-1 -translate-y-1/2 w-2 h-2 bg-gray-900 dark:bg-gray-50 rotate-45"></div>
        </div>
      </div>
    </nav>
    
    <!-- Sidebar Footer -->
    <div class="p-2 border-t border-gray-200 dark:border-gray-700 space-y-2">
      <!-- Tour Button -->
      <button
        :class="[
          'w-full flex items-center rounded-lg transition-all duration-200 relative',
          collapsed ? 'px-2 py-2 justify-center' : 'px-3 py-2 text-sm font-medium',
          'text-gray-700 dark:text-gray-300 hover:bg-gray-100 dark:hover:bg-gray-700'
        ]"
        :title="collapsed ? 'Tour starten' : ''"
        @click="tourStore.isActive ? resumeTour() : startTour()"
      >
        <!-- Pulse animation for new users -->
        <div 
          v-if="isNewUser && !tourStore.isActive"
          class="absolute inset-0 rounded-lg"
        >
          <div class="absolute inset-0 bg-primary-500 opacity-20 rounded-lg animate-pulse"></div>
        </div>
        
        <svg 
          class="flex-shrink-0 relative"
          :class="collapsed ? 'w-5 h-5' : 'w-4 h-4 mr-3'"
          fill="none" 
          stroke="currentColor" 
          viewBox="0 0 24 24"
        >
          <path 
            stroke-linecap="round" 
            stroke-linejoin="round" 
            stroke-width="2" 
            d="M12 6.253v13m0-13C10.832 5.477 9.246 5 7.5 5S4.168 5.477 3 6.253v13C4.168 18.477 5.754 18 7.5 18s3.332.477 4.5 1.253m0-13C13.168 5.477 14.754 5 16.5 5c1.747 0 3.332.477 4.5 1.253v13C19.832 18.477 18.247 18 16.5 18c-1.746 0-3.332.477-4.5 1.253"
          />
        </svg>
        
        <span v-if="!collapsed" class="text-sm relative">
          {{ tourStore.isActive ? 'Tour läuft...' : 'Tour starten' }}
        </span>
        
        <!-- New user indicator -->
        <span 
          v-if="isNewUser && !tourStore.isActive && !collapsed" 
          class="ml-auto bg-primary-500 text-white text-xs px-2 py-0.5 rounded-full animate-pulse"
        >
          NEU
        </span>
      </button>

      <!-- Dark Mode Toggle -->
      <button
        :class="[
          'w-full flex items-center rounded-lg transition-all duration-200',
          collapsed ? 'px-2 py-2 justify-center' : 'px-3 py-2 text-sm font-medium',
          'text-gray-700 dark:text-gray-300 hover:bg-gray-100 dark:hover:bg-gray-700'
        ]"
        :title="collapsed ? (uiStore.darkMode ? 'Hell-Modus' : 'Dunkel-Modus') : ''"
        @click="toggleDarkModeWithNotification"
      >
        <svg 
          class="flex-shrink-0"
          :class="collapsed ? 'w-5 h-5' : 'w-4 h-4 mr-3'"
          fill="none" 
          stroke="currentColor" 
          viewBox="0 0 24 24"
        >
          <path 
            v-if="uiStore.darkMode"
            stroke-linecap="round" 
            stroke-linejoin="round" 
            stroke-width="2" 
            d="M12 3v1m0 16v1m9-9h-1M4 12H3m15.364 6.364l-.707-.707M6.343 6.343l-.707-.707m12.728 0l-.707.707M6.343 17.657l-.707.707M16 12a4 4 0 11-8 0 4 4 0 018 0z"
          />
          <path 
            v-else
            stroke-linecap="round" 
            stroke-linejoin="round" 
            stroke-width="2" 
            d="M20.354 15.354A9 9 0 018.646 3.646 9.003 9.003 0 0012 21a9.003 9.003 0 008.354-5.646z"
          />
        </svg>
        
        <span v-if="!collapsed" class="text-sm">
          {{ uiStore.darkMode ? 'Hell-Modus' : 'Dunkel-Modus' }}
        </span>
      </button>
    </div>
  </aside>
</template>

<style scoped>
/* Smooth transitions for all elements */
.transition-all {
  transition-property: all;
  transition-timing-function: cubic-bezier(0.4, 0, 0.2, 1);
  transition-duration: 300ms;
}

/* Ensure sidebar doesn't cause horizontal overflow */
aside {
  min-width: 4rem; /* Minimum width for collapsed state */
  max-width: 16rem; /* Maximum width for expanded state */
  flex-shrink: 0; /* Prevent shrinking */
  /* Stellen Sie sicher, dass erweiterte Sidebar über dem Content liegt */
  position: fixed;
  top: 0;
  left: 0;
  height: 100vh;
}

/* Hide scrollbar completely or make it very minimal */
nav {
  scrollbar-width: none; /* Firefox */
  -ms-overflow-style: none; /* IE/Edge */
}

nav::-webkit-scrollbar {
  display: none; /* Chrome/Safari/WebKit */
}

/* Alternative: Very thin scrollbar if needed */
nav.show-scrollbar::-webkit-scrollbar {
  width: 2px;
}

nav.show-scrollbar::-webkit-scrollbar-track {
  background: transparent;
}

nav.show-scrollbar::-webkit-scrollbar-thumb {
  background: rgba(156, 163, 175, 0.3);
  border-radius: 1px;
}

.dark nav.show-scrollbar::-webkit-scrollbar-thumb {
  background: rgba(75, 85, 99, 0.3);
}

/* Hover effects for navigation items */
.group:hover .opacity-0 {
  opacity: 1;
}

/* Mobile responsiveness */
@media (max-width: 768px) {
  aside {
    /* Auf kleineren Bildschirmen als slide-in overlay */
    transform: translateX(-100%);
    transition: transform 0.3s ease-in-out;
    z-index: 50;
  }
  
  aside:not(.collapsed) {
    transform: translateX(0);
  }
  
  /* Backdrop für mobile overlay wenn expanded */
  aside:not(.collapsed)::after {
    content: '';
    position: fixed;
    top: 0;
    left: 0;
    right: 0;
    bottom: 0;
    background: rgba(0, 0, 0, 0.5);
    z-index: -1;
    pointer-events: auto;
  }
}

/* Prevent text selection on navigation items */
nav button {
  user-select: none;
  -webkit-user-select: none;
  -moz-user-select: none;
  -ms-user-select: none;
}
</style>