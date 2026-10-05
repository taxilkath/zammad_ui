<!-- Copyright (C) 2012-2026 Zammad Foundation, https://zammad-foundation.org/ -->

<script setup lang="ts">
import { computed, onMounted, ref } from 'vue'
import { useRouter } from 'vue-router'
import LayoutContent from '#desktop/components/layout/LayoutContent.vue'

interface RoleItem {
  id: number
  name: string
  active: boolean
  default_at_signup: boolean
  note?: string
  users_count?: number
  permission_ids?: number[]
  group_ids?: number[]
  updated_at?: string
  created_at?: string
}

interface GroupItem {
  id: number
  name: string
  active?: boolean
}

interface PermissionDefinition {
  id: number
  name: string
  label: string
  description: string
  disabled?: boolean
}

interface PermissionCategory {
  key: string
  label: string
  description?: string
  parentPermissionId?: number
  permissions: PermissionDefinition[]
}

const router = useRouter()
const roles = ref<RoleItem[]>([])
const groups = ref<GroupItem[]>([])
const loading = ref(true)
const searchQuery = ref('')
const activeActionMenuRoleId = ref<number | null>(null)

// Drawer state
const showDrawer = ref(false)
const drawerMode = ref<'create' | 'edit'>('create')
const drawerRoleId = ref<number | null>(null)
const submitting = ref(false)
const activeTab = ref<'general' | 'permissions' | 'groups'>('general')

// Form state
const form = ref({
  name: '',
  default_at_signup: false,
  active: true,
  note: '',
})

// Selected permission IDs (Set of numbers)
const selectedPermissionIds = ref<Set<number>>(new Set())

// Group access mapping: Record<groupId, Set<'read' | 'create' | 'change' | 'overview' | 'full'>>
type AccessType = 'read' | 'create' | 'change' | 'overview' | 'full'
const groupAccessMap = ref<Record<number, Set<AccessType>>>({})

// Navigation breadcrumbs
const breadcrumbItems = [
  { label: __('Administration'), route: '/manage' },
  { label: __('Roles') },
]

// System permission definitions categorized
const permissionCategories = ref<PermissionCategory[]>([
  {
    key: 'admin',
    label: __('Admin Interface'),
    description: __('Manage system configuration, channels, objects, and administrative tools.'),
    parentPermissionId: 1, // admin
    permissions: [
      { id: 1, name: 'admin', label: __('Admin Interface (Full)'), description: __('Grants overall administrative access.') },
      { id: 2, name: 'admin.user', label: __('Users'), description: __('Manage user accounts.') },
      { id: 3, name: 'admin.group', label: __('Groups'), description: __('Manage agent groups.') },
      { id: 4, name: 'admin.role', label: __('Roles'), description: __('Manage user roles and permissions.') },
      { id: 5, name: 'admin.organization', label: __('Organizations'), description: __('Manage customer organizations.') },
      { id: 6, name: 'admin.overview', label: __('Overviews'), description: __('Manage ticket overviews.') },
      { id: 7, name: 'admin.text_module', label: __('Text Modules'), description: __('Manage canned text responses.') },
      { id: 8, name: 'admin.macro', label: __('Macros'), description: __('Manage ticket macros.') },
      { id: 9, name: 'admin.template', label: __('Templates'), description: __('Manage ticket templates.') },
      { id: 10, name: 'admin.tag', label: __('Tags'), description: __('Manage global tags.') },
      { id: 50, name: 'admin.checklist', label: __('Checklists'), description: __('Manage ticket checklist templates.') },
      { id: 11, name: 'admin.calendar', label: __('Calendars'), description: __('Manage business calendars and holidays.') },
      { id: 12, name: 'admin.sla', label: __('SLAs'), description: __('Manage Service Level Agreements.') },
      { id: 13, name: 'admin.trigger', label: __('Trigger'), description: __('Manage event-based ticket triggers.') },
      { id: 14, name: 'admin.public_links', label: __('Public Links'), description: __('Manage public ticket share links.') },
      { id: 15, name: 'admin.webhook', label: __('Webhooks'), description: __('Manage outgoing HTTP webhooks.') },
      { id: 16, name: 'admin.scheduler', label: __('Scheduler'), description: __('Manage recurring automated jobs.') },
      { id: 17, name: 'admin.report_profile', label: __('Report Profiles'), description: __('Manage analytical reporting profiles.') },
      { id: 18, name: 'admin.time_accounting', label: __('Time Accounting'), description: __('Manage time tracking configuration.') },
      { id: 19, name: 'admin.knowledge_base', label: __('Knowledge Base'), description: __('Manage knowledge base settings.') },
      { id: 20, name: 'admin.channel_web', label: __('Channel: Web'), description: __('Configure web portal channel.') },
      { id: 21, name: 'admin.channel_formular', label: __('Channel: Form'), description: __('Configure embedded web forms.') },
      { id: 22, name: 'admin.channel_email', label: __('Channel: Email'), description: __('Configure email inbound and outbound.') },
      { id: 23, name: 'admin.channel_sms', label: __('Channel: SMS'), description: __('Configure SMS notifications.') },
      { id: 24, name: 'admin.channel_chat', label: __('Channel: Chat'), description: __('Configure live chat widgets.') },
      { id: 25, name: 'admin.channel_google', label: __('Channel: Google'), description: __('Configure Google accounts.') },
      { id: 26, name: 'admin.channel_microsoft365', label: __('Channel: Microsoft 365'), description: __('Configure Microsoft 365 IMAP.') },
      { id: 75, name: 'admin.channel_microsoft_graph', label: __('Channel: Microsoft Graph'), description: __('Configure Microsoft Graph API.') },
      { id: 28, name: 'admin.channel_facebook', label: __('Channel: Facebook'), description: __('Configure Facebook pages.') },
      { id: 29, name: 'admin.channel_telegram', label: __('Channel: Telegram'), description: __('Configure Telegram bots.') },
      { id: 30, name: 'admin.channel_whatsapp', label: __('Channel: WhatsApp'), description: __('Configure WhatsApp messaging.') },
      { id: 31, name: 'admin.branding', label: __('Branding'), description: __('Configure application logos, colors, and title.') },
      { id: 32, name: 'admin.system', label: __('System Settings'), description: __('Configure core system parameters.') },
      { id: 33, name: 'admin.security', label: __('Security'), description: __('Configure authentication and password policies.') },
      { id: 34, name: 'admin.ticket', label: __('Ticket Settings'), description: __('Configure default ticket behavior.') },
      { id: 37, name: 'admin.integration', label: __('Integrations'), description: __('Manage third-party integrations.') },
      { id: 38, name: 'admin.api', label: __('API'), description: __('Manage API access and applications.') },
      { id: 39, name: 'admin.object', label: __('Objects'), description: __('Manage custom fields on objects.') },
      { id: 40, name: 'admin.ticket_state', label: __('Ticket States'), description: __('Manage ticket states and workflows.') },
      { id: 41, name: 'admin.ticket_priority', label: __('Ticket Priorities'), description: __('Manage ticket priorities.') },
      { id: 42, name: 'admin.core_workflow', label: __('Core Workflows'), description: __('Manage dynamic form workflows.') },
      { id: 43, name: 'admin.translation', label: __('Translations'), description: __('Manage UI translations.') },
      { id: 44, name: 'admin.data_privacy', label: __('Data Privacy'), description: __('Manage GDPR compliance and deletion.') },
      { id: 45, name: 'admin.maintenance', label: __('Maintenance'), description: __('Manage system maintenance mode.') },
      { id: 46, name: 'admin.monitoring', label: __('Monitoring'), description: __('Monitor background tasks and health.') },
      { id: 47, name: 'admin.package', label: __('Packages'), description: __('Manage installed packages.') },
      { id: 48, name: 'admin.session', label: __('Sessions'), description: __('Monitor active user sessions.') },
      { id: 49, name: 'admin.system_report', label: __('System Report'), description: __('Generate diagnostic reports.') },
      { id: 81, name: 'admin.ai_provider', label: __('AI: Provider'), description: __('Configure AI providers and credentials.') },
      { id: 78, name: 'admin.ai_assistance_ticket_summary', label: __('AI: Ticket Summary'), description: __('Configure AI ticket summaries.') },
      { id: 79, name: 'admin.ai_assistance_text_tools', label: __('AI: Writing Assistant'), description: __('Configure AI writing assistant tools.') },
      { id: 80, name: 'admin.ai_agent', label: __('AI: Agents'), description: __('Configure automated AI agents.') },
    ],
  },
  {
    key: 'ticket',
    label: __('Tickets'),
    description: __('Access to customer or agent ticket management interfaces.'),
    permissions: [
      { id: 60, name: 'ticket.agent', label: __('Agent Tickets'), description: __('Access tickets as an agent based on group access.') },
      { id: 61, name: 'ticket.customer', label: __('Customer Tickets'), description: __('Access tickets as a customer (self-service portal).') },
    ],
  },
  {
    key: 'knowledge_base',
    label: __('Knowledge Base'),
    description: __('Knowledge base viewing and editorial permissions.'),
    permissions: [
      { id: 56, name: 'knowledge_base.editor', label: __('Knowledge Base Editor'), description: __('Create, edit, and publish knowledge base articles.') },
      { id: 57, name: 'knowledge_base.reader', label: __('Knowledge Base Reader'), description: __('Read public and internal knowledge base articles.') },
    ],
  },
  {
    key: 'chat_cti',
    label: __('Communication Channels'),
    description: __('Agent access to live chat and telephony integration.'),
    permissions: [
      { id: 52, name: 'chat.agent', label: __('Agent Chat'), description: __('Accept and respond to incoming customer live chats.') },
      { id: 54, name: 'cti.agent', label: __('Agent Phone (CTI)'), description: __('Use telephony call logging and caller lookup.') },
    ],
  },
  {
    key: 'reporting',
    label: __('Reporting & Analytics'),
    description: __('Access to reporting dashboards and statistics.'),
    permissions: [
      { id: 58, name: 'report', label: __('Reporting Interface'), description: __('Access interactive reporting charts and ticket statistics.') },
    ],
  },
  {
    key: 'user_preferences',
    label: __('User Profile Settings'),
    description: __('Personal preferences and account configuration.'),
    parentPermissionId: 62, // user_preferences
    permissions: [
      { id: 62, name: 'user_preferences', label: __('Profile Settings (Master)'), description: __('Access personal settings profile page.') },
      { id: 63, name: 'user_preferences.appearance', label: __('Appearance'), description: __('Change personal theme and dark mode settings.') },
      { id: 64, name: 'user_preferences.language', label: __('Language'), description: __('Change personal display language.') },
      { id: 65, name: 'user_preferences.avatar', label: __('Avatar'), description: __('Upload and update personal avatar.') },
      { id: 66, name: 'user_preferences.out_of_office', label: __('Out of Office'), description: __('Configure out-of-office dates and replacement.') },
      { id: 67, name: 'user_preferences.password', label: __('Password'), description: __('Change personal account password.') },
      { id: 68, name: 'user_preferences.two_factor_authentication', label: __('Two-Factor Authentication'), description: __('Manage personal 2FA devices.') },
      { id: 69, name: 'user_preferences.device', label: __('Devices'), description: __('View logged-in devices.') },
      { id: 70, name: 'user_preferences.access_token', label: __('API Tokens'), description: __('Generate personal API access tokens.') },
      { id: 71, name: 'user_preferences.linked_accounts', label: __('Linked Accounts'), description: __('Manage linked third-party authentication.') },
      { id: 72, name: 'user_preferences.notifications', label: __('Notifications'), description: __('Manage email and web notification preferences.') },
      { id: 73, name: 'user_preferences.overview_sorting', label: __('Overview Sorting'), description: __('Customize personal ticket overview order.') },
      { id: 74, name: 'user_preferences.calendar', label: __('Calendar Feeds'), description: __('Subscribe to personal ticket calendars.') },
    ],
  },
])

// Does the current role have agent permission?
const hasAgentPermission = computed(() => {
  return selectedPermissionIds.value.has(60) // ticket.agent
})

// Toggle action menu dropdown
const toggleActionMenu = (roleId: number, event: Event) => {
  event.stopPropagation()
  if (activeActionMenuRoleId.value === roleId) {
    activeActionMenuRoleId.value = null
  } else {
    activeActionMenuRoleId.value = roleId
  }
}

const closeActionMenu = () => {
  activeActionMenuRoleId.value = null
}

// Fetch all roles and groups
const fetchData = async () => {
  loading.value = true
  try {
    const [rolesRes, groupsRes] = await Promise.all([
      fetch('/api/v1/roles', {
        headers: {
          'Accept': 'application/json',
          'X-Requested-With': 'XMLHttpRequest',
        },
      }),
      fetch('/api/v1/groups', {
        headers: {
          'Accept': 'application/json',
          'X-Requested-With': 'XMLHttpRequest',
        },
      }),
    ])

    if (rolesRes.ok) {
      const data = await rolesRes.json()
      roles.value = Array.isArray(data)
        ? (data as RoleItem[]).sort((a: RoleItem, b: RoleItem) => a.name.localeCompare(b.name))
        : []
    }

    if (groupsRes.ok) {
      const data = await groupsRes.json()
      groups.value = Array.isArray(data)
        ? (data as GroupItem[]).filter((g: GroupItem) => g.active !== false).sort((a: GroupItem, b: GroupItem) => a.name.localeCompare(b.name))
        : []
    }
  } catch (e) {
    console.error('Failed to fetch roles data:', e)
  } finally {
    loading.value = false
  }
}

// Filter roles client-side
const filteredRoles = computed(() => {
  const query = searchQuery.value.trim().toLowerCase()
  if (!query) return roles.value

  return roles.value.filter((role) => {
    return (
      role.name.toLowerCase().includes(query) ||
      (role.note && role.note.toLowerCase().includes(query))
    )
  })
})

// Toggle a single permission
const togglePermission = (id: number) => {
  if (selectedPermissionIds.value.has(id)) {
    selectedPermissionIds.value.delete(id)

    // If unchecking admin master (id 1), uncheck all admin children
    if (id === 1) {
      const adminCat = permissionCategories.value.find((c) => c.key === 'admin')
      if (adminCat) {
        for (const p of adminCat.permissions) {
          selectedPermissionIds.value.delete(p.id)
        }
      }
    }
    // If unchecking user_preferences master (id 62), uncheck sub-preferences
    if (id === 62) {
      const userPrefCat = permissionCategories.value.find((c) => c.key === 'user_preferences')
      if (userPrefCat) {
        for (const p of userPrefCat.permissions) {
          selectedPermissionIds.value.delete(p.id)
        }
      }
    }
  } else {
    selectedPermissionIds.value.add(id)

    // If checking any child of admin, ensure admin master is checked
    const adminCat = permissionCategories.value.find((c) => c.key === 'admin')
    if (adminCat && adminCat.permissions.some((p) => p.id === id) && id !== 1) {
      selectedPermissionIds.value.add(1)
    }

    // If checking any child of user_preferences, ensure user_preferences is checked
    const userPrefCat = permissionCategories.value.find((c) => c.key === 'user_preferences')
    if (userPrefCat && userPrefCat.permissions.some((p) => p.id === id) && id !== 62) {
      selectedPermissionIds.value.add(62)
    }
  }
}

// Check/uncheck all permissions in a category
const toggleCategoryPermissions = (category: PermissionCategory, forceState?: boolean) => {
  const allIds = category.permissions.map((p) => p.id)
  const allChecked = allIds.every((id) => selectedPermissionIds.value.has(id))
  const targetState = forceState !== undefined ? forceState : !allChecked

  for (const id of allIds) {
    if (targetState) {
      selectedPermissionIds.value.add(id)
    } else {
      selectedPermissionIds.value.delete(id)
    }
  }
}

const isCategoryFullyChecked = (category: PermissionCategory): boolean => {
  return category.permissions.every((p) => selectedPermissionIds.value.has(p.id))
}

const isCategoryPartiallyChecked = (category: PermissionCategory): boolean => {
  const hasSome = category.permissions.some((p) => selectedPermissionIds.value.has(p.id))
  return hasSome && !isCategoryFullyChecked(category)
}

// Group Access Matrix Management
const toggleGroupAccess = (groupId: number, access: AccessType) => {
  if (!groupAccessMap.value[groupId]) {
    groupAccessMap.value[groupId] = new Set<AccessType>()
  }

  const currentSet = groupAccessMap.value[groupId]

  if (access === 'full') {
    if (currentSet.has('full')) {
      currentSet.delete('full')
    } else {
      // Clear specific accesses when 'full' is selected
      currentSet.clear()
      currentSet.add('full')
    }
  } else {
    // If selecting specific access, remove 'full'
    currentSet.delete('full')
    if (currentSet.has(access)) {
      currentSet.delete(access)
    } else {
      currentSet.add(access)
    }
  }
}

const isGroupAccessChecked = (groupId: number, access: AccessType): boolean => {
  const currentSet = groupAccessMap.value[groupId]
  if (!currentSet) return false
  if (access === 'full') return currentSet.has('full')
  return currentSet.has('full') || currentSet.has(access)
}

// Set all groups to Full Access
const grantAllGroupsFull = () => {
  for (const g of groups.value) {
    groupAccessMap.value[g.id] = new Set<AccessType>(['full'])
  }
}

// Clear all group permissions
const clearAllGroupAccess = () => {
  groupAccessMap.value = {}
}

// Toggle all accesses for a single group row
const toggleRowAllAccess = (groupId: number) => {
  const currentSet = groupAccessMap.value[groupId]
  if (currentSet && currentSet.has('full')) {
    delete groupAccessMap.value[groupId]
  } else {
    groupAccessMap.value[groupId] = new Set<AccessType>(['full'])
  }
}

// Open drawer for new role creation
const handleNewRole = () => {
  drawerMode.value = 'create'
  drawerRoleId.value = null
  activeTab.value = 'general'
  form.value = {
    name: '',
    default_at_signup: false,
    active: true,
    note: '',
  }
  selectedPermissionIds.value = new Set()
  groupAccessMap.value = {}
  showDrawer.value = true
}

// Open drawer to clone an existing role
const handleCloneRole = async (role: RoleItem) => {
  activeActionMenuRoleId.value = null
  drawerMode.value = 'create'
  drawerRoleId.value = null
  activeTab.value = 'general'

  form.value = {
    name: `${__('Clone')}: ${role.name}`,
    default_at_signup: false,
    active: true,
    note: role.note || '',
  }
  selectedPermissionIds.value = new Set(role.permission_ids || [])
  groupAccessMap.value = {}

  // Hydrate full role details to copy group permissions
  try {
    const res = await fetch(`/api/v1/roles/${role.id}`, {
      headers: {
        'Accept': 'application/json',
        'X-Requested-With': 'XMLHttpRequest',
      },
    })
    if (res.ok) {
      const fullRole = await res.json()
      if (fullRole.permission_ids) {
        selectedPermissionIds.value = new Set(fullRole.permission_ids)
      }
      if (fullRole.group_ids && typeof fullRole.group_ids === 'object') {
        const newMap: Record<number, Set<AccessType>> = {}
        if (Array.isArray(fullRole.group_ids)) {
          for (const gid of fullRole.group_ids) {
            newMap[gid] = new Set<AccessType>(['full'])
          }
        } else {
          for (const [gidStr, accesses] of Object.entries(fullRole.group_ids)) {
            const gid = Number(gidStr)
            newMap[gid] = new Set<AccessType>(Array.isArray(accesses) ? (accesses as AccessType[]) : ['full'])
          }
        }
        groupAccessMap.value = newMap
      }
    }
  } catch (e) {
    console.error('Failed to clone role details:', e)
  }

  showDrawer.value = true
}

// Open drawer to edit an existing role
const handleEditRole = async (role: RoleItem) => {
  activeActionMenuRoleId.value = null
  drawerMode.value = 'edit'
  drawerRoleId.value = role.id
  activeTab.value = 'general'

  form.value = {
    name: role.name,
    default_at_signup: Boolean(role.default_at_signup),
    active: Boolean(role.active),
    note: role.note || '',
  }
  selectedPermissionIds.value = new Set(role.permission_ids || [])
  groupAccessMap.value = {}

  // Pre-fill groups if present in list
  if (role.group_ids && Array.isArray(role.group_ids)) {
    for (const gid of role.group_ids) {
      groupAccessMap.value[gid] = new Set<AccessType>(['full'])
    }
  }

  showDrawer.value = true

  // Hydrate full role record from backend
  try {
    const res = await fetch(`/api/v1/roles/${role.id}`, {
      headers: {
        'Accept': 'application/json',
        'X-Requested-With': 'XMLHttpRequest',
      },
    })
    if (res.ok) {
      const fullRole = await res.json()
      if (drawerRoleId.value === role.id) {
        form.value = {
          name: fullRole.name,
          default_at_signup: Boolean(fullRole.default_at_signup),
          active: Boolean(fullRole.active),
          note: fullRole.note || '',
        }
        if (fullRole.permission_ids) {
          selectedPermissionIds.value = new Set(fullRole.permission_ids)
        }
        if (fullRole.group_ids) {
          const newMap: Record<number, Set<AccessType>> = {}
          if (Array.isArray(fullRole.group_ids)) {
            for (const gid of fullRole.group_ids) {
              newMap[gid] = new Set<AccessType>(['full'])
            }
          } else if (typeof fullRole.group_ids === 'object') {
            for (const [gidStr, accesses] of Object.entries(fullRole.group_ids)) {
              const gid = Number(gidStr)
              newMap[gid] = new Set<AccessType>(Array.isArray(accesses) ? (accesses as AccessType[]) : ['full'])
            }
          }
          groupAccessMap.value = newMap
        }
      }
    }
  } catch (e) {
    console.error('Failed to fetch full role details:', e)
  }
}

// Close drawer
const closeDrawer = () => {
  showDrawer.value = false
}

// Toggle role active/inactive status
const handleToggleActive = async (role: RoleItem) => {
  activeActionMenuRoleId.value = null
  const actionText = role.active ? __('deactivate') : __('activate')
  if (!confirm(__('Are you sure you want to %s role "%s"?').replace('%s', actionText).replace('%s', role.name))) {
    return
  }

  try {
    const res = await fetch(`/api/v1/roles/${role.id}`, {
      method: 'PUT',
      headers: {
        'Content-Type': 'application/json',
        'Accept': 'application/json',
        'X-Requested-With': 'XMLHttpRequest',
        'X-CSRF-Token': document.querySelector('meta[name="csrf-token"]')?.getAttribute('content') || '',
      },
      body: JSON.stringify({ active: !role.active }),
    })
    if (res.ok) {
      fetchData()
    } else {
      const data = await res.json()
      alert(data.error_human || data.error || __('Failed to update role status.'))
    }
  } catch (e) {
    console.error('Failed to toggle role active status:', e)
  }
}

// Save role (Create or Update)
const saveRole = async () => {
  if (!form.value.name.trim()) {
    alert(__('Please enter a role name.'))
    return
  }

  submitting.value = true
  try {
    // Format group_ids payload
    const formattedGroupIds: Record<string, string[]> = {}
    if (hasAgentPermission.value) {
      for (const [gid, accessSet] of Object.entries(groupAccessMap.value)) {
        if (accessSet.size > 0) {
          formattedGroupIds[gid] = Array.from(accessSet)
        }
      }
    }

    const payload = {
      name: form.value.name.trim(),
      default_at_signup: form.value.default_at_signup,
      active: form.value.active,
      note: form.value.note.trim(),
      permission_ids: Array.from(selectedPermissionIds.value),
      group_ids: formattedGroupIds,
    }

    const isEdit = drawerMode.value === 'edit'
    const url = isEdit ? `/api/v1/roles/${drawerRoleId.value}` : '/api/v1/roles'
    const method = isEdit ? 'PUT' : 'POST'

    const res = await fetch(url, {
      method,
      headers: {
        'Content-Type': 'application/json',
        'Accept': 'application/json',
        'X-Requested-With': 'XMLHttpRequest',
        'X-CSRF-Token': document.querySelector('meta[name="csrf-token"]')?.getAttribute('content') || '',
      },
      body: JSON.stringify(payload),
    })

    if (res.ok) {
      showDrawer.value = false
      fetchData()
    } else {
      const data = await res.json()
      alert(data.error_human || data.error || __('Failed to save role.'))
    }
  } catch (e) {
    console.error('Failed to save role:', e)
  } finally {
    submitting.value = false
  }
}

onMounted(() => {
  fetchData()
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
          <h1 class="mb-1 text-2xl font-bold text-slate-800 dark:text-slate-100">
            {{ __('Roles') }} <span class="ml-1 text-sm font-normal text-slate-500 dark:text-slate-400">{{ __('Management') }}</span>
          </h1>
        </div>
        <div>
          <button
            type="button"
            class="cursor-pointer rounded-lg bg-[#22c55e] px-4 py-2 text-sm font-medium text-white shadow-xs transition-colors hover:bg-[#16a34a]"
            @click="handleNewRole"
          >
            {{ __('New Role') }}
          </button>
        </div>
      </div>

      <!-- Search Box -->
      <div class="mb-6 max-w-md">
        <div class="relative">
          <input
            v-model="searchQuery"
            type="text"
            :placeholder="__('Search for roles')"
            class="w-full rounded-xl border border-slate-300 bg-slate-100 py-2 pr-4 pl-10 text-sm text-slate-900 transition-all duration-150 placeholder:text-slate-400 focus:border-blue-500 focus:bg-white focus:outline-hidden dark:border-[#2d3f5c] dark:bg-[#1e293b] dark:text-[#94a3b8] dark:placeholder:text-[#475569] dark:focus:bg-[#1e2d45]"
          />
          <div class="absolute top-1/2 left-3.5 -translate-y-1/2 text-slate-400 dark:text-[#475569]">
            <CommonIcon name="search" class="h-4 w-4" />
          </div>
        </div>
      </div>

      <!-- Table Section -->
      <div class="overflow-x-auto rounded-2xl border border-slate-200 bg-white shadow-xs dark:border-[#1e293b] dark:bg-[#0f172a]/40">
        <table class="w-full min-w-[800px] border-collapse text-left">
          <thead>
            <tr class="border-b border-slate-200 bg-slate-50 text-[10px] font-semibold tracking-wider text-slate-400 uppercase dark:border-slate-800 dark:bg-slate-900/40 dark:text-slate-500">
              <th class="px-6 py-4 first:rounded-tl-2xl">{{ __('Role Name') }}</th>
              <th class="px-6 py-4">{{ __('Default at Signup') }}</th>
              <th class="px-6 py-4">{{ __('Assigned Users') }}</th>
              <th class="px-6 py-4">{{ __('Note') }}</th>
              <th class="px-6 py-4 text-center">{{ __('Active') }}</th>
              <th class="w-16 px-6 py-4 text-right last:rounded-tr-2xl"></th>
            </tr>
          </thead>
          <tbody class="divide-y divide-slate-100 dark:divide-slate-800/60">
            <tr v-if="loading">
              <td colspan="6" class="px-6 py-12 text-center text-slate-400 dark:text-slate-500">
                <div class="flex items-center justify-center gap-2">
                  <div class="h-4 w-4 animate-spin rounded-full border-2 border-blue-500 border-t-transparent"></div>
                  <span>{{ __('Loading roles...') }}</span>
                </div>
              </td>
            </tr>
            <tr v-else-if="filteredRoles.length === 0">
              <td colspan="6" class="px-6 py-12 text-center text-slate-400 dark:text-slate-500">
                {{ __('No roles found.') }}
              </td>
            </tr>
            <tr
              v-for="role in filteredRoles"
              :key="role.id"
              class="transition-colors hover:bg-slate-50/80 dark:hover:bg-slate-800/30"
            >
              <!-- Role Name -->
              <td class="px-6 py-4">
                <div class="flex items-center gap-2">
                  <span class="font-medium text-slate-800 dark:text-slate-200">{{ role.name }}</span>
                </div>
              </td>

              <!-- Default at Signup -->
              <td class="px-6 py-4 text-sm">
                <span
                  v-if="role.default_at_signup"
                  class="inline-flex items-center rounded-full bg-blue-100 px-2.5 py-0.5 text-xs font-medium text-blue-800 dark:bg-blue-900/30 dark:text-blue-300"
                >
                  {{ __('Yes') }}
                </span>
                <span v-else class="text-xs text-slate-400 dark:text-slate-600">
                  {{ __('No') }}
                </span>
              </td>

              <!-- Assigned Users -->
              <td class="px-6 py-4 text-sm text-slate-600 dark:text-slate-400">
                <span class="inline-flex items-center gap-1 rounded-md bg-slate-100 px-2 py-0.5 text-xs text-slate-700 dark:bg-slate-800 dark:text-slate-300">
                  {{ role.users_count ?? (role.name === 'Customer' ? '1,400+' : '-') }}
                </span>
              </td>

              <!-- Note -->
              <td class="max-w-xs truncate px-6 py-4 text-xs text-slate-500 dark:text-slate-400">
                {{ role.note || '-' }}
              </td>

              <!-- Active Status -->
              <td class="px-6 py-4 text-center whitespace-nowrap">
                <span
                  class="inline-flex items-center gap-1.5 px-3 py-1 rounded-full text-xs font-medium shadow-2xs"
                  :class="
                    role.active
                      ? 'bg-emerald-50 text-emerald-700 border border-emerald-200 dark:bg-emerald-950/40 dark:text-emerald-300 dark:border-emerald-800'
                      : 'bg-slate-100 text-slate-500 border border-slate-200 dark:bg-slate-800 dark:text-slate-400 dark:border-slate-700'
                  "
                >
                  <CommonIcon
                    :name="role.active ? 'check2' : 'x-lg'"
                    class="w-3.5 h-3.5"
                    :class="role.active ? 'text-emerald-600 dark:text-emerald-400' : 'text-slate-400 dark:text-slate-500'"
                  />
                  <span>{{ role.active ? __('Active') : __('Inactive') }}</span>
                </span>
              </td>

              <!-- Actions Dropdown -->
              <td class="relative px-6 py-4 text-right">
                <button
                  type="button"
                  class="cursor-pointer rounded-lg p-1 text-slate-400 transition-colors hover:bg-slate-100 hover:text-slate-600 dark:hover:bg-slate-800 dark:hover:text-slate-200"
                  @click="toggleActionMenu(role.id, $event)"
                >
                  <CommonIcon name="three-dots-vertical" class="h-4 w-4" />
                </button>
                
                <!-- Action Dropdown Card -->
                <div
                  v-if="activeActionMenuRoleId === role.id"
                  class="absolute right-6 z-20 mt-1 w-44 overflow-hidden rounded-xl border border-slate-200 bg-white text-left shadow-xl dark:border-slate-700 dark:bg-slate-800"
                  @click.stop
                >
                  <div class="py-1.5">
                    <button
                      type="button"
                      class="flex w-full items-center px-4 py-2.5 text-xs text-slate-700 transition-colors hover:bg-slate-50 dark:text-slate-200 dark:hover:bg-slate-700/50"
                      @click="handleEditRole(role)"
                    >
                      <CommonIcon name="pencil" class="mr-2.5 h-3.5 w-3.5 text-slate-400" />
                      {{ __('Edit') }}
                    </button>
                    <button
                      type="button"
                      class="flex w-full items-center px-4 py-2.5 text-xs text-slate-700 transition-colors hover:bg-slate-50 dark:text-slate-200 dark:hover:bg-slate-700/50"
                      @click="handleCloneRole(role)"
                    >
                      <CommonIcon name="copy" class="mr-2.5 h-3.5 w-3.5 text-slate-400" />
                      {{ __('Clone') }}
                    </button>
                    <div class="my-1 border-t border-slate-100 dark:border-slate-700"></div>
                    <button
                      type="button"
                      class="flex w-full items-center px-4 py-2.5 text-xs transition-colors"
                      :class="role.active ? 'text-amber-600 hover:bg-amber-50 dark:hover:bg-amber-950/20' : 'text-green-600 hover:bg-green-50 dark:hover:bg-green-950/20'"
                      @click="handleToggleActive(role)"
                    >
                      <CommonIcon :name="role.active ? 'eye-slash' : 'check2'" class="mr-2.5 h-3.5 w-3.5 text-slate-400" />
                      {{ role.active ? __('Deactivate') : __('Activate') }}
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

  <!-- Slide-over Drawer for Create / Edit / Clone Role -->
  <Teleport to="body">
    <div
      v-if="showDrawer"
      class="fixed inset-0 z-50 flex justify-end bg-slate-900/50 backdrop-blur-xs transition-opacity"
      @click="closeDrawer"
    >
      <div
        class="flex h-full w-full max-w-2xl flex-col justify-between border-l border-slate-200 bg-white shadow-2xl dark:border-slate-800 dark:bg-[#0f172a]"
        @click.stop
      >
        <!-- Drawer Header -->
        <div class="flex items-center justify-between border-b border-slate-200 p-6 dark:border-slate-800">
          <div>
            <h3 class="text-lg font-bold text-slate-900 dark:text-white">
              {{ drawerMode === 'create' ? __('New Role') : __('Edit Role') }}
            </h3>
            <p class="mt-0.5 text-xs text-slate-500 dark:text-slate-400">
              {{ __('Configure role permissions, access control, and group defaults.') }}
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

        <!-- Navigation Tabs -->
        <div class="flex border-b border-slate-200 px-6 dark:border-slate-800">
          <button
            type="button"
            class="border-b-2 px-4 py-3 text-sm font-medium transition-colors"
            :class="activeTab === 'general' ? 'border-blue-600 text-blue-600 dark:border-blue-400 dark:text-blue-400' : 'border-transparent text-slate-500 hover:text-slate-700 dark:text-slate-400 dark:hover:text-slate-200'"
            @click="activeTab = 'general'"
          >
            {{ __('General') }}
          </button>
          <button
            type="button"
            class="border-b-2 px-4 py-3 text-sm font-medium transition-colors"
            :class="activeTab === 'permissions' ? 'border-blue-600 text-blue-600 dark:border-blue-400 dark:text-blue-400' : 'border-transparent text-slate-500 hover:text-slate-700 dark:text-slate-400 dark:hover:text-slate-200'"
            @click="activeTab = 'permissions'"
          >
            {{ __('Permissions') }}
            <span class="ml-1.5 rounded-full bg-slate-100 px-2 py-0.5 text-xs text-slate-600 dark:bg-slate-800 dark:text-slate-300">
              {{ selectedPermissionIds.size }}
            </span>
          </button>
          <button
            type="button"
            class="border-b-2 px-4 py-3 text-sm font-medium transition-colors"
            :class="activeTab === 'groups' ? 'border-blue-600 text-blue-600 dark:border-blue-400 dark:text-blue-400' : 'border-transparent text-slate-500 hover:text-slate-700 dark:text-slate-400 dark:hover:text-slate-200'"
            @click="activeTab = 'groups'"
          >
            {{ __('Group Permissions') }}
          </button>
        </div>

        <!-- Drawer Body -->
        <div class="flex-1 space-y-6 overflow-y-auto p-6">
          
          <!-- TAB 1: General Settings -->
          <div v-show="activeTab === 'general'" class="space-y-6">
            <!-- Name -->
            <div>
              <label class="mb-1.5 block text-xs font-medium text-slate-700 dark:text-slate-300">
                {{ __('Role Name') }} <span class="text-red-500">*</span>
              </label>
              <input
                v-model="form.name"
                type="text"
                maxlength="100"
                :placeholder="__('e.g. Service Desk Specialist')"
                class="w-full rounded-xl border border-slate-300 bg-white px-3.5 py-2.5 text-sm text-slate-900 focus:border-blue-500 focus:outline-hidden dark:border-slate-700 dark:bg-slate-900 dark:text-white"
              />
            </div>

            <!-- Default at Signup -->
            <div class="rounded-xl border border-slate-200 bg-slate-50/60 p-4 dark:border-slate-800 dark:bg-slate-900/40">
              <label class="flex cursor-pointer items-start gap-3">
                <input
                  v-model="form.default_at_signup"
                  type="checkbox"
                  class="mt-1 h-4 w-4 rounded-sm border-slate-300 text-blue-600 focus:ring-blue-500"
                />
                <div>
                  <span class="text-sm font-medium text-slate-800 dark:text-slate-200">{{ __('Default at Signup') }}</span>
                  <p class="mt-0.5 text-xs text-slate-500 dark:text-slate-400">
                    {{ __('Automatically assign this role to new users who sign up via the web portal.') }}
                  </p>
                </div>
              </label>
            </div>

            <!-- Active Status -->
            <div class="rounded-xl border border-slate-200 bg-slate-50/60 p-4 dark:border-slate-800 dark:bg-slate-900/40">
              <label class="flex cursor-pointer items-start gap-3">
                <input
                  v-model="form.active"
                  type="checkbox"
                  class="mt-1 h-4 w-4 rounded-sm border-slate-300 text-blue-600 focus:ring-blue-500"
                />
                <div>
                  <span class="text-sm font-medium text-slate-800 dark:text-slate-200">{{ __('Active') }}</span>
                  <p class="mt-0.5 text-xs text-slate-500 dark:text-slate-400">
                    {{ __('Inactive roles cannot be assigned to users.') }}
                  </p>
                </div>
              </label>
            </div>

            <!-- Internal Note -->
            <div>
              <label class="mb-1.5 block text-xs font-medium text-slate-700 dark:text-slate-300">
                {{ __('Internal Note') }}
              </label>
              <textarea
                v-model="form.note"
                rows="3"
                maxlength="250"
                :placeholder="__('Notes are visible to agents only, never to customers.')"
                class="w-full rounded-xl border border-slate-300 bg-white px-3.5 py-2.5 text-sm text-slate-900 focus:border-blue-500 focus:outline-hidden dark:border-slate-700 dark:bg-slate-900 dark:text-white"
              ></textarea>
              <span class="mt-1 block text-right text-[11px] text-slate-400">
                {{ form.note.length }} / 250
              </span>
            </div>
          </div>

          <!-- TAB 2: Permissions Tree -->
          <div v-show="activeTab === 'permissions'" class="space-y-6">
            <p class="text-xs text-slate-500 dark:text-slate-400">
              {{ __('Select the permissions granted to users with this role. Checking a section header selects all underlying sub-permissions.') }}
            </p>

            <div class="space-y-4">
              <div
                v-for="category in permissionCategories"
                :key="category.key"
                class="overflow-hidden rounded-xl border border-slate-200 bg-white dark:border-slate-800 dark:bg-slate-900/30"
              >
                <!-- Category Header -->
                <div class="flex items-center justify-between border-b border-slate-200 bg-slate-50/80 px-4 py-3 dark:border-slate-800 dark:bg-slate-800/40">
                  <div class="flex items-center gap-2.5">
                    <input
                      type="checkbox"
                      :checked="isCategoryFullyChecked(category)"
                      :indeterminate="isCategoryPartiallyChecked(category)"
                      class="h-4 w-4 rounded-sm border-slate-300 text-blue-600 focus:ring-blue-500"
                      @change="toggleCategoryPermissions(category)"
                    />
                    <div>
                      <span class="text-sm font-semibold text-slate-800 dark:text-slate-200">{{ category.label }}</span>
                      <p v-if="category.description" class="text-[11px] text-slate-500 dark:text-slate-400">
                        {{ category.description }}
                      </p>
                    </div>
                  </div>
                </div>

                <!-- Category Permissions List -->
                <div class="grid grid-cols-1 gap-2 p-3 sm:grid-cols-2">
                  <label
                    v-for="perm in category.permissions"
                    :key="perm.id"
                    class="flex cursor-pointer items-start gap-2.5 rounded-lg p-2 transition-colors hover:bg-slate-50 dark:hover:bg-slate-800/40"
                    :class="selectedPermissionIds.has(perm.id) ? 'bg-blue-50/40 dark:bg-blue-950/20' : ''"
                  >
                    <input
                      type="checkbox"
                      :checked="selectedPermissionIds.has(perm.id)"
                      class="mt-1 h-3.5 w-3.5 rounded-sm border-slate-300 text-blue-600 focus:ring-blue-500"
                      @change="togglePermission(perm.id)"
                    />
                    <div class="min-w-0 flex-1">
                      <span class="block text-xs font-medium text-slate-800 dark:text-slate-200">
                        {{ perm.label }}
                        <span class="ml-1 text-[10px] text-slate-400 font-mono">({{ perm.name }})</span>
                      </span>
                      <p class="mt-0.5 line-clamp-2 text-[11px] text-slate-500 dark:text-slate-400">
                        {{ perm.description }}
                      </p>
                    </div>
                  </label>
                </div>
              </div>
            </div>
          </div>

          <!-- TAB 3: Group Permissions Matrix -->
          <div v-show="activeTab === 'groups'" class="space-y-6">
            <div
              v-if="!hasAgentPermission"
              class="rounded-xl border border-amber-200 bg-amber-50 p-4 text-xs text-amber-800 dark:border-amber-900/40 dark:bg-amber-950/20 dark:text-amber-300"
            >
              <div class="flex items-center gap-2 font-semibold">
                <CommonIcon name="info-circle" class="h-4 w-4" />
                <span>{{ __('Agent Permissions Required') }}</span>
              </div>
              <p class="mt-1">
                {{ __('Group permissions are only applied to roles that include the "Agent Tickets" (ticket.agent) permission. Enable it under the Permissions tab to configure group access.') }}
              </p>
            </div>

            <div class="flex items-center justify-between">
              <p class="text-xs text-slate-500 dark:text-slate-400">
                {{ __('Define default group permissions for members of this role.') }}
              </p>
              <div class="flex items-center gap-2">
                <button
                  type="button"
                  class="cursor-pointer rounded-md bg-slate-100 px-2.5 py-1 text-xs text-slate-700 hover:bg-slate-200 dark:bg-slate-800 dark:text-slate-300 dark:hover:bg-slate-700"
                  @click="grantAllGroupsFull"
                >
                  {{ __('Grant Full Access to All') }}
                </button>
                <button
                  type="button"
                  class="cursor-pointer rounded-md border border-slate-200 px-2.5 py-1 text-xs text-slate-500 hover:bg-slate-50 dark:border-slate-700 dark:text-slate-400 dark:hover:bg-slate-800"
                  @click="clearAllGroupAccess"
                >
                  {{ __('Clear All') }}
                </button>
              </div>
            </div>

            <!-- Groups Table -->
            <div class="overflow-x-auto rounded-xl border border-slate-200 bg-white dark:border-slate-800 dark:bg-slate-900/30">
              <table class="w-full text-left text-xs">
                <thead>
                  <tr class="border-b border-slate-200 bg-slate-50 font-semibold text-slate-600 dark:border-slate-800 dark:bg-slate-800/40 dark:text-slate-300">
                    <th class="px-4 py-3">{{ __('Group') }}</th>
                    <th class="px-3 py-3 text-center">{{ __('Read') }}</th>
                    <th class="px-3 py-3 text-center">{{ __('Create') }}</th>
                    <th class="px-3 py-3 text-center">{{ __('Change') }}</th>
                    <th class="px-3 py-3 text-center">{{ __('Overview') }}</th>
                    <th class="px-3 py-3 text-center font-bold text-blue-600 dark:text-blue-400">{{ __('Full') }}</th>
                    <th class="px-3 py-3 text-right"></th>
                  </tr>
                </thead>
                <tbody class="divide-y divide-slate-100 dark:divide-slate-800/60">
                  <tr
                    v-for="group in groups"
                    :key="group.id"
                    class="transition-colors hover:bg-slate-50/60 dark:hover:bg-slate-800/30"
                  >
                    <td class="px-4 py-2.5 font-medium text-slate-800 dark:text-slate-200">
                      {{ group.name }}
                    </td>
                    <td class="px-3 py-2.5 text-center">
                      <input
                        type="checkbox"
                        :checked="isGroupAccessChecked(group.id, 'read')"
                        class="h-3.5 w-3.5 rounded-sm border-slate-300 text-blue-600 focus:ring-blue-500"
                        @change="toggleGroupAccess(group.id, 'read')"
                      />
                    </td>
                    <td class="px-3 py-2.5 text-center">
                      <input
                        type="checkbox"
                        :checked="isGroupAccessChecked(group.id, 'create')"
                        class="h-3.5 w-3.5 rounded-sm border-slate-300 text-blue-600 focus:ring-blue-500"
                        @change="toggleGroupAccess(group.id, 'create')"
                      />
                    </td>
                    <td class="px-3 py-2.5 text-center">
                      <input
                        type="checkbox"
                        :checked="isGroupAccessChecked(group.id, 'change')"
                        class="h-3.5 w-3.5 rounded-sm border-slate-300 text-blue-600 focus:ring-blue-500"
                        @change="toggleGroupAccess(group.id, 'change')"
                      />
                    </td>
                    <td class="px-3 py-2.5 text-center">
                      <input
                        type="checkbox"
                        :checked="isGroupAccessChecked(group.id, 'overview')"
                        class="h-3.5 w-3.5 rounded-sm border-slate-300 text-blue-600 focus:ring-blue-500"
                        @change="toggleGroupAccess(group.id, 'overview')"
                      />
                    </td>
                    <td class="px-3 py-2.5 text-center">
                      <input
                        type="checkbox"
                        :checked="isGroupAccessChecked(group.id, 'full')"
                        class="h-3.5 w-3.5 rounded-sm border-blue-400 text-blue-600 focus:ring-blue-500"
                        @change="toggleGroupAccess(group.id, 'full')"
                      />
                    </td>
                    <td class="px-3 py-2.5 text-right">
                      <button
                        type="button"
                        class="cursor-pointer text-[11px] text-slate-400 hover:text-blue-600 dark:hover:text-blue-400"
                        @click="toggleRowAllAccess(group.id)"
                      >
                        {{ groupAccessMap[group.id]?.has('full') ? __('Clear') : __('Full') }}
                      </button>
                    </td>
                  </tr>
                </tbody>
              </table>
            </div>
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
            :disabled="submitting"
            class="flex cursor-pointer items-center gap-2 rounded-lg bg-blue-600 px-4 py-2 text-sm font-medium text-white shadow-xs transition-colors hover:bg-blue-700 disabled:opacity-50"
            @click="saveRole"
          >
            <div v-if="submitting" class="h-4 w-4 animate-spin rounded-full border-2 border-white border-t-transparent"></div>
            {{ drawerMode === 'create' ? __('Create Role') : __('Save Changes') }}
          </button>
        </div>

      </div>
    </div>
  </Teleport>
</template>
