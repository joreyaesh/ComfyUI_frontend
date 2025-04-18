<template>
  <div
    class="flex flex-col mx-auto overflow-hidden h-[83vh] relative"
    :aria-label="$t('manager.title')"
  >
    <Button
      v-if="isSmallScreen"
      :icon="isSideNavOpen ? 'pi pi-chevron-left' : 'pi pi-chevron-right'"
      text
      class="absolute top-1/2 -translate-y-1/2 z-10"
      :class="isSideNavOpen ? 'left-[19rem]' : 'left-2'"
      @click="toggleSideNav"
    />
    <div class="flex flex-1 relative overflow-hidden">
      <ManagerNavSidebar
        v-if="isSideNavOpen"
        :tabs="tabs"
        v-model:selectedTab="selectedTab"
      />
      <div
        class="flex-1 overflow-auto"
        :class="{
          'transition-all duration-300': isSmallScreen,
          'pl-80': isSideNavOpen || !isSmallScreen,
          'pl-8': !isSideNavOpen && isSmallScreen,
          'pr-80': showInfoPanel
        }"
      >
        <div class="px-6 pt-6 flex flex-col h-full">
          <RegistrySearchBar
            v-if="!hideSearchBar"
            v-model:searchQuery="searchQuery"
            v-model:searchMode="searchMode"
            :searchResults="searchResults"
          />
          <div class="flex-1 overflow-auto">
            <div
              v-if="isLoading || isInitialLoad"
              class="flex justify-center items-center h-full"
            >
              <ProgressSpinner />
            </div>
            <NoResultsPlaceholder
              v-else-if="searchResults.length === 0"
              :title="
                comfyManagerStore.error
                  ? $t('manager.errorConnecting')
                  : $t('manager.noResultsFound')
              "
              :message="
                comfyManagerStore.error
                  ? $t('manager.tryAgainLater')
                  : $t('manager.tryDifferentSearch')
              "
            />
            <div v-else class="h-full" @click="handleGridContainerClick">
              <VirtualGrid
                :items="resultsWithKeys"
                :defaultItemSize="DEFAULT_CARD_SIZE"
                class="p-0 m-0 max-w-full"
                :buffer-rows="2"
                :gridStyle="{
                  display: 'grid',
                  gridTemplateColumns: `repeat(auto-fill, minmax(${DEFAULT_CARD_SIZE}px, 1fr))`,
                  padding: '0.5rem',
                  gap: '1.125rem 1.25rem',
                  justifyContent: 'stretch'
                }"
              >
                <template #item="{ item }">
                  <div
                    class="relative w-full aspect-square cursor-pointer"
                    @click.stop="(event) => selectNodePack(item, event)"
                  >
                    <PackCard
                      :node-pack="item"
                      :is-selected="
                        selectedNodePacks.some((pack) => pack.id === item.id)
                      "
                    />
                  </div>
                </template>
              </VirtualGrid>
            </div>
          </div>
        </div>
      </div>
      <div
        v-if="showInfoPanel"
        class="w-80 border-l-0 border-surface-border absolute right-0 top-0 bottom-0 flex z-20"
      >
        <ContentDivider orientation="vertical" :width="0.2" />
        <div class="flex-1 flex flex-col isolate">
          <InfoPanel
            v-if="!hasMultipleSelections"
            :node-pack="selectedNodePack"
          />
          <InfoPanelMultiItem v-else :node-packs="selectedNodePacks" />
        </div>
      </div>
    </div>
  </div>
</template>

<script setup lang="ts">
import Button from 'primevue/button'
import ProgressSpinner from 'primevue/progressspinner'
import { computed, ref, watchEffect } from 'vue'
import { useI18n } from 'vue-i18n'

import ContentDivider from '@/components/common/ContentDivider.vue'
import NoResultsPlaceholder from '@/components/common/NoResultsPlaceholder.vue'
import VirtualGrid from '@/components/common/VirtualGrid.vue'
import ManagerNavSidebar from '@/components/dialog/content/manager/ManagerNavSidebar.vue'
import InfoPanel from '@/components/dialog/content/manager/infoPanel/InfoPanel.vue'
import InfoPanelMultiItem from '@/components/dialog/content/manager/infoPanel/InfoPanelMultiItem.vue'
import PackCard from '@/components/dialog/content/manager/packCard/PackCard.vue'
import RegistrySearchBar from '@/components/dialog/content/manager/registrySearchBar/RegistrySearchBar.vue'
import { useResponsiveCollapse } from '@/composables/element/useResponsiveCollapse'
import { useInstalledPacks } from '@/composables/useInstalledPacks'
import { useRegistrySearch } from '@/composables/useRegistrySearch'
import { useComfyManagerStore } from '@/stores/comfyManagerStore'
import type { TabItem } from '@/types/comfyManagerTypes'
import { components } from '@/types/comfyRegistryTypes'

const DEFAULT_CARD_SIZE = 512

const { t } = useI18n()
const comfyManagerStore = useComfyManagerStore()

const {
  isSmallScreen,
  isOpen: isSideNavOpen,
  toggle: toggleSideNav
} = useResponsiveCollapse()
const hideSearchBar = computed(() => isSmallScreen.value && showInfoPanel.value)

const tabs = ref<TabItem[]>([
  { id: 'all', label: t('g.all'), icon: 'pi-list' },
  { id: 'installed', label: t('g.installed'), icon: 'pi-box' }
])
const selectedTab = ref<TabItem>(tabs.value[0])

const { searchQuery, pageNumber, isLoading, searchResults, searchMode } =
  useRegistrySearch()
pageNumber.value = 0

const isInitialLoad = computed(
  () => searchResults.value.length === 0 && searchQuery.value === ''
)

const { getInstalledPacks } = useInstalledPacks()
const displayPacks = ref<components['schemas']['Node'][]>([])
const isEmptySearch = computed(() => searchQuery.value === '')

const getInstalledSearchResults = async () => {
  if (isEmptySearch.value) return getInstalledPacks()
  return searchResults.value.filter((pack) =>
    comfyManagerStore.installedPacksIds.has(pack.name)
  )
}

watchEffect(async () => {
  if (selectedTab.value.id === 'installed') {
    displayPacks.value = await getInstalledSearchResults()
  } else {
    displayPacks.value = searchResults.value
  }
})

const resultsWithKeys = computed(() =>
  displayPacks.value.map((item) => ({
    ...item,
    key: item.id || item.name
  }))
)

const selectedNodePacks = ref<components['schemas']['Node'][]>([])
const selectedNodePack = computed(() =>
  selectedNodePacks.value.length === 1 ? selectedNodePacks.value[0] : null
)

const selectNodePack = (
  nodePack: components['schemas']['Node'],
  event: MouseEvent
) => {
  // Handle multi-select with Shift or Ctrl/Cmd key
  if (event.shiftKey || event.ctrlKey || event.metaKey) {
    const index = selectedNodePacks.value.findIndex(
      (pack) => pack.id === nodePack.id
    )

    if (index === -1) {
      // Add to selection if not already selected
      selectedNodePacks.value.push(nodePack)
    } else {
      // Remove from selection if already selected
      selectedNodePacks.value.splice(index, 1)
    }
  } else {
    // Single select behavior
    selectedNodePacks.value = [nodePack]
  }
}

const unSelectItems = () => {
  selectedNodePacks.value = []
}
const handleGridContainerClick = (event: MouseEvent) => {
  const targetElement = event.target as HTMLElement
  if (targetElement && !targetElement.closest('[data-virtual-grid-item]')) {
    unSelectItems()
  }
}

const showInfoPanel = computed(() => selectedNodePacks.value.length > 0)
const hasMultipleSelections = computed(() => selectedNodePacks.value.length > 1)
</script>
