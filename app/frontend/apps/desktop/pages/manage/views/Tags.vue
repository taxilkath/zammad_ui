<!-- Copyright (C) 2012-2026 Zammad Foundation, https://zammad-foundation.org/ -->

<script setup lang="ts">
import { computed, onMounted, ref } from 'vue'
import { useRouter } from 'vue-router'
import LayoutContent from '#desktop/components/layout/LayoutContent.vue'

interface TagItem {
  id: number
  name: string
  count: number
}

const router = useRouter()
const tags = ref<TagItem[]>([])
const loading = ref(true)
const searchQuery = ref('')

// Setting: tag_new (Allow users to add new tags on the fly)
const allowNewTags = ref(true)
const savingSetting = ref(false)

// Add Tag form
const newTagInput = ref('')
const addingTags = ref(false)

// Rename modal state
const showRenameModal = ref(false)
const selectedTag = ref<TagItem | null>(null)
const renameInput = ref('')
const savingRename = ref(false)

// Delete confirmation modal state
const showDeleteModal = ref(false)
const tagToDelete = ref<TagItem | null>(null)
const deletingTag = ref(false)

// Breadcrumbs
const breadcrumbItems = [
  { label: __('Administration'), route: '/manage' },
  { label: __('Tags') },
]

// CSRF Token Helper
const getCsrfToken = () => {
  return document.querySelector('meta[name="csrf-token"]')?.getAttribute('content') || ''
}

// Fetch all tags from backend
const fetchTags = async () => {
  loading.value = true
  try {
    const res = await fetch('/api/v1/tag_list', {
      headers: {
        Accept: 'application/json',
        'X-Requested-With': 'XMLHttpRequest',
      },
    })
    if (res.ok) {
      const data = await res.json()
      tags.value = Array.isArray(data)
        ? (data as TagItem[]).sort((a: TagItem, b: TagItem) => a.name.localeCompare(b.name))
        : []
    }
  } catch (e) {
    console.error('Failed to fetch tags:', e)
  } finally {
    loading.value = false
  }
}

// Fetch the `tag_new` setting
const fetchSetting = async () => {
  try {
    const res = await fetch('/api/v1/settings/tag_new', {
      headers: {
        Accept: 'application/json',
        'X-Requested-With': 'XMLHttpRequest',
      },
    })
    if (res.ok) {
      const data = await res.json()
      if (typeof data.value === 'boolean') {
        allowNewTags.value = data.value
      } else if (data.state_current && typeof data.state_current.value === 'boolean') {
        allowNewTags.value = data.state_current.value
      }
    }
  } catch (e) {
    console.error('Failed to fetch tag_new setting:', e)
  }
}

// Toggle `tag_new` setting
const toggleAllowNewTags = async () => {
  const nextVal = !allowNewTags.value
  allowNewTags.value = nextVal
  savingSetting.value = true

  try {
    const res = await fetch('/api/v1/settings/tag_new', {
      method: 'PUT',
      headers: {
        'Content-Type': 'application/json',
        Accept: 'application/json',
        'X-Requested-With': 'XMLHttpRequest',
        'X-CSRF-Token': getCsrfToken(),
      },
      body: JSON.stringify({ value: nextVal }),
    })
    if (!res.ok) {
      // Revert on failure
      allowNewTags.value = !nextVal
      alert(__('Failed to update tag setting.'))
    }
  } catch (e) {
    allowNewTags.value = !nextVal
    console.error('Failed to update tag_new setting:', e)
  } finally {
    savingSetting.value = false
  }
}

// Create one or multiple tags (comma-separated)
const handleCreateTags = async () => {
  const rawInput = newTagInput.value.trim()
  if (!rawInput) return

  const names = rawInput
    .split(',')
    .map((s) => s.trim())
    .filter((s) => s.length > 0)

  if (names.length === 0) return

  addingTags.value = true
  try {
    const res = await fetch('/api/v1/tag_list', {
      method: 'POST',
      headers: {
        'Content-Type': 'application/json',
        Accept: 'application/json',
        'X-Requested-With': 'XMLHttpRequest',
        'X-CSRF-Token': getCsrfToken(),
      },
      body: JSON.stringify({ name: names }),
    })

    if (res.ok) {
      newTagInput.value = ''
      await fetchTags()
    } else {
      const data = await res.json()
      alert(data.error_human || data.error || __('Failed to add tags.'))
    }
  } catch (e) {
    console.error('Failed to create tags:', e)
  } finally {
    addingTags.value = false
  }
}

// Open Rename Modal
const openRenameModal = (tag: TagItem) => {
  selectedTag.value = tag
  renameInput.value = tag.name
  showRenameModal.value = true
}

const closeRenameModal = () => {
  showRenameModal.value = false
  selectedTag.value = null
  renameInput.value = ''
}

// Submit Tag Rename
const handleRenameTag = async () => {
  if (!selectedTag.value) return
  const newName = renameInput.value.trim()
  if (!newName) {
    alert(__('Please enter a valid tag name.'))
    return
  }
  if (newName === selectedTag.value.name) {
    closeRenameModal()
    return
  }

  savingRename.value = true
  try {
    const res = await fetch(`/api/v1/tag_list/${selectedTag.value.id}`, {
      method: 'PUT',
      headers: {
        'Content-Type': 'application/json',
        Accept: 'application/json',
        'X-Requested-With': 'XMLHttpRequest',
        'X-CSRF-Token': getCsrfToken(),
      },
      body: JSON.stringify({
        id: selectedTag.value.id,
        name: newName,
      }),
    })

    if (res.ok) {
      closeRenameModal()
      await fetchTags()
    } else {
      const data = await res.json()
      alert(data.error_human || data.error || __('Failed to rename tag.'))
    }
  } catch (e) {
    console.error('Failed to rename tag:', e)
  } finally {
    savingRename.value = false
  }
}

// Open Delete Modal
const openDeleteModal = (tag: TagItem) => {
  tagToDelete.value = tag
  showDeleteModal.value = true
}

const closeDeleteModal = () => {
  showDeleteModal.value = false
  tagToDelete.value = null
}

// Confirm Tag Deletion
const handleDeleteTag = async () => {
  if (!tagToDelete.value) return

  deletingTag.value = true
  try {
    const res = await fetch(`/api/v1/tag_list/${tagToDelete.value.id}`, {
      method: 'DELETE',
      headers: {
        Accept: 'application/json',
        'X-Requested-With': 'XMLHttpRequest',
        'X-CSRF-Token': getCsrfToken(),
      },
    })

    if (res.ok) {
      closeDeleteModal()
      await fetchTags()
    } else {
      const data = await res.json()
      alert(data.error_human || data.error || __('Failed to delete tag.'))
    }
  } catch (e) {
    console.error('Failed to delete tag:', e)
  } finally {
    deletingTag.value = false
  }
}

// Filtered tags for live search
const filteredTags = computed(() => {
  const query = searchQuery.value.trim().toLowerCase()
  if (!query) return tags.value

  return tags.value.filter((tag) => tag.name.toLowerCase().includes(query))
})

onMounted(() => {
  fetchTags()
  fetchSetting()
})
</script>

<template>
  <LayoutContent :breadcrumb-items="breadcrumbItems" background-variant="tertiary" width="full">
    <div class="relative w-full px-8 py-6 text-slate-800 dark:text-slate-100">
      
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
              {{ __('Tags') }} <span class="ml-1 text-sm font-normal text-slate-500 dark:text-slate-400">{{ __('Management') }}</span>
            </h1>
            <p class="text-xs text-slate-500 dark:text-slate-400">
              {{ __('Manage global ticket tags, permissions, and rename or clean up existing tags.') }}
            </p>
          </div>
        </div>
      </div>

      <!-- Setting Card: New Tags Permission -->
      <div class="mb-8 rounded-2xl border border-slate-200 bg-white p-5 shadow-xs dark:border-[#1e293b] dark:bg-[#0f172a]/40">
        <div class="flex flex-col gap-4 sm:flex-row sm:items-center sm:justify-between">
          <div class="flex items-start gap-3.5">
            <div class="mt-0.5 flex h-9 w-9 items-center justify-center rounded-xl bg-blue-50 text-blue-600 dark:bg-blue-950/40 dark:text-blue-400">
              <CommonIcon name="tag" class="h-5 w-5" />
            </div>
            <div>
              <h2 class="text-sm font-semibold text-slate-800 dark:text-slate-100">
                {{ __('Allow Users to Add New Tags') }}
              </h2>
              <p class="mt-0.5 text-xs text-slate-500 dark:text-slate-400">
                {{ __('When enabled, agents can freely create new tags in tickets on the fly. When disabled, only tags configured here can be assigned.') }}
              </p>
            </div>
          </div>
          <div class="flex items-center gap-3">
            <label class="relative inline-flex cursor-pointer items-center">
              <input
                type="checkbox"
                :checked="allowNewTags"
                :disabled="savingSetting"
                class="peer sr-only"
                @change="toggleAllowNewTags"
              />
              <div class="peer h-6 w-11 rounded-full bg-slate-200 after:absolute after:top-[2px] after:left-[2px] after:h-5 after:w-5 after:rounded-full after:border after:border-slate-300 after:bg-white after:transition-all after:content-[''] peer-checked:bg-blue-600 peer-checked:after:translate-x-full peer-checked:after:border-white peer-focus:outline-hidden dark:border-slate-600 dark:bg-slate-700"></div>
              <span class="ml-3 text-xs font-medium text-slate-700 dark:text-slate-300">
                {{ allowNewTags ? __('Enabled') : __('Restricted') }}
              </span>
            </label>
          </div>
        </div>
      </div>

      <!-- Add New Tags Section -->
      <div class="mb-8 rounded-2xl border border-slate-200 bg-white p-5 shadow-xs dark:border-[#1e293b] dark:bg-[#0f172a]/40">
        <h2 class="mb-2 text-sm font-semibold text-slate-800 dark:text-slate-100">
          {{ __('Add Tags') }}
        </h2>
        <p class="mb-4 text-xs text-slate-500 dark:text-slate-400">
          {{ __('Enter one or multiple tag names separated by commas (e.g. bug, high-priority, billing).') }}
        </p>
        <form class="flex flex-col gap-3 sm:flex-row" @submit.prevent="handleCreateTags">
          <div class="relative flex-1">
            <input
              v-model="newTagInput"
              type="text"
              :placeholder="__('Tag names (comma-separated)...')"
              class="w-full rounded-xl border border-slate-300 bg-slate-50 px-4 py-2.5 text-sm text-slate-900 transition-all placeholder:text-slate-400 focus:border-blue-500 focus:bg-white focus:outline-hidden dark:border-slate-700 dark:bg-[#1e293b] dark:text-white dark:placeholder:text-[#475569] dark:focus:bg-[#1e2d45]"
            />
          </div>
          <button
            type="submit"
            :disabled="addingTags || !newTagInput.trim()"
            class="flex cursor-pointer items-center justify-center gap-2 rounded-xl bg-blue-600 px-5 py-2.5 text-sm font-medium text-white shadow-xs transition-colors hover:bg-blue-700 disabled:cursor-not-allowed disabled:opacity-50"
          >
            <div v-if="addingTags" class="h-4 w-4 animate-spin rounded-full border-2 border-white border-t-transparent"></div>
            <CommonIcon v-else name="plus-small" class="h-4 w-4" />
            <span>{{ __('Add Tag(s)') }}</span>
          </button>
        </form>
      </div>

      <!-- Tags Table Section -->
      <div class="rounded-2xl border border-slate-200 bg-white shadow-xs dark:border-[#1e293b] dark:bg-[#0f172a]/40">
        <!-- Table Search & Header Bar -->
        <div class="flex flex-col gap-3 border-b border-slate-200 p-5 sm:flex-row sm:items-center sm:justify-between dark:border-slate-800">
          <div class="flex items-center gap-2">
            <h2 class="text-sm font-semibold text-slate-800 dark:text-slate-100">
              {{ __('Existing Tags') }}
            </h2>
            <span class="rounded-full bg-slate-100 px-2.5 py-0.5 text-xs font-medium text-slate-600 dark:bg-slate-800 dark:text-slate-300">
              {{ tags.length }}
            </span>
          </div>
          
          <div class="relative w-full max-w-xs">
            <input
              v-model="searchQuery"
              type="text"
              :placeholder="__('Search tags...')"
              class="w-full rounded-xl border border-slate-300 bg-slate-100 py-1.5 pr-4 pl-9 text-xs text-slate-900 transition-all placeholder:text-slate-400 focus:border-blue-500 focus:bg-white focus:outline-hidden dark:border-[#2d3f5c] dark:bg-[#1e293b] dark:text-[#94a3b8] dark:placeholder:text-[#475569] dark:focus:bg-[#1e2d45]"
            />
            <div class="absolute top-1/2 left-3 -translate-y-1/2 text-slate-400 dark:text-[#475569]">
              <CommonIcon name="search" class="h-3.5 w-3.5" />
            </div>
          </div>
        </div>

        <!-- Table View -->
        <div class="overflow-x-auto">
          <table class="w-full min-w-[600px] border-collapse text-left">
            <thead>
              <tr class="border-b border-slate-200 bg-slate-50 text-[10px] font-semibold tracking-wider text-slate-400 uppercase dark:border-slate-800 dark:bg-slate-900/40 dark:text-slate-500">
                <th class="px-6 py-3.5">{{ __('Tag Name') }}</th>
                <th class="px-6 py-3.5">{{ __('Ticket Usage Count') }}</th>
                <th class="w-32 px-6 py-3.5 text-right">{{ __('Actions') }}</th>
              </tr>
            </thead>
            <tbody class="divide-y divide-slate-100 dark:divide-slate-800/60">
              <tr v-if="loading">
                <td colspan="3" class="px-6 py-12 text-center text-slate-400 dark:text-slate-500">
                  <div class="flex items-center justify-center gap-2">
                    <div class="h-4 w-4 animate-spin rounded-full border-2 border-blue-500 border-t-transparent"></div>
                    <span>{{ __('Loading tags...') }}</span>
                  </div>
                </td>
              </tr>
              <tr v-else-if="filteredTags.length === 0">
                <td colspan="3" class="px-6 py-12 text-center text-slate-400 dark:text-slate-500">
                  <div class="flex flex-col items-center justify-center">
                    <CommonIcon name="tag" class="mb-2 h-8 w-8 text-slate-300 dark:text-slate-600" />
                    <p class="text-sm font-medium">{{ __('No tags found.') }}</p>
                    <p class="mt-0.5 text-xs text-slate-400">{{ __('Create a new tag using the form above.') }}</p>
                  </div>
                </td>
              </tr>
              <tr
                v-for="tag in filteredTags"
                :key="tag.id"
                class="transition-colors hover:bg-slate-50/80 dark:hover:bg-slate-800/30"
              >
                <!-- Tag Name -->
                <td class="px-6 py-3.5">
                  <div class="flex items-center gap-2">
                    <span class="inline-flex items-center gap-1.5 rounded-lg border border-slate-200 bg-slate-50 px-2.5 py-1 text-xs font-medium text-slate-700 dark:border-slate-700 dark:bg-slate-800 dark:text-slate-200">
                      <span class="text-slate-400 dark:text-slate-500">#</span>
                      <span>{{ tag.name }}</span>
                    </span>
                  </div>
                </td>

                <!-- Count -->
                <td class="px-6 py-3.5">
                  <span
                    class="inline-flex items-center rounded-full px-2.5 py-0.5 text-xs font-semibold"
                    :class="tag.count > 0 ? 'bg-blue-50 text-blue-700 dark:bg-blue-950/40 dark:text-blue-300' : 'bg-slate-100 text-slate-500 dark:bg-slate-800 dark:text-slate-400'"
                  >
                    {{ tag.count }} {{ tag.count === 1 ? __('ticket') : __('tickets') }}
                  </span>
                </td>

                <!-- Actions -->
                <td class="px-6 py-3.5 text-right">
                  <div class="flex items-center justify-end gap-1.5">
                    <button
                      type="button"
                      class="flex h-8 w-8 cursor-pointer items-center justify-center rounded-lg text-slate-400 transition-colors hover:bg-slate-100 hover:text-slate-700 dark:hover:bg-slate-800 dark:hover:text-slate-200"
                      :title="__('Rename Tag')"
                      @click="openRenameModal(tag)"
                    >
                      <CommonIcon name="pencil" class="h-3.5 w-3.5" />
                    </button>
                    <button
                      type="button"
                      class="flex h-8 w-8 cursor-pointer items-center justify-center rounded-lg text-slate-400 transition-colors hover:bg-red-50 hover:text-red-600 dark:hover:bg-red-950/30 dark:hover:text-red-400"
                      :title="__('Delete Tag')"
                      @click="openDeleteModal(tag)"
                    >
                      <CommonIcon name="trash" class="h-3.5 w-3.5" />
                    </button>
                  </div>
                </td>
              </tr>
            </tbody>
          </table>
        </div>
      </div>

    </div>
  </LayoutContent>

  <!-- Rename Tag Modal -->
  <Teleport to="body">
    <div
      v-if="showRenameModal"
      class="fixed inset-0 z-50 flex items-center justify-center bg-slate-900/50 p-4 backdrop-blur-xs"
      @click="closeRenameModal"
    >
      <div
        class="w-full max-w-md overflow-hidden rounded-2xl border border-slate-200 bg-white shadow-2xl dark:border-slate-800 dark:bg-[#0f172a]"
        @click.stop
      >
        <div class="flex items-center justify-between border-b border-slate-200 p-5 dark:border-slate-800">
          <h3 class="text-base font-bold text-slate-900 dark:text-white">
            {{ __('Rename Tag') }}
          </h3>
          <button
            type="button"
            class="cursor-pointer text-slate-400 hover:text-slate-600 dark:hover:text-slate-200"
            @click="closeRenameModal"
          >
            <CommonIcon name="x-mark" class="h-5 w-5" />
          </button>
        </div>

        <form @submit.prevent="handleRenameTag">
          <div class="p-5">
            <p class="mb-3 text-xs text-slate-500 dark:text-slate-400">
              {{ __('Renaming this tag will update it across all tickets where it is currently assigned.') }}
            </p>
            <label class="mb-1.5 block text-xs font-medium text-slate-700 dark:text-slate-300">
              {{ __('Tag Name') }}
            </label>
            <input
              v-model="renameInput"
              type="text"
              required
              class="w-full rounded-xl border border-slate-300 bg-white px-3.5 py-2.5 text-sm text-slate-900 focus:border-blue-500 focus:outline-hidden dark:border-slate-700 dark:bg-slate-900 dark:text-white"
            />
          </div>

          <div class="flex items-center justify-end gap-2.5 border-t border-slate-200 bg-slate-50/60 p-4 dark:border-slate-800 dark:bg-slate-900/50">
            <button
              type="button"
              class="cursor-pointer rounded-lg border border-slate-300 px-4 py-2 text-xs font-medium text-slate-700 transition-colors hover:bg-slate-100 dark:border-slate-700 dark:text-slate-300 dark:hover:bg-slate-800"
              @click="closeRenameModal"
            >
              {{ __('Cancel') }}
            </button>
            <button
              type="submit"
              :disabled="savingRename || !renameInput.trim()"
              class="flex cursor-pointer items-center gap-1.5 rounded-lg bg-blue-600 px-4 py-2 text-xs font-medium text-white shadow-xs transition-colors hover:bg-blue-700 disabled:opacity-50"
            >
              <div v-if="savingRename" class="h-3.5 w-3.5 animate-spin rounded-full border-2 border-white border-t-transparent"></div>
              <span>{{ __('Save') }}</span>
            </button>
          </div>
        </form>
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
            {{ __('Delete Tag') }}
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
            {{ __('Are you sure you want to delete the tag "%s"?').replace('%s', tagToDelete?.name || '') }}
          </p>
          <p class="mt-2 text-xs text-slate-500 dark:text-slate-400">
            {{ __('This will permanently remove this tag from all tickets where it is currently used. This action cannot be undone.') }}
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
            :disabled="deletingTag"
            class="flex cursor-pointer items-center gap-1.5 rounded-lg bg-red-600 px-4 py-2 text-xs font-medium text-white shadow-xs transition-colors hover:bg-red-700 disabled:opacity-50"
            @click="handleDeleteTag"
          >
            <div v-if="deletingTag" class="h-3.5 w-3.5 animate-spin rounded-full border-2 border-white border-t-transparent"></div>
            <span>{{ __('Delete Tag') }}</span>
          </button>
        </div>
      </div>
    </div>
  </Teleport>
</template>
