<template>
  <div id="app">
    <div class="app-shell" :class="{ 'is-collapsed': isCollapsed }">
      <!-- Fixed sidebar -->
      <aside class="sidebar">
        <div class="sidebar-logo">
          <div class="sidebar-logo-text">
            <h1>{{ t('nav.companyName') }}</h1>
            <span class="sidebar-subtitle">{{ t('nav.subtitle') }}</span>
          </div>
          <button class="sidebar-toggle" @click="toggleSidebar" :aria-label="isCollapsed ? 'Expand sidebar' : 'Collapse sidebar'">
            <span class="toggle-icon">{{ isCollapsed ? '›' : '‹' }}</span>
          </button>
        </div>

        <nav class="sidebar-nav">
          <router-link to="/" class="nav-item" exact-active-class="nav-item--active" data-label="Overview">
            <span class="nav-icon">⊞</span>
            <span class="nav-label">{{ t('nav.overview') }}</span>
          </router-link>
          <router-link to="/inventory" class="nav-item" active-class="nav-item--active" data-label="Inventory">
            <span class="nav-icon">▤</span>
            <span class="nav-label">{{ t('nav.inventory') }}</span>
          </router-link>
          <router-link to="/orders" class="nav-item" active-class="nav-item--active" data-label="Orders">
            <span class="nav-icon">◫</span>
            <span class="nav-label">{{ t('nav.orders') }}</span>
          </router-link>
          <router-link to="/spending" class="nav-item" active-class="nav-item--active" data-label="Finance">
            <span class="nav-icon">◈</span>
            <span class="nav-label">{{ t('nav.finance') }}</span>
          </router-link>
          <router-link to="/demand" class="nav-item" active-class="nav-item--active" data-label="Demand Forecast">
            <span class="nav-icon">◎</span>
            <span class="nav-label">{{ t('nav.demandForecast') }}</span>
          </router-link>
          <router-link to="/reports" class="nav-item" active-class="nav-item--active" data-label="Reports">
            <span class="nav-icon">≡</span>
            <span class="nav-label">Reports</span>
          </router-link>
        </nav>

        <div class="sidebar-spacer"></div>

        <div class="sidebar-footer">
          <LanguageSwitcher />
          <ProfileMenu
            @show-profile-details="showProfileDetails = true"
            @show-tasks="showTasks = true"
          />
        </div>
      </aside>

      <!-- Content area -->
      <div class="content-wrapper">
        <FilterBar />
        <main class="main-content">
          <router-view />
        </main>
      </div>
    </div>

    <ProfileDetailsModal
      :is-open="showProfileDetails"
      @close="showProfileDetails = false"
    />

    <TasksModal
      :is-open="showTasks"
      :tasks="tasks"
      @close="showTasks = false"
      @add-task="addTask"
      @delete-task="deleteTask"
      @toggle-task="toggleTask"
    />
  </div>
</template>

<script>
import { ref, onMounted, computed } from 'vue'
import { api } from './api'
import { useAuth } from './composables/useAuth'
import { useI18n } from './composables/useI18n'
import FilterBar from './components/FilterBar.vue'
import ProfileMenu from './components/ProfileMenu.vue'
import ProfileDetailsModal from './components/ProfileDetailsModal.vue'
import TasksModal from './components/TasksModal.vue'
import LanguageSwitcher from './components/LanguageSwitcher.vue'

export default {
  name: 'App',
  components: {
    FilterBar,
    ProfileMenu,
    ProfileDetailsModal,
    TasksModal,
    LanguageSwitcher
  },
  setup() {
    const { currentUser } = useAuth()
    const { t } = useI18n()

    const isCollapsed = ref(false)

    function toggleSidebar() {
      isCollapsed.value = !isCollapsed.value
    }

    function initSidebarState() {
      if (window.innerWidth < 1024) isCollapsed.value = true
    }

    const showProfileDetails = ref(false)
    const showTasks = ref(false)
    const apiTasks = ref([])

    // Merge mock tasks from currentUser with API tasks
    const tasks = computed(() => {
      return [...currentUser.value.tasks, ...apiTasks.value]
    })

    const loadTasks = async () => {
      try {
        apiTasks.value = await api.getTasks()
      } catch (err) {
        console.error('Failed to load tasks:', err)
      }
    }

    const addTask = async (taskData) => {
      try {
        const newTask = await api.createTask(taskData)
        // Add new task to the beginning of the array
        apiTasks.value.unshift(newTask)
      } catch (err) {
        console.error('Failed to add task:', err)
      }
    }

    const deleteTask = async (taskId) => {
      try {
        // Check if it's a mock task (from currentUser)
        const isMockTask = currentUser.value.tasks.some(t => t.id === taskId)

        if (isMockTask) {
          // Remove from mock tasks
          const index = currentUser.value.tasks.findIndex(t => t.id === taskId)
          if (index !== -1) {
            currentUser.value.tasks.splice(index, 1)
          }
        } else {
          // Remove from API tasks
          await api.deleteTask(taskId)
          apiTasks.value = apiTasks.value.filter(t => t.id !== taskId)
        }
      } catch (err) {
        console.error('Failed to delete task:', err)
      }
    }

    const toggleTask = async (taskId) => {
      try {
        // Check if it's a mock task (from currentUser)
        const mockTask = currentUser.value.tasks.find(t => t.id === taskId)

        if (mockTask) {
          // Toggle mock task status
          mockTask.status = mockTask.status === 'pending' ? 'completed' : 'pending'
        } else {
          // Toggle API task
          const updatedTask = await api.toggleTask(taskId)
          const index = apiTasks.value.findIndex(t => t.id === taskId)
          if (index !== -1) {
            apiTasks.value[index] = updatedTask
          }
        }
      } catch (err) {
        console.error('Failed to toggle task:', err)
      }
    }

    onMounted(() => {
      initSidebarState()
      loadTasks()
    })

    return {
      t,
      isCollapsed,
      toggleSidebar,
      showProfileDetails,
      showTasks,
      tasks,
      addTask,
      deleteTask,
      toggleTask
    }
  }
}
</script>

<style>
:root {
  --sidebar-width: 240px;
  --sidebar-collapsed-width: 60px;
  --sidebar-bg:            #0f172a;
  --sidebar-border:        rgba(255, 255, 255, 0.07);
  --sidebar-text:          #94a3b8;
  --sidebar-text-active:   #f1f5f9;
  --sidebar-hover-bg:      rgba(255, 255, 255, 0.05);
  --sidebar-active-bg:     rgba(59, 130, 246, 0.14);
  --sidebar-active-border: #3b82f6;
  --content-bg:  #f8fafc;
  --surface:     #ffffff;
  --border:      #e2e8f0;
  --border-mid:  #cbd5e1;
  --text-1: #0f172a;
  --text-2: #334155;
  --text-3: #64748b;
  --text-4: #94a3b8;
  --accent:       #3b82f6;
  --accent-hover: #2563eb;
  --green:    #059669;  --green-bg:  #d1fae5;
  --orange:   #ea580c;  --orange-bg: #fed7aa;
  --red:      #dc2626;  --red-bg:    #fecaca;
  --blue:     #2563eb;  --blue-bg:   #dbeafe;
  --indigo:   #3730a3;  --indigo-bg: #e0e7ff;
  --page-padding: 2rem;
  --radius:    8px;
  --radius-sm: 5px;
}

* {
  margin: 0;
  padding: 0;
  box-sizing: border-box;
}

body {
  font-family: 'Inter', -apple-system, BlinkMacSystemFont, 'Segoe UI', Roboto, Oxygen, Ubuntu, Cantarell, sans-serif;
  background: #f8fafc;
  color: #1e293b;
  -webkit-font-smoothing: antialiased;
  -moz-osx-font-smoothing: grayscale;
}

.app {
  display: flex;
  flex-direction: column;
  min-height: 100vh;
}

.logo {
  display: flex;
  align-items: baseline;
  gap: 0.75rem;
}

.logo h1 {
  font-size: 1.375rem;
  font-weight: 700;
  color: #0f172a;
  letter-spacing: -0.025em;
}

.subtitle {
  font-size: 0.813rem;
  color: #64748b;
  font-weight: 400;
  padding-left: 0.75rem;
  border-left: 1px solid #e2e8f0;
}

.main-content {
  flex: 1;
  max-width: 1600px;
  width: 100%;
  margin: 0 auto;
  padding: 1.5rem 2rem;
}

.page-header {
  margin-bottom: 1.5rem;
}

.page-header h2 {
  font-size: 1.875rem;
  font-weight: 700;
  color: #0f172a;
  margin-bottom: 0.375rem;
  letter-spacing: -0.025em;
}

.page-header p {
  color: #64748b;
  font-size: 0.938rem;
}

.stats-grid {
  display: grid;
  grid-template-columns: repeat(auto-fit, minmax(280px, 1fr));
  gap: 1.25rem;
  margin-bottom: 1.5rem;
}

.stat-card {
  background: white;
  padding: 1.25rem;
  border-radius: 10px;
  border: 1px solid #e2e8f0;
  transition: all 0.2s ease;
}

.stat-card:hover {
  border-color: #cbd5e1;
  box-shadow: 0 4px 12px rgba(0, 0, 0, 0.06);
}

.stat-label {
  color: #64748b;
  font-size: 0.875rem;
  font-weight: 600;
  text-transform: uppercase;
  letter-spacing: 0.5px;
  margin-bottom: 0.625rem;
}

.stat-value {
  font-size: 2.25rem;
  font-weight: 700;
  color: #0f172a;
  letter-spacing: -0.025em;
}

.stat-card.warning .stat-value {
  color: #ea580c;
}

.stat-card.success .stat-value {
  color: #059669;
}

.stat-card.danger .stat-value {
  color: #dc2626;
}

.stat-card.info .stat-value {
  color: #2563eb;
}

.card {
  background: white;
  border-radius: 10px;
  padding: 1.25rem;
  border: 1px solid #e2e8f0;
  margin-bottom: 1.25rem;
}

.card-header {
  display: flex;
  justify-content: space-between;
  align-items: center;
  margin-bottom: 1rem;
  padding-bottom: 0.875rem;
  border-bottom: 1px solid #e2e8f0;
}

.card-title {
  font-size: 1.125rem;
  font-weight: 700;
  color: #0f172a;
  letter-spacing: -0.025em;
}

.table-container {
  overflow-x: auto;
}

table {
  width: 100%;
  border-collapse: collapse;
}

thead {
  background: #f8fafc;
  border-top: 1px solid #e2e8f0;
  border-bottom: 1px solid #e2e8f0;
}

th {
  text-align: left;
  padding: 0.5rem 0.75rem;
  font-weight: 600;
  color: #475569;
  font-size: 0.75rem;
  text-transform: uppercase;
  letter-spacing: 0.05em;
}

td {
  padding: 0.5rem 0.75rem;
  border-top: 1px solid #f1f5f9;
  color: #334155;
  font-size: 0.875rem;
}

tbody tr {
  transition: background-color 0.15s ease;
}

tbody tr:hover {
  background: #f8fafc;
}

.badge {
  display: inline-block;
  padding: 0.313rem 0.75rem;
  border-radius: 6px;
  font-size: 0.75rem;
  font-weight: 600;
  text-transform: uppercase;
  letter-spacing: 0.025em;
}

.badge.success {
  background: #d1fae5;
  color: #065f46;
}

.badge.warning {
  background: #fed7aa;
  color: #92400e;
}

.badge.danger {
  background: #fecaca;
  color: #991b1b;
}

.badge.info {
  background: #dbeafe;
  color: #1e40af;
}

.badge.increasing {
  background: #d1fae5;
  color: #065f46;
}

.badge.decreasing {
  background: #fecaca;
  color: #991b1b;
}

.badge.stable {
  background: #e0e7ff;
  color: #3730a3;
}

.badge.high {
  background: #fecaca;
  color: #991b1b;
}

.badge.medium {
  background: #fed7aa;
  color: #92400e;
}

.badge.low {
  background: #dbeafe;
  color: #1e40af;
}

.loading {
  text-align: center;
  padding: 3rem;
  color: #64748b;
  font-size: 0.938rem;
}

.error {
  background: #fef2f2;
  border: 1px solid #fecaca;
  color: #991b1b;
  padding: 1rem;
  border-radius: 8px;
  margin: 1rem 0;
  font-size: 0.938rem;
}

/* ── Shell ── */
.app-shell {
  display: flex;
  min-height: 100vh;
  background: var(--content-bg);
}

/* ── Sidebar ── */
.sidebar {
  position: fixed;
  left: 0;
  top: 0;
  width: var(--sidebar-width);
  height: 100vh;
  background: var(--sidebar-bg);
  border-right: 1px solid var(--sidebar-border);
  display: flex;
  flex-direction: column;
  overflow-y: auto;
  z-index: 100;
  transition: width 0.2s ease;
}

.sidebar-logo {
  padding: 1.5rem 1.25rem 1rem;
  border-bottom: 1px solid var(--sidebar-border);
  flex-shrink: 0;
}
.sidebar-logo h1 {
  font-size: 0.9375rem;
  font-weight: 700;
  color: var(--sidebar-text-active);
  letter-spacing: -0.01em;
  margin: 0 0 0.25rem;
  line-height: 1.3;
}
.sidebar-subtitle {
  display: block;
  font-size: 0.6875rem;
  color: var(--sidebar-text);
  letter-spacing: 0.02em;
  text-transform: uppercase;
}

/* Nav links */
.sidebar-nav {
  padding: 0.75rem 0.625rem;
  display: flex;
  flex-direction: column;
  gap: 2px;
  flex: 1;
}
.nav-item {
  display: flex;
  align-items: center;
  gap: 0.625rem;
  padding: 0.5rem 0.75rem;
  border-radius: var(--radius-sm);
  font-size: 0.8125rem;
  font-weight: 500;
  color: var(--sidebar-text);
  text-decoration: none;
  border-left: 3px solid transparent;
  transition: background 0.15s, color 0.15s, border-color 0.15s;
  line-height: 1.4;
}
.nav-item:hover {
  background: var(--sidebar-hover-bg);
  color: var(--sidebar-text-active);
}
.nav-item--active {
  background: var(--sidebar-active-bg);
  color: var(--sidebar-text-active);
  border-left-color: var(--sidebar-active-border);
}
.nav-icon {
  font-size: 0.875rem;
  opacity: 0.65;
  flex-shrink: 0;
  width: 1.125rem;
  text-align: center;
}

/* Sidebar footer */
.sidebar-spacer {
  flex: 1;
  min-height: 1rem;
}
.sidebar-footer {
  padding: 0.75rem 0.625rem 1rem;
  border-top: 1px solid var(--sidebar-border);
  display: flex;
  flex-direction: column;
  gap: 4px;
  flex-shrink: 0;
}

/* ── Content wrapper ── */
.content-wrapper {
  margin-left: var(--sidebar-width);
  flex: 1;
  display: flex;
  flex-direction: column;
  min-height: 100vh;
  min-width: 0;
  transition: margin-left 0.2s ease;
}
.main-content {
  flex: 1;
  padding: 1.5rem 2rem;
  max-width: 1400px;
  width: 100%;
}

/* ── Collapsed sidebar state ── */
.app-shell.is-collapsed .sidebar {
  width: var(--sidebar-collapsed-width);
}
.app-shell.is-collapsed .content-wrapper {
  margin-left: var(--sidebar-collapsed-width);
}

.app-shell.is-collapsed .sidebar-logo-text,
.app-shell.is-collapsed .nav-label {
  display: none;
}

.app-shell.is-collapsed .sidebar-logo {
  justify-content: center;
  padding: 1rem 0.5rem;
}

.app-shell.is-collapsed .sidebar-nav {
  padding: 0.75rem 0.375rem;
}
.app-shell.is-collapsed .nav-item {
  justify-content: center;
  padding: 0.5rem;
  border-left-color: transparent;
  border-radius: var(--radius-sm);
}
.app-shell.is-collapsed .nav-item--active {
  border-left-color: transparent;
  background: var(--sidebar-active-bg);
}
.app-shell.is-collapsed .nav-icon {
  width: auto;
  opacity: 1;
  font-size: 1rem;
}

.app-shell.is-collapsed .sidebar-footer {
  align-items: center;
  padding: 0.75rem 0.375rem 1rem;
}

.app-shell.is-collapsed .nav-item {
  position: relative;
}
.app-shell.is-collapsed .nav-item::after {
  content: attr(data-label);
  position: absolute;
  left: calc(100% + 10px);
  top: 50%;
  transform: translateY(-50%);
  background: #1e293b;
  color: #f1f5f9;
  padding: 4px 10px;
  border-radius: var(--radius-sm);
  font-size: 12px;
  font-weight: 500;
  white-space: nowrap;
  opacity: 0;
  pointer-events: none;
  transition: opacity 0.15s;
  z-index: 200;
  box-shadow: 0 2px 8px rgba(0,0,0,0.25);
}
.app-shell.is-collapsed .nav-item:hover::after {
  opacity: 1;
}

.sidebar-toggle {
  display: flex;
  align-items: center;
  justify-content: center;
  width: 24px;
  height: 24px;
  border: 1px solid var(--sidebar-border);
  border-radius: 4px;
  background: transparent;
  color: var(--sidebar-text);
  cursor: pointer;
  flex-shrink: 0;
  transition: background 0.15s, color 0.15s;
  padding: 0;
}
.sidebar-toggle:hover {
  background: var(--sidebar-hover-bg);
  color: var(--sidebar-text-active);
}
.toggle-icon {
  font-size: 14px;
  font-weight: 600;
  line-height: 1;
  margin-top: -1px;
}

.sidebar-logo {
  display: flex;
  align-items: center;
  justify-content: space-between;
}
.sidebar-logo-text h1 {
  font-size: 0.9375rem;
  font-weight: 700;
  color: var(--sidebar-text-active);
  letter-spacing: -0.01em;
  margin: 0 0 0.25rem;
  line-height: 1.3;
}
</style>
