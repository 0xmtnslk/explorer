<script lang="ts" setup>
import { computed, ref } from '@vue/reactivity';
import { useBaseStore, useBlockchain, useFormatter } from '@/stores';
import type { PaginatedTxs } from '@/types';
import { useRouter } from 'vue-router';
import { onMounted } from 'vue';
import { Icon } from '@iconify/vue';

const props = defineProps(['chain']);
const vueRouters = useRouter();
const tab = ref('recent');

const base = useBaseStore();
const chainStore = useBlockchain();

const format = useFormatter();
const hashReg = /^[A-Z\d]{64}$/;
const hash = ref('');
const current = chainStore?.current?.chainName || '';
const searchError = ref('');

onMounted(() => {
  tab.value = String(vueRouters.currentRoute.value.query.tab || 'recent');
});

function search() {
  searchError.value = '';
  if (hash.value.length === 0) return;
  
  if (hashReg.test(hash.value.toUpperCase())) {
    vueRouters.push({ path: `/${current}/tx/${hash.value.toUpperCase()}` });
  } else {
    searchError.value = 'Invalid transaction hash. Must be 64 hexadecimal characters.';
  }
}

function handleKeydown(e: KeyboardEvent) {
  if (e.key === 'Enter') {
    search();
  }
}

const totalTxs = computed(() => base.txsInRecents?.length || 0);
</script>

<template>
  <div class="space-y-6">
    <!-- Hero Section -->
    <div class="relative overflow-hidden rounded-2xl bg-gradient-to-r from-rose-500 via-pink-500 to-purple-600 p-6 md:p-8">
      <div class="absolute inset-0 bg-black/10"></div>
      <div class="absolute -top-24 -right-24 w-64 h-64 bg-white/10 rounded-full blur-3xl"></div>
      <div class="absolute -bottom-24 -left-24 w-64 h-64 bg-white/10 rounded-full blur-3xl"></div>
      
      <div class="relative z-10">
        <div class="flex items-center gap-3 mb-4">
          <div class="w-12 h-12 rounded-xl bg-white/20 backdrop-blur flex items-center justify-center">
            <Icon icon="mdi:swap-horizontal-bold" class="text-2xl text-white" />
          </div>
          <div>
            <h1 class="text-2xl md:text-3xl font-bold text-white">Transactions</h1>
            <p class="text-white/80 text-sm">Browse and search blockchain transactions</p>
          </div>
        </div>
        
        <!-- Stats -->
        <div class="flex flex-wrap gap-4">
          <div class="bg-white/10 backdrop-blur rounded-xl px-4 py-2">
            <div class="text-white/70 text-xs">Recent Transactions</div>
            <div class="text-white font-bold text-lg">{{ totalTxs }}</div>
          </div>
          <div class="bg-white/10 backdrop-blur rounded-xl px-4 py-2">
            <div class="text-white/70 text-xs">Latest Block</div>
            <div class="text-white font-bold text-lg">#{{ base.latest?.block?.header?.height?.toLocaleString() || '-' }}</div>
          </div>
        </div>
      </div>
    </div>

    <!-- Tab Navigation -->
    <div class="bg-white dark:bg-gray-800 rounded-2xl shadow-sm border border-gray-200 dark:border-gray-700 p-2">
      <div class="flex flex-wrap gap-2">
        <button
          @click="tab = 'recent'"
          class="flex items-center gap-2 px-4 py-2.5 rounded-xl text-sm font-medium transition-all"
          :class="tab === 'recent' 
            ? 'bg-gradient-to-r from-rose-500 to-pink-500 text-white shadow-md' 
            : 'text-gray-600 dark:text-gray-400 hover:bg-gray-100 dark:hover:bg-gray-700'"
        >
          <Icon icon="mdi:history" class="text-lg" />
          {{ $t('block.recent') }}
        </button>
        <button
          @click="tab = 'search'"
          class="flex items-center gap-2 px-4 py-2.5 rounded-xl text-sm font-medium transition-all"
          :class="tab === 'search' 
            ? 'bg-gradient-to-r from-rose-500 to-pink-500 text-white shadow-md' 
            : 'text-gray-600 dark:text-gray-400 hover:bg-gray-100 dark:hover:bg-gray-700'"
        >
          <Icon icon="mdi:magnify" class="text-lg" />
          Search
        </button>
      </div>
    </div>

    <!-- Recent Transactions -->
    <div v-show="tab === 'recent'" class="space-y-4">
      <div class="bg-white dark:bg-gray-800 rounded-2xl shadow-sm border border-gray-200 dark:border-gray-700 overflow-hidden">
        <div class="overflow-x-auto">
          <table class="w-full">
            <thead>
              <tr class="bg-gradient-to-r from-gray-50 to-gray-100 dark:from-gray-700 dark:to-gray-800">
                <th class="text-left px-6 py-4 text-xs font-semibold text-gray-500 dark:text-gray-400 uppercase tracking-wider">
                  {{ $t('account.height') }}
                </th>
                <th class="text-left px-6 py-4 text-xs font-semibold text-gray-500 dark:text-gray-400 uppercase tracking-wider">
                  {{ $t('account.hash') }}
                </th>
                <th class="text-left px-6 py-4 text-xs font-semibold text-gray-500 dark:text-gray-400 uppercase tracking-wider">
                  {{ $t('account.messages') }}
                </th>
                <th class="text-right px-6 py-4 text-xs font-semibold text-gray-500 dark:text-gray-400 uppercase tracking-wider">
                  {{ $t('block.fees') }}
                </th>
              </tr>
            </thead>
            <tbody class="divide-y divide-gray-100 dark:divide-gray-700">
              <tr 
                v-for="(item, index) in base.txsInRecents" 
                :key="item.hash"
                class="hover:bg-gray-50 dark:hover:bg-gray-700/50 transition-colors group"
              >
                <td class="px-6 py-4">
                  <RouterLink 
                    :to="`/${props.chain}/block/${item.height}`"
                    class="inline-flex items-center gap-2 px-3 py-1 bg-indigo-50 dark:bg-indigo-900/30 text-indigo-600 dark:text-indigo-400 rounded-lg text-sm font-medium hover:bg-indigo-100 dark:hover:bg-indigo-900/50 transition-colors"
                  >
                    <Icon icon="mdi:cube" class="text-sm" />
                    {{ item.height }}
                  </RouterLink>
                </td>
                <td class="px-6 py-4">
                  <RouterLink 
                    :to="`/${props.chain}/tx/${item.hash}`"
                    class="flex items-center gap-2 group/hash"
                  >
                    <div class="w-8 h-8 rounded-lg bg-gradient-to-r from-rose-500 to-pink-500 flex items-center justify-center flex-shrink-0">
                      <Icon icon="mdi:swap-horizontal" class="text-white text-sm" />
                    </div>
                    <span class="font-mono text-sm text-gray-600 dark:text-gray-300 truncate max-w-[200px] md:max-w-[300px] lg:max-w-[400px] group-hover/hash:text-primary transition-colors">
                      {{ item.hash }}
                    </span>
                    <Icon icon="mdi:arrow-right" class="text-gray-400 opacity-0 group-hover/hash:opacity-100 group-hover/hash:translate-x-1 transition-all flex-shrink-0" />
                  </RouterLink>
                </td>
                <td class="px-6 py-4">
                  <div class="flex flex-wrap gap-1">
                    <span 
                      v-for="(msg, msgIndex) in format.messages(item.tx.body.messages).split(',')" 
                      :key="msgIndex"
                      class="inline-flex items-center px-2 py-0.5 rounded-md text-xs font-medium bg-gray-100 dark:bg-gray-700 text-gray-600 dark:text-gray-300"
                    >
                      {{ msg.trim() }}
                    </span>
                  </div>
                </td>
                <td class="px-6 py-4 text-right">
                  <span class="text-sm font-medium text-gray-700 dark:text-gray-300">
                    {{ format.formatTokens(item.tx.authInfo.fee?.amount) }}
                  </span>
                </td>
              </tr>
            </tbody>
          </table>
        </div>

        <div v-if="base.txsInRecents?.length === 0" class="text-center py-12">
          <Icon icon="mdi:swap-horizontal-circle-outline" class="text-5xl text-gray-300 dark:text-gray-600 mb-3" />
          <p class="text-gray-500">No transactions found in recent blocks</p>
        </div>
      </div>

      <!-- Info Alert -->
      <div class="bg-gradient-to-r from-blue-50 to-cyan-50 dark:from-blue-900/20 dark:to-cyan-900/20 rounded-2xl border border-blue-200 dark:border-blue-800 p-4">
        <div class="flex items-start gap-3">
          <div class="w-10 h-10 rounded-xl bg-gradient-to-r from-blue-500 to-cyan-500 flex items-center justify-center flex-shrink-0">
            <Icon icon="mdi:information" class="text-xl text-white" />
          </div>
          <div>
            <h4 class="font-medium text-blue-800 dark:text-blue-300 mb-1">Recent Blocks Only</h4>
            <p class="text-sm text-blue-600 dark:text-blue-400">{{ $t('block.only_tx') }}</p>
          </div>
        </div>
      </div>
    </div>

    <!-- Search Tab -->
    <div v-show="tab === 'search'" class="bg-white dark:bg-gray-800 rounded-2xl shadow-sm border border-gray-200 dark:border-gray-700 p-6">
      <div class="max-w-2xl mx-auto">
        <!-- Search Header -->
        <div class="text-center mb-8">
          <div class="w-16 h-16 mx-auto mb-4 rounded-2xl bg-gradient-to-r from-rose-500 to-pink-500 flex items-center justify-center">
            <Icon icon="mdi:magnify" class="text-3xl text-white" />
          </div>
          <h2 class="text-xl font-bold text-gray-800 dark:text-white mb-2">Search Transaction</h2>
          <p class="text-gray-500 dark:text-gray-400">Enter a transaction hash to view its details</p>
        </div>

        <!-- Search Input -->
        <div class="space-y-4">
          <div class="relative">
            <div class="absolute left-4 top-1/2 -translate-y-1/2 text-gray-400">
              <Icon icon="mdi:file-search" class="text-xl" />
            </div>
            <input
              v-model="hash"
              type="text"
              class="w-full pl-12 pr-4 py-4 bg-gray-100 dark:bg-gray-700 border-2 border-transparent rounded-xl text-gray-800 dark:text-white placeholder-gray-400 focus:border-pink-500 focus:bg-white dark:focus:bg-gray-800 focus:ring-0 transition-all font-mono text-sm"
              placeholder="Enter transaction hash (64 characters)"
              @keydown="handleKeydown"
            />
          </div>

          <button 
            @click="search"
            class="w-full py-4 bg-gradient-to-r from-rose-500 to-pink-500 text-white rounded-xl font-medium hover:shadow-lg hover:shadow-pink-500/25 transition-all flex items-center justify-center gap-2"
          >
            <Icon icon="mdi:magnify" class="text-xl" />
            Search Transaction
          </button>

          <!-- Error Message -->
          <div v-if="searchError" class="flex items-center gap-2 p-4 bg-red-50 dark:bg-red-900/20 border border-red-200 dark:border-red-800 rounded-xl">
            <Icon icon="mdi:alert-circle" class="text-xl text-red-500" />
            <span class="text-sm text-red-600 dark:text-red-400">{{ searchError }}</span>
          </div>
        </div>

        <!-- Help Text -->
        <div class="mt-8 p-4 bg-gray-50 dark:bg-gray-700/50 rounded-xl">
          <h4 class="text-sm font-medium text-gray-700 dark:text-gray-300 mb-2 flex items-center gap-2">
            <Icon icon="mdi:lightbulb" class="text-amber-500" />
            Tips
          </h4>
          <ul class="text-sm text-gray-500 dark:text-gray-400 space-y-1">
            <li class="flex items-center gap-2">
              <Icon icon="mdi:check" class="text-green-500 text-sm" />
              Transaction hash is 64 hexadecimal characters
            </li>
            <li class="flex items-center gap-2">
              <Icon icon="mdi:check" class="text-green-500 text-sm" />
              Example: 3A7F2B...C9E1D4
            </li>
            <li class="flex items-center gap-2">
              <Icon icon="mdi:check" class="text-green-500 text-sm" />
              Press Enter or click Search to find
            </li>
          </ul>
        </div>
      </div>
    </div>
  </div>
</template>

<route>
  {
    meta: {
      i18n: 'tx',
      order: 5
    }
  }
</route>
