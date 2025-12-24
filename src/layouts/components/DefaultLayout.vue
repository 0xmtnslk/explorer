<script lang="ts" setup>
import { Icon } from '@iconify/vue';
import { computed, ref } from 'vue';

import newFooter from '@/layouts/components/NavFooter.vue';
import NavbarThemeSwitcher from '@/layouts/components/NavbarThemeSwitcher.vue';
import ChainProfile from '@/layouts/components/ChainProfile.vue';

import { useDashboard } from '@/stores/useDashboard';
import { NetworkType } from '@/types/chaindata';
import { useBaseStore, useBlockchain } from '@/stores';

import NavBarI18n from './NavBarI18n.vue';
import NavBarWallet from './NavBarWallet.vue';
import dayjs from 'dayjs';
import { useRouter } from 'vue-router';

const dashboard = useDashboard();
dashboard.initial();
const blockchain = useBlockchain();
blockchain.randomSetupEndpoint();
const baseStore = useBaseStore();
const router = useRouter();

const current = ref('');
const temp = ref('');
const mobileMenuOpen = ref(false);
const chainSelectorOpen = ref(false);
const searchOpen = ref(false);
const searchQuery = ref('');
const searchError = ref('');
const selectedNetworkType = ref<'mainnet' | 'testnet'>('mainnet');

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
  return Object.values(dashboard.chains).filter(c => c.networkType === 'mainnet' || c.networkType === NetworkType.Mainnet);
});

const testnetChains = computed(() => {
  return Object.values(dashboard.chains).filter(c => c.networkType === 'testnet' || c.networkType === NetworkType.Testnet);
});

const displayedChains = computed(() => {
  return selectedNetworkType.value === 'mainnet' ? mainnetChains.value : testnetChains.value;
});

const currentChainInfo = computed(() => {
  if (!blockchain.chainName) return null;
  return dashboard.chains[blockchain.chainName];
});

function toggleMobileMenu() {
  mobileMenuOpen.value = !mobileMenuOpen.value;
  if (mobileMenuOpen.value) {
    chainSelectorOpen.value = false;
    searchOpen.value = false;
  }
}

function toggleChainSelector() {
  chainSelectorOpen.value = !chainSelectorOpen.value;
  if (chainSelectorOpen.value) {
    mobileMenuOpen.value = false;
    searchOpen.value = false;
  }
}

function toggleSearch() {
  if (!blockchain.chainName) {
    searchError.value = 'Please select a chain first';
    return;
  }
  searchOpen.value = !searchOpen.value;
  if (searchOpen.value) {
    mobileMenuOpen.value = false;
    chainSelectorOpen.value = false;
    searchError.value = '';
    searchQuery.value = '';
  }
}

function closeAllDropdowns() {
  mobileMenuOpen.value = false;
  chainSelectorOpen.value = false;
  searchOpen.value = false;
}

function selectChain(chainName: string) {
  router.push(`/${chainName}`);
  chainSelectorOpen.value = false;
}

function handleSearch() {
  searchError.value = '';
  const key = searchQuery.value.trim();
  
  if (!key) {
    searchError.value = 'Please enter a value';
    return;
  }

  const chainName = blockchain.chainName;
  if (!chainName) {
    searchError.value = 'Please select a chain first';
    return;
  }

  const height = /^\d+$/;
  const txhash = /^[A-Z\d]{64}$/;
  const addr = /^[a-z\d]+1[a-z\d]{38,58}$/;

  if (height.test(key)) {
    router.push(`/${chainName}/block/${key}`);
    closeAllDropdowns();
  } else if (txhash.test(key)) {
    router.push(`/${chainName}/tx/${key}`);
    closeAllDropdowns();
  } else if (addr.test(key)) {
    router.push(`/${chainName}/account/${key}`);
    closeAllDropdowns();
  } else {
    searchError.value = 'Invalid format. Enter block height, tx hash, or address.';
  }
}

</script>

<template>
  <div class="min-h-screen bg-gradient-to-br from-gray-50 to-gray-100 dark:from-[#0f1318] dark:to-[#1a1f2e]">
    <!-- Overlay for closing dropdowns -->
    <div 
      v-if="mobileMenuOpen || chainSelectorOpen || searchOpen"
      class="fixed inset-0 z-40"
      @click="closeAllDropdowns"
    />
    
    <!-- Top Navigation -->
    <nav class="sticky top-0 z-50 bg-white/80 dark:bg-[#1a1f2e]/90 backdrop-blur-lg border-b border-gray-200 dark:border-gray-700/50">
      <div class="max-w-7xl mx-auto px-4 sm:px-6 lg:px-8">
        <div class="flex items-center h-16 gap-4">
          <!-- Left: Logo -->
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

          <!-- Chain Selector -->
          <div v-if="blockchain.chainName" class="relative flex-shrink-0">
            <button 
              @click.stop="toggleChainSelector"
              class="flex items-center gap-2 px-3 py-1.5 rounded-lg bg-gray-100 dark:bg-gray-800 hover:bg-gray-200 dark:hover:bg-gray-700 transition-colors"
            >
              <img 
                v-if="currentChainInfo?.logo" 
                :src="currentChainInfo.logo" 
                class="w-5 h-5 rounded-full"
                :alt="blockchain.chainName"
              />
              <span class="text-sm font-medium text-gray-700 dark:text-gray-300 hidden sm:inline">
                {{ currentChainInfo?.prettyName || blockchain.chainName }}
              </span>
              <Icon icon="mdi:chevron-down" class="text-gray-500" :class="{ 'rotate-180': chainSelectorOpen }" />
            </button>

            <!-- Chain Selector Dropdown -->
            <div 
              v-if="chainSelectorOpen"
              class="absolute top-full left-0 mt-2 w-72 bg-white dark:bg-gray-800 rounded-xl shadow-xl border border-gray-200 dark:border-gray-700 overflow-hidden z-50"
            >
              <!-- Network Type Tabs -->
              <div class="flex border-b border-gray-200 dark:border-gray-700">
                <button 
                  @click="selectedNetworkType = 'mainnet'"
                  class="flex-1 px-4 py-2 text-sm font-medium transition-colors"
                  :class="selectedNetworkType === 'mainnet' ? 'bg-primary text-white' : 'text-gray-600 dark:text-gray-400 hover:bg-gray-100 dark:hover:bg-gray-700'"
                >
                  Mainnet ({{ mainnetChains.length }})
                </button>
                <button 
                  @click="selectedNetworkType = 'testnet'"
                  class="flex-1 px-4 py-2 text-sm font-medium transition-colors"
                  :class="selectedNetworkType === 'testnet' ? 'bg-primary text-white' : 'text-gray-600 dark:text-gray-400 hover:bg-gray-100 dark:hover:bg-gray-700'"
                >
                  Testnet ({{ testnetChains.length }})
                </button>
              </div>

              <!-- Chains List -->
              <div class="max-h-64 overflow-y-auto">
                <button 
                  v-for="chain in displayedChains" 
                  :key="chain.chainName"
                  @click="selectChain(chain.chainName)"
                  class="w-full flex items-center gap-3 px-4 py-2.5 hover:bg-gray-100 dark:hover:bg-gray-700 transition-colors"
                  :class="{ 'bg-primary/10': blockchain.chainName === chain.chainName }"
                >
                  <img :src="chain.logo" class="w-6 h-6 rounded-full" :alt="chain.chainName" />
                  <span class="text-sm font-medium text-gray-700 dark:text-gray-300">{{ chain.prettyName }}</span>
                  <Icon v-if="blockchain.chainName === chain.chainName" icon="mdi:check" class="ml-auto text-primary" />
                </button>
              </div>
            </div>
          </div>

          <!-- Center: Desktop Navigation (md and above only) -->
          <div v-if="blockchain.chainName" class="hidden md:flex items-center gap-1 flex-1 justify-center">
            <RouterLink
              v-for="item in navItems"
              :key="item.title"
              :to="item.to"
              class="flex items-center gap-1.5 px-3 py-2 rounded-lg text-sm font-medium text-gray-600 dark:text-gray-300 hover:bg-gray-100 dark:hover:bg-gray-700/50 hover:text-primary transition-all whitespace-nowrap"
              :class="{ '!bg-primary/10 !text-primary': $route.path === item.to || $route.path.startsWith(item.to + '/') }"
            >
              <Icon :icon="item.icon" class="text-lg" />
              <span>{{ item.title }}</span>
            </RouterLink>
          </div>
          
          <!-- Spacer for mobile -->
          <div class="flex-1 md:hidden"></div>

          <!-- Right: Actions -->
          <div class="flex items-center gap-2 flex-shrink-0">
            <ChainProfile class="hidden md:flex" />
            <NavBarI18n class="hidden md:block" />
            <NavbarThemeSwitcher />
            
            <!-- Search Button -->
            <div class="dropdown-container relative">
              <button 
                @click.stop="toggleSearch"
                class="btn btn-ghost btn-circle btn-sm"
                :class="{ 'opacity-50': !blockchain.chainName }"
                :title="blockchain.chainName ? 'Search' : 'Select a chain to search'"
              >
                <Icon icon="mdi:magnify" class="text-xl text-gray-500 dark:text-gray-400" />
              </button>

              <!-- Search Dropdown -->
              <div 
                v-if="searchOpen"
                class="absolute top-full right-0 mt-2 w-80 bg-white dark:bg-gray-800 rounded-xl shadow-xl border border-gray-200 dark:border-gray-700 p-4 z-50"
              >
                <div class="flex items-center justify-between mb-3">
                  <span class="text-sm font-semibold text-gray-700 dark:text-gray-300">Search {{ currentChainInfo?.prettyName }}</span>
                  <button @click="searchOpen = false" class="text-gray-400 hover:text-gray-600">
                    <Icon icon="mdi:close" />
                  </button>
                </div>
                <input 
                  v-model="searchQuery"
                  @keyup.enter="handleSearch"
                  type="text"
                  placeholder="Block height, Tx hash, or Address"
                  class="w-full px-3 py-2 rounded-lg border border-gray-300 dark:border-gray-600 bg-gray-50 dark:bg-gray-700 text-sm focus:outline-none focus:ring-2 focus:ring-primary"
                />
                <p v-if="searchError" class="text-xs text-red-500 mt-2">{{ searchError }}</p>
                <button 
                  @click="handleSearch"
                  class="w-full mt-3 px-4 py-2 bg-primary text-white rounded-lg text-sm font-medium hover:bg-primary/90 transition-colors"
                >
                  Search
                </button>
              </div>
            </div>

            <NavBarWallet />

            <!-- Mobile Menu Button (below md) -->
            <button 
              v-if="blockchain.chainName"
              @click.stop="toggleMobileMenu"
              class="md:hidden btn btn-ghost btn-circle btn-sm"
            >
              <Icon :icon="mobileMenuOpen ? 'mdi:close' : 'mdi:menu'" class="text-xl text-gray-600 dark:text-gray-300" />
            </button>
          </div>
        </div>

        <!-- Mobile Menu -->
        <div 
          v-if="mobileMenuOpen && blockchain.chainName"
          class="md:hidden border-t border-gray-200 dark:border-gray-700 py-3"
        >
          <div class="space-y-1">
            <RouterLink
              v-for="item in navItems"
              :key="item.title"
              :to="item.to"
              @click="mobileMenuOpen = false"
              class="flex items-center gap-3 px-3 py-2.5 rounded-lg text-sm font-medium text-gray-600 dark:text-gray-300 hover:bg-gray-100 dark:hover:bg-gray-700/50 hover:text-primary transition-all"
              :class="{ '!bg-primary/10 !text-primary': $route.path === item.to || $route.path.startsWith(item.to + '/') }"
            >
              <Icon :icon="item.icon" class="text-lg" />
              <span>{{ item.title }}</span>
            </RouterLink>
          </div>

          <!-- Mobile Chain Profile & I18n -->
          <div class="flex items-center gap-2 mt-3 pt-3 border-t border-gray-200 dark:border-gray-700">
            <ChainProfile class="flex-1" />
            <NavBarI18n />
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

.scrollbar-hide::-webkit-scrollbar {
  display: none;
}

.scrollbar-hide {
  -ms-overflow-style: none;
  scrollbar-width: none;
}
</style>
