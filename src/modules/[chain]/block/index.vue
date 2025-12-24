<script lang="ts" setup>
import { computed, ref } from '@vue/reactivity';
import { useBaseStore, useFormatter } from '@/stores';
import TxsInBlocksChart from '@/components/charts/TxsInBlocksChart.vue';
import { Icon } from '@iconify/vue';

const props = defineProps(['chain']);

const tab = ref('blocks');

const base = useBaseStore();

const format = useFormatter();

const list = computed(() => {
  return base.recents;
});

const totalTxs = computed(() => {
  return base.recents.reduce((acc, block) => acc + (block.block?.data?.txs?.length || 0), 0);
});

const avgBlockTime = computed(() => {
  return base.blocktime ? (base.blocktime / 1000).toFixed(1) : '-';
});
</script>

<template>
  <div class="space-y-6">
    <!-- Hero Section -->
    <div class="relative overflow-hidden rounded-2xl bg-gradient-to-r from-indigo-500 via-purple-500 to-pink-500 p-6 md:p-8">
      <div class="absolute inset-0 bg-black/10"></div>
      <div class="absolute -top-24 -right-24 w-64 h-64 bg-white/10 rounded-full blur-3xl"></div>
      <div class="absolute -bottom-24 -left-24 w-64 h-64 bg-white/10 rounded-full blur-3xl"></div>
      
      <div class="relative z-10">
        <div class="flex items-center gap-3 mb-4">
          <div class="w-12 h-12 rounded-xl bg-white/20 backdrop-blur flex items-center justify-center">
            <Icon icon="mdi:cube-outline" class="text-2xl text-white" />
          </div>
          <div>
            <h1 class="text-2xl md:text-3xl font-bold text-white">{{ $t('block.recent') }}</h1>
            <p class="text-white/80 text-sm">Live blockchain blocks and transactions</p>
          </div>
        </div>
        
        <!-- Stats -->
        <div class="flex flex-wrap gap-4">
          <div class="bg-white/10 backdrop-blur rounded-xl px-4 py-2">
            <div class="text-white/70 text-xs">Latest Block</div>
            <div class="text-white font-bold text-lg">#{{ base.latest?.block?.header?.height?.toLocaleString() || '-' }}</div>
          </div>
          <div class="bg-white/10 backdrop-blur rounded-xl px-4 py-2">
            <div class="text-white/70 text-xs">Recent Transactions</div>
            <div class="text-white font-bold text-lg">{{ totalTxs }}</div>
          </div>
          <div class="bg-white/10 backdrop-blur rounded-xl px-4 py-2">
            <div class="text-white/70 text-xs">Avg Block Time</div>
            <div class="text-white font-bold text-lg">{{ avgBlockTime }}s</div>
          </div>
        </div>
      </div>
    </div>

    <!-- Tab Navigation -->
    <div class="bg-white dark:bg-gray-800 rounded-2xl shadow-sm border border-gray-200 dark:border-gray-700 p-2">
      <div class="flex flex-wrap gap-2">
        <button
          @click="tab = 'blocks'"
          class="flex items-center gap-2 px-4 py-2.5 rounded-xl text-sm font-medium transition-all"
          :class="tab === 'blocks' 
            ? 'bg-gradient-to-r from-indigo-500 to-purple-500 text-white shadow-md' 
            : 'text-gray-600 dark:text-gray-400 hover:bg-gray-100 dark:hover:bg-gray-700'"
        >
          <Icon icon="mdi:view-grid" class="text-lg" />
          {{ $t('block.recent') }}
        </button>
        <RouterLink
          :to="`/${chain}/block/${Number(base.latest?.block?.header.height || 0) + 10000}`"
          class="flex items-center gap-2 px-4 py-2.5 rounded-xl text-sm font-medium text-gray-600 dark:text-gray-400 hover:bg-gray-100 dark:hover:bg-gray-700 transition-all"
        >
          <Icon icon="mdi:clock-fast" class="text-lg" />
          {{ $t('block.future') }}
        </RouterLink>
      </div>
    </div>

    <!-- Content -->
    <div v-show="tab === 'blocks'" class="space-y-6">
      <!-- Chart Section -->
      <div class="bg-white dark:bg-gray-800 rounded-2xl shadow-sm border border-gray-200 dark:border-gray-700 p-6">
        <div class="flex items-center justify-between mb-4">
          <div class="flex items-center gap-3">
            <div class="w-10 h-10 rounded-xl bg-gradient-to-r from-indigo-500 to-purple-500 flex items-center justify-center">
              <Icon icon="mdi:chart-bar" class="text-xl text-white" />
            </div>
            <div>
              <h3 class="font-bold text-gray-800 dark:text-white">Transaction Activity</h3>
              <p class="text-sm text-gray-500">Transactions per block (last 50 blocks)</p>
            </div>
          </div>
        </div>
        <TxsInBlocksChart />
      </div>

      <!-- Blocks Grid -->
      <div class="bg-white dark:bg-gray-800 rounded-2xl shadow-sm border border-gray-200 dark:border-gray-700 p-6">
        <div class="flex items-center gap-3 mb-6">
          <div class="w-10 h-10 rounded-xl bg-gradient-to-r from-purple-500 to-pink-500 flex items-center justify-center">
            <Icon icon="mdi:cube-scan" class="text-xl text-white" />
          </div>
          <div>
            <h3 class="font-bold text-gray-800 dark:text-white">Recent Blocks</h3>
            <p class="text-sm text-gray-500">{{ list.length }} blocks loaded</p>
          </div>
        </div>

        <div class="grid xl:grid-cols-6 lg:grid-cols-4 md:grid-cols-3 sm:grid-cols-2 grid-cols-1 gap-4">
          <RouterLink
            v-for="(item, index) in list"
            :key="item.block?.header?.height"
            class="group relative bg-gradient-to-br from-gray-50 to-gray-100 dark:from-gray-700 dark:to-gray-800 rounded-xl p-4 hover:shadow-lg hover:scale-105 transition-all duration-300 border border-gray-200 dark:border-gray-600 overflow-hidden"
            :to="`/${chain}/block/${item.block.header.height}`"
          >
            <!-- New Block Indicator -->
            <div 
              v-if="index === 0" 
              class="absolute top-2 right-2 w-2 h-2 rounded-full bg-green-500 animate-pulse"
            ></div>
            
            <!-- Block Number -->
            <div class="flex items-center justify-between mb-3">
              <div class="flex items-center gap-2">
                <div class="w-8 h-8 rounded-lg bg-gradient-to-r from-indigo-500 to-purple-500 flex items-center justify-center group-hover:scale-110 transition-transform">
                  <Icon icon="mdi:cube" class="text-white text-sm" />
                </div>
                <span class="font-bold text-gray-800 dark:text-white text-lg">
                  {{ Number(item.block.header.height).toLocaleString() }}
                </span>
              </div>
            </div>

            <!-- Time -->
            <div class="flex items-center gap-1 text-xs text-green-600 dark:text-green-400 mb-2">
              <Icon icon="mdi:clock-outline" class="text-sm" />
              {{ format.toDay(item.block?.header?.time, 'from') }}
            </div>

            <!-- Proposer -->
            <div class="text-xs text-gray-500 dark:text-gray-400 truncate mb-2" :title="format.validator(item.block?.header?.proposer_address)">
              <Icon icon="mdi:account" class="inline text-sm" />
              {{ format.validator(item.block?.header?.proposer_address) }}
            </div>

            <!-- Tx Count -->
            <div class="flex items-center justify-between">
              <div 
                class="inline-flex items-center gap-1 px-2 py-1 rounded-lg text-xs font-medium"
                :class="item.block?.data?.txs?.length > 0 
                  ? 'bg-blue-100 dark:bg-blue-900/30 text-blue-600 dark:text-blue-400' 
                  : 'bg-gray-100 dark:bg-gray-600 text-gray-500 dark:text-gray-400'"
              >
                <Icon icon="mdi:swap-horizontal" class="text-sm" />
                {{ item.block?.data?.txs?.length || 0 }} txs
              </div>
              <Icon icon="mdi:chevron-right" class="text-gray-400 group-hover:text-primary group-hover:translate-x-1 transition-all" />
            </div>

            <!-- Hover Gradient Overlay -->
            <div class="absolute inset-0 bg-gradient-to-r from-indigo-500/0 via-purple-500/0 to-pink-500/0 group-hover:from-indigo-500/5 group-hover:via-purple-500/5 group-hover:to-pink-500/5 transition-all duration-300 pointer-events-none rounded-xl"></div>
          </RouterLink>
        </div>

        <div v-if="list.length === 0" class="text-center py-12">
          <Icon icon="mdi:cube-off-outline" class="text-5xl text-gray-300 dark:text-gray-600 mb-3" />
          <p class="text-gray-500">Loading blocks...</p>
        </div>
      </div>
    </div>
  </div>
</template>

<route>
  {
    meta: {
      i18n: 'blocks',
      order: 5
    }
  }
</route>
