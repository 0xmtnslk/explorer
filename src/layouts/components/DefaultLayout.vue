<script lang="ts" setup>
import { Icon } from '@iconify/vue';
import { computed, ref, onMounted } from 'vue';
import { useRouter } from 'vue-router';

import newFooter from '@/layouts/components/NavFooter.vue';
import NavbarThemeSwitcher from '@/layouts/components/NavbarThemeSwitcher.vue';

import { useDashboard } from '@/stores/useDashboard';
import { NetworkType } from '@/types/chaindata';
import type { ChainConfig } from '@/types/chaindata';
import { useBaseStore, useBlockchain } from '@/stores';

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
const allMainnetChains = ref<ChainConfig[]>([]);
const allTestnetChains = ref<ChainConfig[]>([]);
const chainSearchQuery = ref('');

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

onMounted(async () => {
  const mainnetConfig = await dashboard.loadLocalConfig(NetworkType.Mainnet);
  const testnetConfig = await dashboard.loadLocalConfig(NetworkType.Testnet);
  allMainnetChains.value = Object.values(mainnetConfig).sort((a, b) => 
    (a.prettyName || a.chainName).localeCompare(b.prettyName || b.chainName)
  );
  allTestnetChains.value = Object.values(testnetConfig).sort((a, b) => 
    (a.prettyName || a.chainName).localeCompare(b.prettyName || b.chainName)
  );
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

const displayedChains = computed(() => {
  const chains = selectedNetworkType.value === 'mainnet' ? allMainnetChains.value : allTestnetChains.value;
  const query = chainSearchQuery.value.toLowerCase().trim();
  if (!query) return chains;
  return chains.filter(c => 
    (c.prettyName || c.chainName).toLowerCase().includes(query) ||
    c.chainName.toLowerCase().includes(query)
  );
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
  chainSearchQuery.value = '';
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
    <nav class="sticky top-0 z-50 bg-white dark:bg-[#1a1f2e] shadow-sm">
      <!-- Main Navbar -->
      <div class="border-b border-gray-200 dark:border-gray-700/50">
        <div class="max-w-7xl mx-auto px-4 sm:px-6 lg:px-8">
          <div class="flex items-center justify-between h-16">
            <!-- Left: Logo -->
            <div class="flex items-center gap-3">
              <RouterLink to="/" class="flex items-center gap-2 group flex-shrink-0">
                <img 
                  src="/logo.webp" 
                  alt="CoinHunters" 
                  class="w-9 h-9 rounded-full ring-2 ring-primary/30 group-hover:ring-primary transition-all"
                />
                <span class="hidden lg:block text-lg font-bold text-gray-800 dark:text-white">
                  CoinHunters
                </span>
              </RouterLink>
            </div>

            <!-- Center: Search Bar -->
            <div class="flex-1 flex justify-center px-4">
              <div class="relative w-full max-w-md">
                <Icon icon="mdi:magnify" class="absolute left-4 top-1/2 -translate-y-1/2 text-gray-400" />
                <input 
                  v-model="searchQuery"
                  @keyup.enter="handleSearch"
                  type="text"
                  placeholder="Search Block / Tx / Address"
                  class="w-full pl-11 pr-4 py-2.5 text-sm bg-gray-100 dark:bg-gray-800/80 border border-gray-200 dark:border-gray-700 rounded-xl focus:ring-2 focus:ring-primary/50 focus:border-primary text-gray-700 dark:text-gray-300 placeholder-gray-400 transition-all"
                />
                <div v-if="searchError" class="absolute top-full left-0 mt-1 text-xs text-red-500 bg-red-50 dark:bg-red-900/30 px-2 py-1 rounded">{{ searchError }}</div>
              </div>
            </div>

            <!-- Right: Chain Selector + Actions -->
            <div class="flex items-center gap-3">
              <!-- Current Chain Selector -->
              <div class="relative">
                <button 
                  @click="toggleNetworkDropdown"
                  class="flex items-center gap-2 px-3 py-2 rounded-xl bg-gray-100 dark:bg-gray-800 hover:bg-gray-200 dark:hover:bg-gray-700 transition-all border border-gray-200 dark:border-gray-700"
                >
                  <img 
                    v-if="blockchain.current?.logo" 
                    :src="blockchain.current.logo" 
                    class="w-6 h-6 rounded-full"
                    :alt="blockchain.current?.prettyName"
                  />
                  <div v-else class="w-6 h-6 rounded-full bg-gray-300 dark:bg-gray-600 flex items-center justify-center">
                    <Icon icon="mdi:cube-outline" class="text-xs text-gray-500" />
                  </div>
                  <span class="text-sm font-medium text-gray-700 dark:text-gray-200 max-w-[120px] truncate">
                    {{ blockchain.current?.prettyName || 'Select Network' }}
                  </span>
                  <Icon :icon="networkDropdownOpen ? 'mdi:chevron-up' : 'mdi:chevron-down'" class="text-gray-400" />
                </button>

                <!-- Backdrop -->
                <div v-if="networkDropdownOpen" @click="closeDropdown" class="fixed inset-0 z-40"></div>

                <!-- Dropdown Menu -->
                <div 
                  v-if="networkDropdownOpen" 
                  class="absolute left-0 top-full mt-2 w-80 bg-white dark:bg-gray-800 rounded-2xl shadow-2xl border border-gray-200 dark:border-gray-700 overflow-hidden z-50"
                >
                  <!-- Header -->
                  <div class="px-4 py-3 bg-gradient-to-r from-primary/10 to-emerald-500/10 border-b border-gray-200 dark:border-gray-700">
                    <div class="text-sm font-semibold text-gray-800 dark:text-white">Select Network</div>
                    <div class="text-xs text-gray-500 dark:text-gray-400">Choose a blockchain to explore</div>
                  </div>

                  <!-- Network Type Tabs -->
                  <div class="flex p-2 gap-2 border-b border-gray-200 dark:border-gray-700">
                    <button 
                      @click="selectedNetworkType = 'mainnet'"
                      class="flex-1 px-4 py-2 text-sm font-medium rounded-lg transition-all"
                      :class="selectedNetworkType === 'mainnet' ? 'bg-primary text-white shadow-md' : 'text-gray-600 dark:text-gray-400 hover:bg-gray-100 dark:hover:bg-gray-700'"
                    >
                      Mainnet ({{ allMainnetChains.length }})
                    </button>
                    <button 
                      @click="selectedNetworkType = 'testnet'"
                      class="flex-1 px-4 py-2 text-sm font-medium rounded-lg transition-all"
                      :class="selectedNetworkType === 'testnet' ? 'bg-primary text-white shadow-md' : 'text-gray-600 dark:text-gray-400 hover:bg-gray-100 dark:hover:bg-gray-700'"
                    >
                      Testnet ({{ allTestnetChains.length }})
                    </button>
                  </div>

                  <!-- Chain Search -->
                  <div class="p-2 border-b border-gray-200 dark:border-gray-700">
                    <div class="relative">
                      <Icon icon="mdi:magnify" class="absolute left-3 top-1/2 -translate-y-1/2 text-gray-400 text-sm" />
                      <input 
                        v-model="chainSearchQuery"
                        type="text"
                        :placeholder="selectedNetworkType === 'mainnet' ? 'Search mainnet chains...' : 'Search testnet chains...'"
                        class="w-full pl-9 pr-3 py-2 text-sm bg-gray-100 dark:bg-gray-700 border-0 rounded-lg focus:ring-2 focus:ring-primary/50 text-gray-700 dark:text-gray-300 placeholder-gray-400"
                      />
                    </div>
                  </div>
                  
                  <!-- Chain List -->
                  <div class="max-h-72 overflow-y-auto p-2">
                    <button 
                      v-for="chain in displayedChains" 
                      :key="chain.chainName"
                      @click="selectChain(chain.chainName)"
                      class="w-full flex items-center gap-3 px-3 py-2.5 rounded-xl hover:bg-gray-100 dark:hover:bg-gray-700/50 transition-all"
                      :class="{ 'bg-primary/10 ring-1 ring-primary/30': blockchain.chainName === chain.chainName }"
                    >
                      <img :src="chain.logo" class="w-8 h-8 rounded-full" :alt="chain.prettyName" />
                      <div class="flex-1 text-left">
                        <div class="text-sm font-medium text-gray-800 dark:text-white">{{ chain.prettyName || chain.chainName }}</div>
                      </div>
                      <Icon v-if="blockchain.chainName === chain.chainName" icon="mdi:check-circle" class="text-primary text-lg" />
                    </button>
                    <div v-if="displayedChains.length === 0" class="px-4 py-8 text-center">
                      <Icon icon="mdi:cube-off-outline" class="text-4xl text-gray-300 dark:text-gray-600 mb-2" />
                      <div class="text-sm text-gray-500">No chains available</div>
                    </div>
                  </div>
                </div>
              </div>

              <!-- Theme & Wallet -->
              <NavbarThemeSwitcher />
              <NavBarWallet />
            </div>
          </div>
        </div>
      </div>

      <!-- Navigation Menu -->
      <div v-if="blockchain.chainName" class="bg-gray-50 dark:bg-gray-800/50 border-b border-gray-200 dark:border-gray-700/50">
        <div class="max-w-7xl mx-auto px-4 sm:px-6 lg:px-8">
          <div class="flex items-center gap-1 py-2 overflow-x-auto scrollbar-hide">
            <RouterLink
              v-for="item in navItems"
              :key="item.title"
              :to="item.to"
              class="flex items-center gap-2 px-4 py-2 rounded-lg text-sm font-medium transition-all whitespace-nowrap"
              :class="$route.path === item.to || $route.path.startsWith(item.to + '/') 
                ? 'bg-primary text-white shadow-md' 
                : 'text-gray-600 dark:text-gray-300 hover:bg-gray-200 dark:hover:bg-gray-700'"
            >
              <Icon :icon="item.icon" class="text-lg" />
              <span>{{ item.title }}</span>
            </RouterLink>
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
