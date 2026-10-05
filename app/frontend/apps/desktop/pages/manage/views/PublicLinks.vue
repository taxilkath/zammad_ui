<!-- Copyright (C) 2012-2026 Zammad Foundation, https://zammad-foundation.org/ -->

<script setup lang="ts">
import { computed, onMounted, ref } from 'vue'
import { useRouter } from 'vue-router'
import LayoutContent from '#desktop/components/layout/LayoutContent.vue'

interface PublicLinkItem {
  id: number
  link: string
  title: string
  description?: string
  screen: string[]
  new_tab: boolean
  prio: number
  updated_at?: string
  created_at?: string
}

const router = useRouter()
const links = ref<PublicLinkItem[]>([])
const loading = ref(true)
const searchQuery = ref('')
const activeActionMenuLinkId = ref<number | null>(null)

// Drawer state
const showDrawer = ref(false)
const drawerMode = ref<'create' | 'edit'>('create')
const drawerLinkId = ref<number | null>(null)
const submitting = ref(false)

// Delete confirmation modal state
const showDeleteModal = ref(false)
const linkToDelete = ref<PublicLinkItem | null>(null)
const deletingLink = ref(false)

// Reordering state
const reordering = ref(false)

// Form state
const form = ref({
  title: '',
  link: '',
  description: '',
  screen: ['login', 'signup', 'password_reset'] as string[],
  new_tab: true,
})

// Screen context options
const screenOptions = [
  { value: 'login', label: __('Login Screen'), desc: __('Shown in footer of the login page') },
  { value: 'signup', label: __('Signup Screen'), desc: __('Shown in footer of user registration') },
  { value: 'password_reset', label: __('Forgot Password Screen'), desc: __('Shown in password reset page') },
]

// Breadcrumbs
const breadcrumbItems = [
  { label: __('Administration'), route: '/manage' },
  { label: __('Public Links') },
]

// CSRF Token Helper
const getCsrfToken = () => {
  return document.querySelector('meta[name="csrf-token"]')?.getAttribute('content') || ''
}

// Fetch all public links
const fetchLinks = async () => {
  loading.value = true
  try {
    const res = await fetch('/api/v1/public_links', {
      headers: {
        Accept: 'application/json',
        'X-Requested-With': 'XMLHttpRequest',
      },
    })
    if (res.ok) {
      const data = await res.json()
      links.value = Array.isArray(data)
        ? (data as PublicLinkItem[]).sort((a, b) => (a.prio ?? 0) - (b.prio ?? 0))
        : []
    }
  } catch (e) {
    console.error('Failed to fetch public links:', e)
  } finally {
    loading.value = false
  }
}

// Toggle action menu dropdown
const toggleActionMenu = (id: number, event: Event) => {
  event.stopPropagation()
  if (activeActionMenuLinkId.value === id) {
    activeActionMenuLinkId.value = null
  } else {
    activeActionMenuLinkId.value = id
  }
}

const closeActionMenu = () => {
  activeActionMenuLinkId.value = null
}

// Filtered links for live search
const filteredLinks = computed(() => {
  const query = searchQuery.value.trim().toLowerCase()
  if (!query) return links.value

  return links.value.filter((l) => {
    return (
      l.title.toLowerCase().includes(query) ||
      l.link.toLowerCase().includes(query) ||
      (l.description && l.description.toLowerCase().includes(query))
    )
  })
})

// Toggle screen context checkbox
const toggleScreenOption = (value: string) => {
  if (form.value.screen.includes(value)) {
    form.value.screen = form.value.screen.filter((s) => s !== value)
  } else {
    form.value.screen.push(value)
  }
}

// Reordering: Move up or down
const movePriority = async (index: number, direction: 'up' | 'down') => {
  const targetIndex = direction === 'up' ? index - 1 : index + 1
  if (targetIndex < 0 || targetIndex >= links.value.length) return

  // Swap in local array
  const currentList = [...links.value]
  const temp = currentList[index]
  currentList[index] = currentList[targetIndex]
  currentList[targetIndex] = temp

  links.value = currentList

  // Prepare payload for /api/v1/public_links_prio
  const prios: [number, number][] = currentList.map((item, idx) => [item.id, idx + 1])

  reordering.value = true
  try {
    await fetch('/api/v1/public_links_prio', {
      method: 'POST',
      headers: {
        'Content-Type': 'application/json',
        Accept: 'application/json',
        'X-Requested-With': 'XMLHttpRequest',
        'X-CSRF-Token': getCsrfToken(),
      },
      body: JSON.stringify({ prios }),
    })
  } catch (e) {
    console.error('Failed to update public link priorities:', e)
    // Refresh to restore backend truth
    await fetchLinks()
  } finally {
    reordering.value = false
  }
}

// Open drawer for new link
const handleNewLink = () => {
  drawerMode.value = 'create'
  drawerLinkId.value = null
  form.value = {
    title: '',
    link: '',
    description: '',
    screen: ['login', 'signup', 'password_reset'],
    new_tab: true,
  }
  showDrawer.value = true
}

// Open drawer to clone link
const handleCloneLink = (linkItem: PublicLinkItem) => {
  activeActionMenuLinkId.value = null
  drawerMode.value = 'create'
  drawerLinkId.value = null
  form.value = {
    title: `${__('Clone')}: ${linkItem.title}`,
    link: linkItem.link,
    description: linkItem.description || '',
    screen: Array.isArray(linkItem.screen) ? [...linkItem.screen] : ['login'],
    new_tab: linkItem.new_tab ?? true,
  }
  showDrawer.value = true
}

// Open drawer to edit link
const handleEditLink = (linkItem: PublicLinkItem) => {
  activeActionMenuLinkId.value = null
  drawerMode.value = 'edit'
  drawerLinkId.value = linkItem.id
  form.value = {
    title: linkItem.title,
    link: linkItem.link,
    description: linkItem.description || '',
    screen: Array.isArray(linkItem.screen) ? [...linkItem.screen] : ['login'],
    new_tab: linkItem.new_tab ?? true,
  }
  showDrawer.value = true
}

// Close drawer
const closeDrawer = () => {
  showDrawer.value = false
}

// Save Public Link (Create or Update)
const savePublicLink = async () => {
  if (!form.value.title.trim()) {
    alert(__('Please enter a title for the link.'))
    return
  }
  if (!form.value.link.trim()) {
    alert(__('Please enter a valid target URL.'))
    return
  }
  if (form.value.screen.length === 0) {
    alert(__('Please select at least one screen context.'))
    return
  }

  submitting.value = true
  try {
    const payload = {
      title: form.value.title.trim(),
      link: form.value.link.trim(),
      description: form.value.description.trim(),
      screen: form.value.screen,
      new_tab: form.value.new_tab,
    }

    const isEdit = drawerMode.value === 'edit'
    const url = isEdit ? `/api/v1/public_links/${drawerLinkId.value}` : '/api/v1/public_links'
    const method = isEdit ? 'PUT' : 'POST'

    const res = await fetch(url, {
      method,
      headers: {
        'Content-Type': 'application/json',
        Accept: 'application/json',
        'X-Requested-With': 'XMLHttpRequest',
        'X-CSRF-Token': getCsrfToken(),
      },
      body: JSON.stringify(payload),
    })

    if (res.ok) {
      showDrawer.value = false
      await fetchLinks()
    } else {
      const data = await res.json()
      alert(data.error_human || data.error || __('Failed to save public link.'))
    }
  } catch (e) {
    console.error('Failed to save public link:', e)
  } finally {
    submitting.value = false
  }
}

// Open Delete Modal
const openDeleteModal = (linkItem: PublicLinkItem) => {
  activeActionMenuLinkId.value = null
  linkToDelete.value = linkItem
  showDeleteModal.value = true
}

const closeDeleteModal = () => {
  showDeleteModal.value = false
  linkToDelete.value = null
}

// Confirm Delete
const handleDeleteLink = async () => {
  if (!linkToDelete.value) return

  deletingLink.value = true
  try {
    const res = await fetch(`/api/v1/public_links/${linkToDelete.value.id}`, {
      method: 'DELETE',
      headers: {
        Accept: 'application/json',
        'X-Requested-With': 'XMLHttpRequest',
        'X-CSRF-Token': getCsrfToken(),
      },
    })

    if (res.ok) {
      closeDeleteModal()
      await fetchLinks()
    } else {
      const data = await res.json()
      alert(data.error_human || data.error || __('Failed to delete public link.'))
    }
  } catch (e) {
    console.error('Failed to delete public link:', e)
  } finally {
    deletingLink.value = false
  }
}

// Format screen badges
const getScreenLabel = (val: string) => {
  const match = screenOptions.find((o) => o.value === val)
  return match ? match.label : val
}

onMounted(() => {
  fetchLinks()
  window.addEventListener('click', closeActionMenu)
})
</script>

<template>
  <LayoutContent :breadcrumb-items="breadcrumbItems" background-variant="tertiary" width="full">
    <div class="relative w-full px-8 py-6 text-slate-800 dark:text-slate-100" @click="closeActionMenu">
      
      <!-- Top header area -->
      <div class="mb-8 flex items-center justify-between">
        <div class="flex items-center gap-3">
          <button
            type="button"
            class="flex h-8 w-8 cursor-pointer items-center justify-center rounded-full border border-slate-300 text-slate-600 transition-colors hover:bg-slate-100 dark:border-slate-600 dark:text-slate-400 dark:hover:bg-slate-800"
            :title="__('Back')"
            @click="router.push('/manage')"
          >
            <CommonIcon name="arrow-left" class="h-4 w-4" />
          </button>
          <div>
            <h1 class="text-2xl font-bold text-slate-800 dark:text-slate-100">
              {{ __('Public Links') }} <span class="ml-1 text-sm font-normal text-slate-500 dark:text-slate-400">{{ __('Management') }}</span>
            </h1>
            <p class="text-xs text-slate-500 dark:text-slate-400">
              {{ __('Define external links shown in login, signup, and password reset screen footers (e.g. privacy policies, terms of service).') }}
            </p>
          </div>
        </div>
        <div>
          <button
            type="button"
            class="cursor-pointer rounded-lg bg-[#22c55e] px-4 py-2 text-sm font-medium text-white shadow-xs transition-colors hover:bg-[#16a34a]"
            @click="handleNewLink"
          >
            {{ __('New Public Link') }}
          </button>
        </div>
      </div>

      <!-- Search Box & Reordering Tip -->
      <div class="mb-6 flex flex-col gap-4 sm:flex-row sm:items-center sm:justify-between">
        <div class="relative w-full max-w-md">
          <input
            v-model="searchQuery"
            type="text"
            :placeholder="__('Search public links...')"
            class="w-full rounded-xl border border-slate-300 bg-slate-100 py-2 pr-4 pl-10 text-sm text-slate-900 transition-all placeholder:text-slate-400 focus:border-blue-500 focus:bg-white focus:outline-hidden dark:border-[#2d3f5c] dark:bg-[#1e293b] dark:text-[#94a3b8] dark:placeholder:text-[#475569] dark:focus:bg-[#1e2d45]"
          />
          <div class="absolute top-1/2 left-3.5 -translate-y-1/2 text-slate-400 dark:text-[#475569]">
            <CommonIcon name="search" class="h-4 w-4" />
          </div>
        </div>

        <div class="flex items-center gap-2 text-xs text-slate-500 dark:text-slate-400">
          <CommonIcon name="info-circle" class="h-3.5 w-3.5" />
          <span>{{ __('Use the arrows to reorder display priority.') }}</span>
        </div>
      </div>

      <!-- Links Table Section -->
      <div class="overflow-x-auto rounded-2xl border border-slate-200 bg-white shadow-xs dark:border-[#1e293b] dark:bg-[#0f172a]/40">
        <table class="w-full min-w-[750px] border-collapse text-left">
          <thead>
            <tr class="border-b border-slate-200 bg-slate-50 text-[10px] font-semibold tracking-wider text-slate-400 uppercase dark:border-slate-800 dark:bg-slate-900/40 dark:text-slate-500">
              <th class="w-16 px-4 py-4 text-center first:rounded-tl-2xl">{{ __('Order') }}</th>
              <th class="px-6 py-4">{{ __('Title') }}</th>
              <th class="px-6 py-4">{{ __('URL / Target Link') }}</th>
              <th class="px-6 py-4">{{ __('Context Screens') }}</th>
              <th class="px-6 py-4 text-center">{{ __('New Tab') }}</th>
              <th class="w-16 px-6 py-4 text-right last:rounded-tr-2xl"></th>
            </tr>
          </thead>
          <tbody class="divide-y divide-slate-100 dark:divide-slate-800/60">
            <tr v-if="loading">
              <td colspan="6" class="px-6 py-12 text-center text-slate-400 dark:text-slate-500">
                <div class="flex items-center justify-center gap-2">
                  <div class="h-4 w-4 animate-spin rounded-full border-2 border-blue-500 border-t-transparent"></div>
                  <span>{{ __('Loading public links...') }}</span>
                </div>
              </td>
            </tr>
            <tr v-else-if="filteredLinks.length === 0">
              <td colspan="6" class="px-6 py-12 text-center text-slate-400 dark:text-slate-500">
                <div class="flex flex-col items-center justify-center">
                  <CommonIcon name="link" class="mb-2 h-8 w-8 text-slate-300 dark:text-slate-600" />
                  <p class="text-sm font-medium">{{ __('No public links found.') }}</p>
                  <p class="mt-0.5 text-xs text-slate-400">{{ __('Create a link to display on login or signup footers.') }}</p>
                </div>
              </td>
            </tr>
            <tr
              v-for="(linkItem, index) in filteredLinks"
              :key="linkItem.id"
              class="transition-colors hover:bg-slate-50/80 dark:hover:bg-slate-800/30"
            >
              <!-- Order Buttons -->
              <td class="px-4 py-4 text-center">
                <div class="flex items-center justify-center gap-1">
                  <button
                    type="button"
                    :disabled="index === 0 || reordering"
                    class="rounded-sm p-1 text-slate-400 hover:bg-slate-100 hover:text-slate-700 disabled:opacity-20 dark:hover:bg-slate-800 dark:hover:text-slate-200"
                    :title="__('Move Up')"
                    @click="movePriority(index, 'up')"
                  >
                    <CommonIcon name="arrow-up-short" class="h-4 w-4" />
                  </button>
                  <button
                    type="button"
                    :disabled="index === filteredLinks.length - 1 || reordering"
                    class="rounded-sm p-1 text-slate-400 hover:bg-slate-100 hover:text-slate-700 disabled:opacity-20 dark:hover:bg-slate-800 dark:hover:text-slate-200"
                    :title="__('Move Down')"
                    @click="movePriority(index, 'down')"
                  >
                    <CommonIcon name="arrow-down-short" class="h-4 w-4" />
                  </button>
                </div>
              </td>

              <!-- Title -->
              <td class="px-6 py-4">
                <div class="flex flex-col">
                  <span class="font-medium text-slate-800 dark:text-slate-200">{{ linkItem.title }}</span>
                  <span v-if="linkItem.description" class="text-xs text-slate-400">{{ linkItem.description }}</span>
                </div>
              </td>

              <!-- Link URL -->
              <td class="max-w-xs truncate px-6 py-4 text-xs font-mono text-slate-600 dark:text-slate-400">
                <a
                  :href="linkItem.link"
                  target="_blank"
                  rel="noopener noreferrer"
                  class="inline-flex items-center gap-1 text-blue-600 hover:underline dark:text-blue-400"
                >
                  <span class="truncate">{{ linkItem.link }}</span>
                  <CommonIcon name="box-arrow-up-right" class="h-3 w-3 shrink-0" />
                </a>
              </td>

              <!-- Screens Badges -->
              <td class="px-6 py-4">
                <div class="flex flex-wrap gap-1.5">
                  <span
                    v-for="scr in linkItem.screen"
                    :key="scr"
                    class="inline-flex items-center rounded-md bg-slate-100 px-2 py-0.5 text-[11px] font-medium text-slate-700 dark:bg-slate-800 dark:text-slate-300"
                  >
                    {{ getScreenLabel(scr) }}
                  </span>
                </div>
              </td>

              <!-- New Tab -->
              <td class="px-6 py-4 text-center">
                <span
                  class="inline-flex items-center gap-1.5 px-2.5 py-1 rounded-full text-xs font-medium shadow-2xs"
                  :class="linkItem.new_tab ? 'bg-emerald-50 text-emerald-700 border border-emerald-200 dark:bg-emerald-950/40 dark:text-emerald-300 dark:border-emerald-800' : 'bg-slate-100 text-slate-500 border border-slate-200 dark:bg-slate-800 dark:text-slate-400 dark:border-slate-700'"
                >
                  <CommonIcon
                    :name="linkItem.new_tab ? 'check2' : 'dash'"
                    class="h-3.5 w-3.5"
                    :class="linkItem.new_tab ? 'text-emerald-600 dark:text-emerald-400' : 'text-slate-400 dark:text-slate-500'"
                  />
                  <span>{{ linkItem.new_tab ? __('Yes') : __('No') }}</span>
                </span>
              </td>

              <!-- Action Menu -->
              <td class="relative px-6 py-4 text-right">
                <button
                  type="button"
                  class="cursor-pointer rounded-lg p-1 text-slate-400 transition-colors hover:bg-slate-100 hover:text-slate-600 dark:hover:bg-slate-800 dark:hover:text-slate-200"
                  @click="toggleActionMenu(linkItem.id, $event)"
                >
                  <CommonIcon name="three-dots-vertical" class="h-4 w-4" />
                </button>

                <!-- Actions Dropdown -->
                <div
                  v-if="activeActionMenuLinkId === linkItem.id"
                  class="absolute right-6 z-20 mt-1 w-44 overflow-hidden rounded-xl border border-slate-200 bg-white text-left shadow-xl dark:border-slate-700 dark:bg-slate-800"
                  @click.stop
                >
                  <div class="py-1.5">
                    <button
                      type="button"
                      class="flex w-full items-center px-4 py-2.5 text-xs text-slate-700 transition-colors hover:bg-slate-50 dark:text-slate-200 dark:hover:bg-slate-700/50"
                      @click="handleEditLink(linkItem)"
                    >
                      <CommonIcon name="pencil" class="mr-2.5 h-3.5 w-3.5 text-slate-400" />
                      {{ __('Edit') }}
                    </button>
                    <button
                      type="button"
                      class="flex w-full items-center px-4 py-2.5 text-xs text-slate-700 transition-colors hover:bg-slate-50 dark:text-slate-200 dark:hover:bg-slate-700/50"
                      @click="handleCloneLink(linkItem)"
                    >
                      <CommonIcon name="copy" class="mr-2.5 h-3.5 w-3.5 text-slate-400" />
                      {{ __('Clone') }}
                    </button>
                    <div class="my-1 border-t border-slate-100 dark:border-slate-700"></div>
                    <button
                      type="button"
                      class="flex w-full items-center px-4 py-2.5 text-xs text-red-600 transition-colors hover:bg-red-50 dark:hover:bg-red-950/20"
                      @click="openDeleteModal(linkItem)"
                    >
                      <CommonIcon name="trash" class="mr-2.5 h-3.5 w-3.5 text-red-400" />
                      {{ __('Delete') }}
                    </button>
                  </div>
                </div>
              </td>
            </tr>
          </tbody>
        </table>
      </div>

    </div>
  </LayoutContent>

  <!-- Slide-over Drawer for Create / Edit Link -->
  <Teleport to="body">
    <div
      v-if="showDrawer"
      class="fixed inset-0 z-50 flex justify-end bg-slate-900/50 backdrop-blur-xs transition-opacity"
      @click="closeDrawer"
    >
      <div
        class="flex h-full w-full max-w-lg flex-col justify-between border-l border-slate-200 bg-white shadow-2xl dark:border-slate-800 dark:bg-[#0f172a]"
        @click.stop
      >
        <!-- Drawer Header -->
        <div class="flex items-center justify-between border-b border-slate-200 p-6 dark:border-slate-800">
          <div>
            <h3 class="text-lg font-bold text-slate-900 dark:text-white">
              {{ drawerMode === 'create' ? __('New Public Link') : __('Edit Public Link') }}
            </h3>
            <p class="mt-0.5 text-xs text-slate-500 dark:text-slate-400">
              {{ __('Configure public navigation links shown to users before logging in.') }}
            </p>
          </div>
          <button
            type="button"
            class="cursor-pointer text-slate-400 hover:text-slate-600 dark:hover:text-slate-200"
            @click="closeDrawer"
          >
            <CommonIcon name="x-mark" class="h-5 w-5" />
          </button>
        </div>

        <!-- Drawer Form Body -->
        <div class="flex-1 space-y-5 overflow-y-auto p-6">
          <!-- Title -->
          <div>
            <label class="mb-1.5 block text-xs font-medium text-slate-700 dark:text-slate-300">
              {{ __('Title') }} <span class="text-red-500">*</span>
            </label>
            <input
              v-model="form.title"
              type="text"
              maxlength="200"
              required
              :placeholder="__('e.g. Terms & Conditions')"
              class="w-full rounded-xl border border-slate-300 bg-white px-3.5 py-2.5 text-sm text-slate-900 focus:border-blue-500 focus:outline-hidden dark:border-slate-700 dark:bg-slate-900 dark:text-white"
            />
          </div>

          <!-- URL Link -->
          <div>
            <label class="mb-1.5 block text-xs font-medium text-slate-700 dark:text-slate-300">
              {{ __('URL / Link') }} <span class="text-red-500">*</span>
            </label>
            <input
              v-model="form.link"
              type="url"
              maxlength="500"
              required
              :placeholder="__('https://example.com/terms')"
              class="w-full rounded-xl border border-slate-300 bg-white px-3.5 py-2.5 text-sm text-slate-900 focus:border-blue-500 focus:outline-hidden dark:border-slate-700 dark:bg-slate-900 dark:text-white font-mono text-xs"
            />
          </div>

          <!-- Description (Screen Reader title) -->
          <div>
            <label class="mb-1.5 block text-xs font-medium text-slate-700 dark:text-slate-300">
              {{ __('Description (Optional)') }}
            </label>
            <input
              v-model="form.description"
              type="text"
              maxlength="200"
              :placeholder="__('Shown as tooltip and accessible label for screen readers')"
              class="w-full rounded-xl border border-slate-300 bg-white px-3.5 py-2.5 text-sm text-slate-900 focus:border-blue-500 focus:outline-hidden dark:border-slate-700 dark:bg-slate-900 dark:text-white"
            />
          </div>

          <!-- Context / Screens -->
          <div>
            <label class="mb-1.5 block text-xs font-medium text-slate-700 dark:text-slate-300">
              {{ __('Context Screens') }} <span class="text-red-500">*</span>
            </label>
            <p class="mb-2 text-xs text-slate-500 dark:text-slate-400">
              {{ __('Select on which public application screens this link should appear.') }}
            </p>
            <div class="space-y-2 rounded-xl border border-slate-200 bg-slate-50/60 p-3 dark:border-slate-800 dark:bg-slate-900/40">
              <label
                v-for="opt in screenOptions"
                :key="opt.value"
                class="flex cursor-pointer items-start gap-3 rounded-lg p-2 transition-colors hover:bg-slate-100/60 dark:hover:bg-slate-800/40"
              >
                <input
                  type="checkbox"
                  :checked="form.screen.includes(opt.value)"
                  class="mt-0.5 h-4 w-4 rounded-sm border-slate-300 text-blue-600 focus:ring-blue-500"
                  @change="toggleScreenOption(opt.value)"
                />
                <div>
                  <span class="block text-xs font-medium text-slate-800 dark:text-slate-200">{{ opt.label }}</span>
                  <span class="text-[11px] text-slate-500 dark:text-slate-400">{{ opt.desc }}</span>
                </div>
              </label>
            </div>
          </div>

          <!-- Open in New Tab -->
          <div class="rounded-xl border border-slate-200 bg-slate-50/60 p-4 dark:border-slate-800 dark:bg-slate-900/40">
            <label class="flex cursor-pointer items-start gap-3">
              <input
                v-model="form.new_tab"
                type="checkbox"
                class="mt-1 h-4 w-4 rounded-sm border-slate-300 text-blue-600 focus:ring-blue-500"
              />
              <div>
                <span class="text-sm font-medium text-slate-800 dark:text-slate-200">{{ __('Open in New Tab') }}</span>
                <p class="mt-0.5 text-xs text-slate-500 dark:text-slate-400">
                  {{ __('Opens the target URL in a new browser tab with target="_blank".') }}
                </p>
              </div>
            </label>
          </div>
        </div>

        <!-- Drawer Footer -->
        <div class="flex items-center justify-end gap-3 border-t border-slate-200 bg-slate-50/50 p-6 dark:border-slate-800 dark:bg-slate-900/50">
          <button
            type="button"
            class="cursor-pointer rounded-lg border border-slate-300 px-4 py-2 text-sm font-medium text-slate-700 transition-colors hover:bg-slate-100 dark:border-slate-700 dark:text-slate-300 dark:hover:bg-slate-800"
            @click="closeDrawer"
          >
            {{ __('Cancel') }}
          </button>
          <button
            type="button"
            :disabled="submitting || !form.title.trim() || !form.link.trim() || form.screen.length === 0"
            class="flex cursor-pointer items-center gap-2 rounded-lg bg-blue-600 px-4 py-2 text-sm font-medium text-white shadow-xs transition-colors hover:bg-blue-700 disabled:opacity-50"
            @click="savePublicLink"
          >
            <div v-if="submitting" class="h-4 w-4 animate-spin rounded-full border-2 border-white border-t-transparent"></div>
            {{ drawerMode === 'create' ? __('Create Link') : __('Save Changes') }}
          </button>
        </div>

      </div>
    </div>
  </Teleport>

  <!-- Delete Confirmation Modal -->
  <Teleport to="body">
    <div
      v-if="showDeleteModal"
      class="fixed inset-0 z-50 flex items-center justify-center bg-slate-900/50 p-4 backdrop-blur-xs"
      @click="closeDeleteModal"
    >
      <div
        class="w-full max-w-md overflow-hidden rounded-2xl border border-slate-200 bg-white shadow-2xl dark:border-slate-800 dark:bg-[#0f172a]"
        @click.stop
      >
        <div class="flex items-center justify-between border-b border-slate-200 p-5 dark:border-slate-800">
          <h3 class="text-base font-bold text-red-600 dark:text-red-400">
            {{ __('Delete Public Link') }}
          </h3>
          <button
            type="button"
            class="cursor-pointer text-slate-400 hover:text-slate-600 dark:hover:text-slate-200"
            @click="closeDeleteModal"
          >
            <CommonIcon name="x-mark" class="h-5 w-5" />
          </button>
        </div>

        <div class="p-5">
          <p class="text-sm text-slate-700 dark:text-slate-300">
            {{ __('Are you sure you want to delete the public link "%s"?').replace('%s', linkToDelete?.title || '') }}
          </p>
          <p class="mt-2 text-xs text-slate-500 dark:text-slate-400">
            {{ __('This link will no longer appear on public screens.') }}
          </p>
        </div>

        <div class="flex items-center justify-end gap-2.5 border-t border-slate-200 bg-slate-50/60 p-4 dark:border-slate-800 dark:bg-slate-900/50">
          <button
            type="button"
            class="cursor-pointer rounded-lg border border-slate-300 px-4 py-2 text-xs font-medium text-slate-700 transition-colors hover:bg-slate-100 dark:border-slate-700 dark:text-slate-300 dark:hover:bg-slate-800"
            @click="closeDeleteModal"
          >
            {{ __('Cancel') }}
          </button>
          <button
            type="button"
            :disabled="deletingLink"
            class="flex cursor-pointer items-center gap-1.5 rounded-lg bg-red-600 px-4 py-2 text-xs font-medium text-white shadow-xs transition-colors hover:bg-red-700 disabled:opacity-50"
            @click="handleDeleteLink"
          >
            <div v-if="deletingLink" class="h-3.5 w-3.5 animate-spin rounded-full border-2 border-white border-t-transparent"></div>
            <span>{{ __('Delete Link') }}</span>
          </button>
        </div>
      </div>
    </div>
  </Teleport>
</template>
