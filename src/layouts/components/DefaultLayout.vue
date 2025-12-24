<script lang="ts" setup>
import { Icon } from '@iconify/vue';
import { computed, ref } from 'vue';

import newFooter from '@/layouts/components/NavFooter.vue';
import NavbarThemeSwitcher from '@/layouts/components/NavbarThemeSwitcher.vue';
import NavbarSearch from '@/layouts/components/NavbarSearch.vue';
import ChainProfile from '@/layouts/components/ChainProfile.vue';

import { useDashboard } from '@/stores/useDashboard';
import { NetworkType } from '@/types/chaindata';
import { useBaseStore, useBlockchain } from '@/stores';

import NavBarI18n from './NavBarI18n.vue';
import NavBarWallet from './NavBarWallet.vue';
import dayjs from 'dayjs';

const dashboard = useDashboard();
dashboard.initial();
const blockchain = useBlockchain();
blockchain.randomSetupEndpoint();
const baseStore = useBaseStore();

const current = ref('');
const temp = ref('');
blockchain.$subscribe((m, s) => {
  if (current.value === s.chainName && temp.value != s.endpoint.address) {
    temp.value = s.endpoint.address;
    blockchain.initial();
  }
  if (current.value != s.chainName) {
    current.value = s.chainName;
    blockchain.randomSetupEndpoint();
  }
});

const navItems = computed(() => [
  { title: 'Dashboard', icon: 'mdi:view-dashboard', to: `/${blockchain.chainName}` },
  { title: 'Governance', icon: 'mdi:vote', to: `/${blockchain.chainName}/gov` },
  { title: 'Staking', icon: 'mdi:server', to: `/${blockchain.chainName}/staking` },
  { title: 'Uptime', icon: 'mdi:chart-line', to: `/${blockchain.chainName}/uptime` },
  { title: 'Blocks', icon: 'mdi:cube-outline', to: `/${blockchain.chainName}/block` },
  { title: 'Transactions', icon: 'mdi:swap-horizontal', to: `/${blockchain.chainName}/tx` },
]);

const blocktime = computed(() => {
  return dayjs(baseStore.latest?.block?.header?.time);
});

const behind = computed(() => {
  const current = dayjs().subtract(10, 'minute');
  return blocktime.value.isBefore(current);
});
</script>

<template>
  <div class="min-h-screen bg-gradient-to-br from-gray-50 to-gray-100 dark:from-[#0f1318] dark:to-[#1a1f2e]">
    <!-- Top Navigation -->
    <nav class="sticky top-0 z-50 bg-white/80 dark:bg-[#1a1f2e]/90 backdrop-blur-lg border-b border-gray-200 dark:border-gray-700/50">
      <div class="max-w-7xl mx-auto px-4 sm:px-6 lg:px-8">
        <div class="flex items-center justify-between h-16">
          <!-- Logo -->
          <RouterLink to="/" class="flex items-center gap-3 group">
            <img 
              src="/logo.webp" 
              alt="CoinHunters" 
              class="w-10 h-10 rounded-full ring-2 ring-primary/20 group-hover:ring-primary/50 transition-all"
            />
            <div class="hidden sm:block">
              <h1 class="text-lg font-bold bg-gradient-to-r from-primary to-emerald-500 bg-clip-text text-transparent">
                CoinHunters
              </h1>
              <p class="text-[10px] text-gray-500 dark:text-gray-400 -mt-1">Explorer</p>
            </div>
          </RouterLink>

          <!-- Navigation -->
          <div v-if="blockchain.chainName" class="flex items-center gap-1 overflow-x-auto scrollbar-hide">
            <RouterLink
              v-for="item in navItems"
              :key="item.title"
              :to="item.to"
              class="flex items-center gap-1.5 px-3 py-2 rounded-lg text-sm font-medium text-gray-600 dark:text-gray-300 hover:bg-gray-100 dark:hover:bg-gray-700/50 hover:text-primary transition-all whitespace-nowrap"
              :class="{ '!bg-primary/10 !text-primary': $route.path === item.to || $route.path.startsWith(item.to + '/') }"
              :title="item.title"
            >
              <Icon :icon="item.icon" class="text-lg flex-shrink-0" />
              <span>{{ item.title }}</span>
            </RouterLink>
          </div>

          <!-- Right Actions -->
          <div class="flex items-center gap-2">
            <ChainProfile class="hidden md:flex" />
            <NavBarI18n class="hidden md:block" />
            <NavbarThemeSwitcher />
            <NavbarSearch />
            <NavBarWallet />
            
          </div>
        </div>
      </div>
    </nav>

    <!-- Main Content -->
    <main class="max-w-7xl mx-auto px-4 sm:px-6 lg:px-8 py-6">
      <!-- Sync Warning -->
      <div v-if="behind" class="mb-6 p-4 bg-red-500/10 border border-red-500/30 rounded-xl flex items-center gap-3">
        <Icon icon="mdi:alert-circle" class="text-2xl text-red-500" />
        <span class="text-red-600 dark:text-red-400">
          Chain is out of sync. Last block: {{ blocktime.format() }} ({{ blocktime.fromNow() }})
        </span>
      </div>

      <RouterView v-slot="{ Component }">
        <Transition mode="out-in" name="fade">
          <Component :is="Component" />
        </Transition>
      </RouterView>
    </main>

    <!-- Footer -->
    <newFooter />
  </div>
</template>

<style scoped>
.fade-enter-active,
.fade-leave-active {
  transition: opacity 0.2s ease;
}

.fade-enter-from,
.fade-leave-to {
  opacity: 0;
}
</style>
