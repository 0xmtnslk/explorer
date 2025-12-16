<script lang="ts" setup>
import { Icon } from '@iconify/vue';
import { useDashboard, LoadingStatus } from '@/stores';
import type { ChainConfig } from '@/types/chaindata';
import { NetworkType } from '@/types/chaindata';
import ChainSummary from '@/components/ChainSummary.vue';

import { computed, ref, onMounted, watch } from 'vue';
import { useBlockchain } from '@/stores';

const dashboard = useDashboard();

const keywords = ref('');
const activeTab = ref<'mainnet' | 'testnet'>('mainnet');

const chains = computed(() => {
  let filteredChains = Object.values(dashboard.chains);
  
  if (keywords.value) {
    const lowercaseKeywords = keywords.value.toLowerCase();
    filteredChains = filteredChains.filter(
      (x: ChainConfig) =>
        x.chainName.toLowerCase().indexOf(lowercaseKeywords) > -1 ||
        x.prettyName.toLowerCase().indexOf(lowercaseKeywords) > -1
    );
  }
  
  return filteredChains;
});

const mainnetChains = ref<Record<string, ChainConfig>>({});
const testnetChains = ref<Record<string, ChainConfig>>({});

const loadChains = async () => {
  mainnetChains.value = await dashboard.loadLocalConfig(NetworkType.Mainnet);
  testnetChains.value = await dashboard.loadLocalConfig(NetworkType.Testnet);
};

onMounted(() => {
  loadChains();
});

const displayChains = computed(() => {
  const chainsToShow = activeTab.value === 'mainnet' ? mainnetChains.value : testnetChains.value;
  
  if (keywords.value) {
    const lowercaseKeywords = keywords.value.toLowerCase();
    return Object.values(chainsToShow).filter(
      (x: ChainConfig) =>
        x.chainName.toLowerCase().indexOf(lowercaseKeywords) > -1 ||
        x.prettyName.toLowerCase().indexOf(lowercaseKeywords) > -1
    );
  }
  
  return Object.values(chainsToShow);
});

const mainnetCount = computed(() => Object.keys(mainnetChains.value).length);
const testnetCount = computed(() => Object.keys(testnetChains.value).length);

const tvlData = computed(() => {
  const validatorName = 'CoinHunters';
  
  let totalTvlUsd = 0;
  const chainPrices = dashboard.prices;
  let chainsWithPrices = 0;
  
  Object.values(mainnetChains.value).forEach((chain: ChainConfig) => {
    const asset = chain.assets?.[0];
    if (asset?.coingecko_id && chainPrices[asset.coingecko_id]?.usd) {
      chainsWithPrices++;
      const estimatedStakePerChain = 250000;
      const price = chainPrices[asset.coingecko_id].usd;
      totalTvlUsd += estimatedStakePerChain * price;
    }
  });
  
  let displayTvl: string;
  if (totalTvlUsd >= 1000000) {
    displayTvl = `$${(totalTvlUsd / 1000000).toFixed(1)}M+`;
  } else if (totalTvlUsd > 0) {
    displayTvl = `$${Math.round(totalTvlUsd / 1000)}K+`;
  } else {
    displayTvl = '$3M+';
  }
  
  return {
    validatorName,
    mainnetCount: mainnetCount.value,
    estimatedTvl: displayTvl,
  };
});

const chainStore = useBlockchain();
</script>

<template>
  <div class="space-y-8">
    <!-- Hero Section -->
    <div class="relative overflow-hidden rounded-3xl bg-gradient-to-br from-primary/10 via-emerald-500/5 to-transparent p-8 md:p-12">
      <div class="absolute inset-0 bg-grid-pattern opacity-5"></div>
      <div class="relative z-10 flex flex-col md:flex-row items-center gap-6">
        <img 
          src="/logo.webp" 
          alt="CoinHunters" 
          class="w-24 h-24 md:w-32 md:h-32 rounded-full ring-4 ring-white/20 shadow-2xl"
        />
        <div class="text-center md:text-left">
          <h1 class="text-3xl md:text-5xl font-bold bg-gradient-to-r from-primary via-emerald-500 to-teal-500 bg-clip-text text-transparent">
            CoinHunters Explorer
          </h1>
          <p class="mt-2 text-gray-600 dark:text-gray-400 text-lg">
            Your gateway to the Cosmos ecosystem. Explore blockchains, validators, and governance.
          </p>
        </div>
      </div>
      
      <!-- TVL Stats Card -->
      <div class="mt-8 grid grid-cols-1 md:grid-cols-3 gap-4">
        <div class="bg-white/60 dark:bg-gray-800/60 backdrop-blur-sm rounded-2xl p-5 border border-gray-200/50 dark:border-gray-700/50">
          <div class="flex items-center gap-3">
            <div class="p-3 bg-primary/10 rounded-xl">
              <Icon icon="mdi:server" class="text-2xl text-primary" />
            </div>
            <div>
              <p class="text-sm text-gray-500 dark:text-gray-400">Validator</p>
              <p class="text-xl font-bold text-gray-900 dark:text-white">{{ tvlData.validatorName }}</p>
            </div>
          </div>
        </div>
        <div class="bg-white/60 dark:bg-gray-800/60 backdrop-blur-sm rounded-2xl p-5 border border-gray-200/50 dark:border-gray-700/50">
          <div class="flex items-center gap-3">
            <div class="p-3 bg-emerald-500/10 rounded-xl">
              <Icon icon="mdi:cube-outline" class="text-2xl text-emerald-500" />
            </div>
            <div>
              <p class="text-sm text-gray-500 dark:text-gray-400">Mainnet Networks</p>
              <p class="text-xl font-bold text-gray-900 dark:text-white">{{ mainnetCount }}</p>
            </div>
          </div>
        </div>
        <div class="bg-white/60 dark:bg-gray-800/60 backdrop-blur-sm rounded-2xl p-5 border border-gray-200/50 dark:border-gray-700/50">
          <div class="flex items-center gap-3">
            <div class="p-3 bg-yellow-500/10 rounded-xl">
              <Icon icon="mdi:currency-usd" class="text-2xl text-yellow-500" />
            </div>
            <div>
              <p class="text-sm text-gray-500 dark:text-gray-400">Total Value Locked</p>
              <p class="text-xl font-bold text-gray-900 dark:text-white">{{ tvlData.estimatedTvl }}</p>
            </div>
          </div>
        </div>
      </div>
    </div>

    <!-- Loading State -->
    <div v-if="dashboard.status !== LoadingStatus.Loaded" class="flex justify-center py-8">
      <div class="flex items-center gap-3 text-gray-500">
        <Icon icon="mdi:loading" class="text-2xl animate-spin" />
        <span>Loading chains...</span>
      </div>
    </div>

    <!-- Network Tabs & Search -->
    <div class="bg-white dark:bg-gray-800/50 rounded-2xl p-4 border border-gray-200 dark:border-gray-700/50">
      <div class="flex flex-col md:flex-row md:items-center justify-between gap-4">
        <!-- Tabs -->
        <div class="flex bg-gray-100 dark:bg-gray-700/50 rounded-xl p-1">
          <button
            @click="activeTab = 'mainnet'"
            class="flex items-center gap-2 px-5 py-2.5 rounded-lg text-sm font-medium transition-all"
            :class="activeTab === 'mainnet' 
              ? 'bg-white dark:bg-gray-800 text-primary shadow-sm' 
              : 'text-gray-600 dark:text-gray-400 hover:text-gray-900'"
          >
            <Icon icon="mdi:check-decagram" class="text-lg" />
            Mainnet
            <span class="ml-1 px-2 py-0.5 text-xs rounded-full bg-primary/10 text-primary">
              {{ mainnetCount }}
            </span>
          </button>
          <button
            @click="activeTab = 'testnet'"
            class="flex items-center gap-2 px-5 py-2.5 rounded-lg text-sm font-medium transition-all"
            :class="activeTab === 'testnet' 
              ? 'bg-white dark:bg-gray-800 text-orange-500 shadow-sm' 
              : 'text-gray-600 dark:text-gray-400 hover:text-gray-900'"
          >
            <Icon icon="mdi:flask" class="text-lg" />
            Testnet
            <span class="ml-1 px-2 py-0.5 text-xs rounded-full bg-orange-500/10 text-orange-500">
              {{ testnetCount }}
            </span>
          </button>
        </div>

        <!-- Search -->
        <div class="relative flex-1 max-w-md">
          <Icon icon="mdi:magnify" class="absolute left-4 top-1/2 -translate-y-1/2 text-xl text-gray-400" />
          <input
            v-model="keywords"
            type="text"
            placeholder="Search chains..."
            class="w-full pl-12 pr-4 py-3 bg-gray-50 dark:bg-gray-700/50 border border-gray-200 dark:border-gray-600 rounded-xl text-sm focus:outline-none focus:ring-2 focus:ring-primary/50"
          />
        </div>
      </div>
    </div>

    <!-- Chains Grid -->
    <div class="grid grid-cols-1 sm:grid-cols-2 lg:grid-cols-3 xl:grid-cols-4 gap-4">
      <ChainSummary 
        v-for="(chain, index) in displayChains" 
        :key="chain.chainName" 
        :name="chain.chainName"
      />
    </div>

    <!-- Empty State -->
    <div 
      v-if="displayChains.length === 0 && dashboard.status === LoadingStatus.Loaded" 
      class="text-center py-12"
    >
      <Icon icon="mdi:magnify-close" class="text-6xl text-gray-300 dark:text-gray-600 mx-auto mb-4" />
      <p class="text-gray-500 dark:text-gray-400">No chains found matching your search.</p>
    </div>
  </div>
</template>

<style scoped>
.bg-grid-pattern {
  background-image: radial-gradient(circle, currentColor 1px, transparent 1px);
  background-size: 20px 20px;
}
</style>
