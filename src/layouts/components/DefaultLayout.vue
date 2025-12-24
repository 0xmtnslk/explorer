<script lang="ts" setup>
import { Icon } from '@iconify/vue';
import { computed, ref } from 'vue';
import { useRouter } from 'vue-router';

import newFooter from '@/layouts/components/NavFooter.vue';
import NavbarThemeSwitcher from '@/layouts/components/NavbarThemeSwitcher.vue';
import ChainProfile from '@/layouts/components/ChainProfile.vue';
import { onMounted, onUnmounted } from 'vue';

import { useDashboard } from '@/stores/useDashboard';
import { NetworkType } from '@/types/chaindata';
import { useBaseStore, useBlockchain } from '@/stores';

import NavBarI18n from './NavBarI18n.vue';
import NavBarWallet from './NavBarWallet.vue';
import dayjs from 'dayjs';

const router = useRouter();
const dashboard = useDashboard();
dashboard.initial();
const blockchain = useBlockchain();
blockchain.randomSetupEndpoint();
const baseStore = useBaseStore();

const current = ref('');
const temp = ref('');
const networkDropdownOpen = ref(false);
const selectedNetworkType = ref<'mainnet' | 'testnet'>('mainnet');
const searchQuery = ref('');
const searchError = ref('');

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

const mainnetChains = computed(() => {
  return Object.values(dashboard.chains)
    .filter(c => c.networkType === NetworkType.Mainnet || c.networkType === 'mainnet')
    .sort((a, b) => (a.prettyName || a.chainName).localeCompare(b.prettyName || b.chainName));
});

const testnetChains = computed(() => {
  return Object.values(dashboard.chains)
    .filter(c => c.networkType === NetworkType.Testnet || c.networkType === 'testnet')
    .sort((a, b) => (a.prettyName || a.chainName).localeCompare(b.prettyName || b.chainName));
});

const displayedChains = computed(() => {
  return selectedNetworkType.value === 'mainnet' ? mainnetChains.value : testnetChains.value;
});

function selectChain(chainName: string) {
  networkDropdownOpen.value = false;
  router.push(`/${chainName}`);
}

function toggleNetworkDropdown() {
  networkDropdownOpen.value = !networkDropdownOpen.value;
}

function closeDropdown() {
  networkDropdownOpen.value = false;
}

function handleSearch() {
  searchError.value = '';
  const key = searchQuery.value.trim();
  if (!key) return;
  
  const height = /^\d+$/;
  const txhash = /^[A-Z\d]{64}$/;
  const addr = /^[a-z\d]+1[a-z\d]{38,58}$/;
  const currentChain = blockchain?.chainName || '';

  if (height.test(key)) {
    router.push(`/${currentChain}/block/${key}`);
    searchQuery.value = '';
  } else if (txhash.test(key)) {
    router.push(`/${currentChain}/tx/${key}`);
    searchQuery.value = '';
  } else if (addr.test(key)) {
    router.push(`/${currentChain}/account/${key}`);
    searchQuery.value = '';
  } else {
    searchError.value = 'Invalid format';
  }
}
</script>

<template>
  <div class="min-h-screen bg-gradient-to-br from-gray-50 to-gray-100 dark:from-[#0f1318] dark:to-[#1a1f2e]">
    <!-- Top Navigation -->
    <nav class="sticky top-0 z-50 bg-white/80 dark:bg-[#1a1f2e]/90 backdrop-blur-lg border-b border-gray-200 dark:border-gray-700/50">
      <div class="max-w-7xl mx-auto px-4 sm:px-6 lg:px-8">
        <div class="flex items-center justify-between h-16">
          <!-- Logo -->
          <RouterLink to="/" class="flex items-center gap-3 group flex-shrink-0">
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

          <!-- Network Dropdown -->
          <div class="relative ml-4">
            <button 
              @click="toggleNetworkDropdown"
              class="flex items-center gap-2 px-3 py-2 rounded-xl bg-gray-100 dark:bg-gray-800 hover:bg-gray-200 dark:hover:bg-gray-700 transition-all"
            >
              <img 
                v-if="blockchain.current?.logo" 
                :src="blockchain.current.logo" 
                class="w-6 h-6 rounded-full"
                :alt="blockchain.current?.prettyName"
              />
              <Icon v-else icon="mdi:cube-outline" class="text-lg text-gray-500" />
              <span class="text-sm font-medium text-gray-700 dark:text-gray-300 hidden md:inline max-w-[100px] truncate">
                {{ blockchain.current?.prettyName || 'Select Network' }}
              </span>
              <Icon :icon="networkDropdownOpen ? 'mdi:chevron-up' : 'mdi:chevron-down'" class="text-gray-500" />
            </button>

            <!-- Backdrop -->
            <div v-if="networkDropdownOpen" @click="closeDropdown" class="fixed inset-0 z-40"></div>

            <!-- Dropdown Menu -->
            <div 
              v-if="networkDropdownOpen" 
              class="absolute left-0 top-full mt-2 w-72 bg-white dark:bg-gray-800 rounded-xl shadow-xl border border-gray-200 dark:border-gray-700 overflow-hidden z-50"
            >
              <!-- Network Type Tabs -->
              <div class="flex border-b border-gray-200 dark:border-gray-700">
                <button 
                  @click="selectedNetworkType = 'mainnet'"
                  class="flex-1 px-4 py-2.5 text-sm font-medium transition-colors"
                  :class="selectedNetworkType === 'mainnet' ? 'bg-primary/10 text-primary border-b-2 border-primary' : 'text-gray-600 dark:text-gray-400 hover:bg-gray-50 dark:hover:bg-gray-700'"
                >
                  Mainnet ({{ mainnetChains.length }})
                </button>
                <button 
                  @click="selectedNetworkType = 'testnet'"
                  class="flex-1 px-4 py-2.5 text-sm font-medium transition-colors"
                  :class="selectedNetworkType === 'testnet' ? 'bg-primary/10 text-primary border-b-2 border-primary' : 'text-gray-600 dark:text-gray-400 hover:bg-gray-50 dark:hover:bg-gray-700'"
                >
                  Testnet ({{ testnetChains.length }})
                </button>
              </div>
              
              <!-- Chain List -->
              <div class="max-h-64 overflow-y-auto">
                <button 
                  v-for="chain in displayedChains" 
                  :key="chain.chainName"
                  @click="selectChain(chain.chainName)"
                  class="w-full flex items-center gap-3 px-4 py-2.5 hover:bg-gray-50 dark:hover:bg-gray-700 transition-colors"
                  :class="{ 'bg-primary/5': blockchain.chainName === chain.chainName }"
                >
                  <img :src="chain.logo" class="w-7 h-7 rounded-full" :alt="chain.prettyName" />
                  <span class="text-sm font-medium text-gray-700 dark:text-gray-300">{{ chain.prettyName || chain.chainName }}</span>
                  <Icon v-if="blockchain.chainName === chain.chainName" icon="mdi:check" class="ml-auto text-primary" />
                </button>
                <div v-if="displayedChains.length === 0" class="px-4 py-6 text-center text-sm text-gray-500">
                  No chains available
                </div>
              </div>
            </div>
          </div>

          <!-- Navigation -->
          <div v-if="blockchain.chainName" class="hidden lg:flex items-center gap-1 mx-4">
            <RouterLink
              v-for="item in navItems"
              :key="item.title"
              :to="item.to"
              class="flex items-center gap-1.5 px-3 py-2 rounded-lg text-sm font-medium text-gray-600 dark:text-gray-300 hover:bg-gray-100 dark:hover:bg-gray-700/50 hover:text-primary transition-all whitespace-nowrap"
              :class="{ '!bg-primary/10 !text-primary': $route.path === item.to || $route.path.startsWith(item.to + '/') }"
              :title="item.title"
            >
              <Icon :icon="item.icon" class="text-lg flex-shrink-0" />
              <span class="hidden xl:inline">{{ item.title }}</span>
            </RouterLink>
          </div>

          <!-- Search Bar -->
          <div class="hidden md:flex items-center flex-1 max-w-xs mx-4">
            <div class="relative w-full">
              <Icon icon="mdi:magnify" class="absolute left-3 top-1/2 -translate-y-1/2 text-gray-400" />
              <input 
                v-model="searchQuery"
                @keyup.enter="handleSearch"
                type="text"
                placeholder="Block / Tx / Address"
                class="w-full pl-10 pr-4 py-2 text-sm bg-gray-100 dark:bg-gray-800 border-0 rounded-xl focus:ring-2 focus:ring-primary/50 text-gray-700 dark:text-gray-300 placeholder-gray-400"
              />
              <div v-if="searchError" class="absolute top-full left-0 mt-1 text-xs text-red-500">{{ searchError }}</div>
            </div>
          </div>

          <!-- Right Actions -->
          <div class="flex items-center gap-2 flex-shrink-0">
            <ChainProfile class="hidden md:flex" />
            <NavBarI18n class="hidden lg:block" />
            <NavbarThemeSwitcher />
            <NavBarWallet />
          </div>
        </div>

        <!-- Mobile Navigation -->
        <div v-if="blockchain.chainName" class="lg:hidden flex items-center gap-1 pb-3 overflow-x-auto scrollbar-hide">
          <RouterLink
            v-for="item in navItems"
            :key="item.title"
            :to="item.to"
            class="flex items-center gap-1.5 px-3 py-1.5 rounded-lg text-xs font-medium text-gray-600 dark:text-gray-300 hover:bg-gray-100 dark:hover:bg-gray-700/50 hover:text-primary transition-all whitespace-nowrap"
            :class="{ '!bg-primary/10 !text-primary': $route.path === item.to || $route.path.startsWith(item.to + '/') }"
          >
            <Icon :icon="item.icon" class="text-base flex-shrink-0" />
            <span>{{ item.title }}</span>
          </RouterLink>
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
