<script setup lang="ts">
import { useChatStore } from '@/stores/chat'
import { storeToRefs } from 'pinia'
import { ref, onMounted, nextTick } from 'vue'
import { useRouter, useRoute } from 'vue-router'
import type { AnalysisSession } from '@/types/chat'

import dayjs from 'dayjs'
import relativeTime from 'dayjs/plugin/relativeTime'
import 'dayjs/locale/zh-cn'

dayjs.extend(relativeTime)
dayjs.locale('zh-cn')

const chatStore = useChatStore()
const { sessions, isSidebarCollapsed: isCollapsed } = storeToRefs(chatStore)
const { toggleSidebar } = chatStore
const router = useRouter()
const route = useRoute()

// 重命名相关状态
const showRenameModal = ref(false)
const renameTarget = ref<AnalysisSession | null>(null)
const newName = ref('')
const renameInputRef = ref<HTMLInputElement | null>(null)

// 加载会话列表
onMounted(() => {
  chatStore.loadSessions()
})

function handleImport() {
  // Navigate to home (Welcome Guide)
  router.push('/')
}

function formatTime(timestamp: number): string {
  return dayjs.unix(timestamp).fromNow()
}

// 打开重命名弹窗
function openRenameModal(session: AnalysisSession) {
  renameTarget.value = session
  newName.value = session.name
  showRenameModal.value = true
  // 等待 DOM 更新后聚焦输入框
  nextTick(() => {
    renameInputRef.value?.focus()
    renameInputRef.value?.select()
  })
}

// 执行重命名
async function handleRename() {
  if (!renameTarget.value || !newName.value.trim()) return

  const success = await chatStore.renameSession(renameTarget.value.id, newName.value.trim())
  if (success) {
    showRenameModal.value = false
    renameTarget.value = null
    newName.value = ''
  }
}

// 关闭重命名弹窗
function closeRenameModal() {
  showRenameModal.value = false
  renameTarget.value = null
  newName.value = ''
}

// 删除会话
async function handleDelete(session: AnalysisSession) {
  await chatStore.deleteSession(session.id)
}

// 生成右键菜单项
function getContextMenuItems(session: AnalysisSession) {
  return [
    [
      {
        label: '重命名',
        icon: 'i-lucide-pencil',
        onSelect: () => openRenameModal(session),
      },
      {
        label: '删除',
        icon: 'i-lucide-trash',
        color: 'error' as const,
        onSelect: () => handleDelete(session),
      },
    ],
  ]
}
</script>

<template>
  <div
    class="flex h-full flex-col border-r border-gray-200 bg-gray-50 transition-all duration-300 ease-in-out dark:border-gray-800 dark:bg-gray-900"
    :class="[isCollapsed ? 'w-20' : 'w-72']"
  >
    <!-- Top Section -->
    <div class="flex flex-col p-4">
      <!-- Header / Toggle -->
      <div class="mb-6 flex items-center" :class="[isCollapsed ? 'justify-center' : 'justify-between']">
        <div v-if="!isCollapsed" class="text-lg font-semibold text-gray-900 dark:text-white">ChatLab</div>
        <UTooltip :text="isCollapsed ? '展开侧边栏' : '收起侧边栏'" :popper="{ placement: 'right' }">
          <UButton
            icon="i-heroicons-bars-3"
            color="gray"
            variant="ghost"
            size="md"
            class="flex h-12 w-12 cursor-pointer items-center justify-center rounded-full hover:bg-gray-200/60 dark:hover:bg-gray-800"
            @click="toggleSidebar"
          />
        </UTooltip>
      </div>

      <!-- New Analysis Button -->
      <UTooltip :text="isCollapsed ? '分析新聊天' : ''" :popper="{ placement: 'right' }">
        <UButton
          :block="!isCollapsed"
          class="transition-all rounded-full hover:bg-gray-200/60 dark:hover:bg-gray-800 h-12 cursor-pointer"
          :class="[isCollapsed ? 'flex w-12 items-center justify-center px-0' : 'justify-start pl-4']"
          color="gray"
          variant="ghost"
          @click="handleImport"
        >
          <UIcon name="i-heroicons-plus" class="h-5 w-5 shrink-0" :class="[isCollapsed ? '' : 'mr-2']" />
          <span v-if="!isCollapsed" class="truncate">分析新聊天</span>
        </UButton>
      </UTooltip>

      <!-- Tools Button -->
      <UTooltip :text="isCollapsed ? '实用工具' : ''" :popper="{ placement: 'right' }">
        <UButton
          :block="!isCollapsed"
          class="transition-all rounded-full hover:bg-gray-200/60 dark:hover:bg-gray-800 h-12 cursor-pointer mt-2"
          :class="[
            isCollapsed ? 'flex w-12 items-center justify-center px-0' : 'justify-start pl-4',
            route.name === 'tools'
              ? 'bg-primary-100 text-primary-600 dark:bg-primary-900/30 dark:text-primary-400'
              : '',
          ]"
          color="gray"
          variant="ghost"
          @click="router.push({ name: 'tools' })"
        >
          <UIcon name="i-heroicons-squares-2x2" class="h-5 w-5 shrink-0" :class="[isCollapsed ? '' : 'mr-2']" />
          <span v-if="!isCollapsed" class="truncate">实用工具</span>
        </UButton>
      </UTooltip>
    </div>

    <!-- Session List -->
    <div class="flex-1 overflow-y-auto px-3">
      <div v-if="sessions.length === 0 && !isCollapsed" class="py-8 text-center text-sm text-gray-500">暂无记录</div>

      <div class="space-y-1">
        <div v-if="!isCollapsed && sessions.length > 0" class="mb-2 px-2">
          <div class="text-xs font-medium text-gray-500">聊天记录</div>
          <div class="text-[10px] text-gray-400 font-normal mt-0.5">右键删除或重命名</div>
        </div>

        <UTooltip
          v-for="session in sessions"
          :key="session.id"
          :text="isCollapsed ? session.name : ''"
          :popper="{ placement: 'right' }"
        >
          <UContextMenu :items="getContextMenuItems(session)">
            <div
              class="group relative flex w-full items-center rounded-full p-2 text-left transition-colors"
              :class="[
                route.params.id === session.id && !isCollapsed
                  ? 'bg-primary-100 text-gray-900 dark:bg-primary-900/30 dark:text-primary-100'
                  : 'text-gray-700 dark:text-gray-200 hover:bg-gray-200/60 dark:hover:bg-gray-800',
                isCollapsed ? 'justify-center cursor-pointer' : 'cursor-pointer',
              ]"
              @click="router.push({ name: 'chat', params: { id: session.id } })"
            >
              <!-- Platform Icon / Text Avatar -->
              <div
                class="flex h-9 w-9 shrink-0 items-center justify-center rounded-full text-sm font-bold"
                :class="[
                  route.params.id === session.id
                    ? 'bg-primary-600 text-white dark:bg-primary-500 dark:text-white'
                    : 'bg-gray-400 text-white dark:bg-gray-600 dark:text-white',
                  isCollapsed ? '' : 'mr-3',
                ]"
              >
                {{ session.name ? session.name.charAt(0) : '?' }}
              </div>

              <!-- Session Info -->
              <div v-if="!isCollapsed" class="min-w-0 flex-1">
                <p class="truncate text-sm font-medium">
                  {{ session.name }}
                </p>
                <p class="truncate text-xs text-gray-500 dark:text-gray-400">
                  {{ session.messageCount }} 条消息 · {{ formatTime(session.importedAt) }}
                </p>
              </div>
            </div>
          </UContextMenu>
        </UTooltip>
      </div>
    </div>

    <!-- Rename Modal -->
    <UModal v-model:open="showRenameModal">
      <template #content>
        <div class="p-4">
          <h3 class="mb-3 font-semibold text-gray-900 dark:text-white">重命名</h3>
          <UInput
            ref="renameInputRef"
            v-model="newName"
            placeholder="请输入新名称"
            class="mb-4"
            @keydown.enter="handleRename"
          />
          <div class="flex justify-end gap-2">
            <UButton size="sm" color="gray" variant="soft" @click="closeRenameModal">取消</UButton>
            <UButton size="sm" color="primary" :disabled="!newName.trim()" @click="handleRename">确定</UButton>
          </div>
        </div>
      </template>
    </UModal>

    <!-- Footer -->
    <div class="border-t border-gray-200 p-4 dark:border-gray-800">
      <UTooltip :text="isCollapsed ? '设置和帮助' : ''" :popper="{ placement: 'right' }">
        <UButton
          :block="!isCollapsed"
          class="transition-all rounded-full hover:bg-gray-200/60 dark:hover:bg-gray-800 h-12 cursor-pointer"
          :class="[isCollapsed ? 'flex w-12 items-center justify-center px-0' : 'justify-start pl-4']"
          color="gray"
          variant="ghost"
        >
          <UIcon name="i-heroicons-cog-6-tooth" class="h-5 w-5 shrink-0" :class="[isCollapsed ? '' : 'mr-2']" />
          <span v-if="!isCollapsed" class="truncate">设置和帮助</span>
        </UButton>
      </UTooltip>
    </div>
  </div>
</template>
