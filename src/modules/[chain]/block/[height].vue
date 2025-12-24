<script lang="ts" setup>
import { onMounted, ref, watchEffect } from 'vue';
import { Icon } from '@iconify/vue';
import TxsElement from '@/components/dynamic/TxsElement.vue';
import DynamicComponent from '@/components/dynamic/DynamicComponent.vue';
import { computed } from '@vue/reactivity';
import { onBeforeRouteUpdate } from 'vue-router';
import { useBaseStore, useFormatter } from '@/stores';
import type { Block } from '@/types';
import Countdown from '@/components/Countdown.vue';

const props = defineProps(['height', 'chain']);

const store = useBaseStore();
const format = useFormatter();
const current = ref({} as Block);
const target = ref(Number(props.height || 0));

const height = computed(() => {
  return Number(current.value.block?.header?.height || props.height || 0);
});

const isFutureBlock = computed({
  get: () => {
    const latest = store.latest?.block?.header.height;
    const isFuture = latest ? target.value > Number(latest) : true;
    if (!isFuture && !current.value.block_id) store.fetchBlock(target.value).then((x) => (current.value = x));
    return isFuture;
  },
  set: (val) => {
    console.log(val);
  },
});

const remainingBlocks = computed(() => {
  const latest = store.latest?.block?.header.height;
  return latest ? Number(target.value) - Number(latest) : 0;
});

const estimateTime = computed(() => {
  const seconds = Number((remainingBlocks.value * store.blocktime).toFixed(2));
  return seconds;
});

const estimateDate = computed(() => {
  return new Date(new Date().getTime() + estimateTime.value);
});

const progress = computed(() => {
  const latest = Number(store.latest?.block?.header.height || 0);
  if (target.value <= latest) return 100;
  const start = latest - remainingBlocks.value;
  return Math.max(0, Math.min(100, ((latest - start) / (target.value - start)) * 100));
});

const edit = ref(false);
const newHeight = ref(props.height);

function updateTarget() {
  target.value = Number(newHeight.value);
  edit.value = false;
}

onBeforeRouteUpdate(async (to, from, next) => {
  if (from.path !== to.path) {
    store.fetchBlock(String(to.params.height)).then((x) => (current.value = x));
    next();
  }
});
</script>

<template>
  <div class="space-y-6">
    <!-- Future Block View -->
    <div v-if="isFutureBlock">
      <div v-if="remainingBlocks > 0" class="space-y-6">
        <!-- Hero Section -->
        <div class="relative overflow-hidden rounded-2xl bg-gradient-to-r from-indigo-600 via-purple-600 to-pink-600 p-8 md:p-12">
          <div class="absolute inset-0 bg-black/20"></div>
          
          <!-- Animated Background Elements -->
          <div class="absolute top-0 left-0 w-full h-full overflow-hidden">
            <div class="absolute -top-40 -right-40 w-80 h-80 bg-white/10 rounded-full blur-3xl animate-pulse"></div>
            <div class="absolute -bottom-40 -left-40 w-80 h-80 bg-white/10 rounded-full blur-3xl animate-pulse" style="animation-delay: 1s;"></div>
            <div class="absolute top-1/2 left-1/2 -translate-x-1/2 -translate-y-1/2 w-96 h-96 bg-white/5 rounded-full blur-3xl animate-pulse" style="animation-delay: 0.5s;"></div>
          </div>

          <div class="relative z-10 text-center">
            <!-- Target Block Badge -->
            <div class="inline-flex items-center gap-2 px-4 py-2 bg-white/10 backdrop-blur rounded-full mb-6">
              <Icon icon="mdi:cube-outline" class="text-white" />
              <span class="text-white font-medium">{{ $t('block.future') }}</span>
            </div>

            <!-- Target Block Number -->
            <div class="mb-8">
              <div class="text-white/60 text-sm uppercase tracking-wider mb-2">Target Block</div>
              <div class="text-4xl md:text-6xl font-bold text-white">#{{ target.toLocaleString() }}</div>
            </div>

            <!-- Countdown Timer -->
            <div class="mb-8">
              <Countdown :time="estimateTime" />
            </div>

            <!-- Estimated Arrival -->
            <div class="inline-flex items-center gap-2 px-6 py-3 bg-white/10 backdrop-blur rounded-xl">
              <Icon icon="mdi:calendar-clock" class="text-xl text-white/80" />
              <div class="text-left">
                <div class="text-white/60 text-xs">{{ $t('block.estimated_time') }}</div>
                <div class="text-white font-bold">{{ format.toLocaleDate(estimateDate) }}</div>
              </div>
            </div>
          </div>
        </div>

        <!-- Stats Cards -->
        <div class="grid md:grid-cols-3 gap-4">
          <!-- Current Height -->
          <div class="bg-white dark:bg-gray-800 rounded-2xl shadow-sm border border-gray-200 dark:border-gray-700 p-6">
            <div class="flex items-center gap-3 mb-3">
              <div class="w-10 h-10 rounded-xl bg-gradient-to-r from-cyan-500 to-blue-500 flex items-center justify-center">
                <Icon icon="mdi:cube" class="text-xl text-white" />
              </div>
              <span class="text-sm text-gray-500 dark:text-gray-400">{{ $t('block.current_height') }}</span>
            </div>
            <div class="text-2xl font-bold text-gray-800 dark:text-white">
              #{{ Number(store.latest?.block?.header.height || 0).toLocaleString() }}
            </div>
          </div>

          <!-- Remaining Blocks -->
          <div class="bg-white dark:bg-gray-800 rounded-2xl shadow-sm border border-gray-200 dark:border-gray-700 p-6">
            <div class="flex items-center gap-3 mb-3">
              <div class="w-10 h-10 rounded-xl bg-gradient-to-r from-emerald-500 to-green-500 flex items-center justify-center">
                <Icon icon="mdi:counter" class="text-xl text-white" />
              </div>
              <span class="text-sm text-gray-500 dark:text-gray-400">{{ $t('block.remaining_blocks') }}</span>
            </div>
            <div class="text-2xl font-bold text-gray-800 dark:text-white">
              {{ remainingBlocks.toLocaleString() }}
            </div>
          </div>

          <!-- Block Time -->
          <div class="bg-white dark:bg-gray-800 rounded-2xl shadow-sm border border-gray-200 dark:border-gray-700 p-6">
            <div class="flex items-center gap-3 mb-3">
              <div class="w-10 h-10 rounded-xl bg-gradient-to-r from-amber-500 to-orange-500 flex items-center justify-center">
                <Icon icon="mdi:timer-outline" class="text-xl text-white" />
              </div>
              <span class="text-sm text-gray-500 dark:text-gray-400">{{ $t('block.average_block_time') }}</span>
            </div>
            <div class="text-2xl font-bold text-gray-800 dark:text-white">
              {{ (store.blocktime / 1000).toFixed(1) }}s
            </div>
          </div>
        </div>

        <!-- Change Target Block -->
        <div class="bg-white dark:bg-gray-800 rounded-2xl shadow-sm border border-gray-200 dark:border-gray-700 p-6">
          <div class="flex items-center justify-between mb-4">
            <div class="flex items-center gap-3">
              <div class="w-10 h-10 rounded-xl bg-gradient-to-r from-purple-500 to-pink-500 flex items-center justify-center">
                <Icon icon="mdi:target" class="text-xl text-white" />
              </div>
              <div>
                <h3 class="font-bold text-gray-800 dark:text-white">{{ $t('block.countdown_for_block') }}</h3>
                <p class="text-sm text-gray-500">Change the target block number</p>
              </div>
            </div>
            <button 
              @click="edit = !edit"
              class="flex items-center gap-2 px-4 py-2 rounded-xl text-sm font-medium transition-all"
              :class="edit 
                ? 'bg-gray-100 dark:bg-gray-700 text-gray-600 dark:text-gray-300' 
                : 'bg-gradient-to-r from-purple-500 to-pink-500 text-white'"
            >
              <Icon :icon="edit ? 'mdi:close' : 'mdi:pencil'" />
              {{ edit ? 'Cancel' : 'Change' }}
            </button>
          </div>

          <div v-if="edit" class="mt-4 p-4 bg-gray-50 dark:bg-gray-700/50 rounded-xl">
            <label class="block text-sm font-medium text-gray-700 dark:text-gray-300 mb-2">
              {{ $t('block.countdown_for_block_input') }}
            </label>
            <div class="flex gap-3">
              <input 
                v-model="newHeight" 
                type="number" 
                class="flex-1 px-4 py-3 bg-white dark:bg-gray-800 border border-gray-200 dark:border-gray-600 rounded-xl text-gray-800 dark:text-white focus:ring-2 focus:ring-purple-500/50 focus:border-purple-500 transition-all"
                :placeholder="target.toString()"
              />
              <button 
                @click="updateTarget"
                class="px-6 py-3 bg-gradient-to-r from-purple-500 to-pink-500 text-white rounded-xl font-medium hover:shadow-lg hover:shadow-purple-500/25 transition-all"
              >
                {{ $t('block.btn_update') }}
              </button>
            </div>
          </div>
        </div>
      </div>
    </div>

    <!-- Block Detail View -->
    <div v-else class="space-y-6">
      <!-- Header -->
      <div class="bg-white dark:bg-gray-800 rounded-2xl shadow-sm border border-gray-200 dark:border-gray-700 p-6">
        <div class="flex items-center justify-between">
          <div class="flex items-center gap-4">
            <div class="w-14 h-14 rounded-xl bg-gradient-to-r from-indigo-500 to-purple-500 flex items-center justify-center">
              <Icon icon="mdi:cube" class="text-2xl text-white" />
            </div>
            <div>
              <div class="text-sm text-gray-500 dark:text-gray-400">Block</div>
              <h2 class="text-2xl font-bold text-gray-800 dark:text-white">#{{ current.block?.header?.height }}</h2>
            </div>
          </div>
          <div class="flex gap-2" v-if="props.height">
            <RouterLink
              :to="`/${store.blockchain.chainName}/block/${height - 1}`"
              class="w-10 h-10 rounded-xl bg-gray-100 dark:bg-gray-700 flex items-center justify-center hover:bg-gray-200 dark:hover:bg-gray-600 transition-colors"
            >
              <Icon icon="mdi:arrow-left" class="text-xl text-gray-600 dark:text-gray-300" />
            </RouterLink>
            <RouterLink
              :to="`/${store.blockchain.chainName}/block/${height + 1}`"
              class="w-10 h-10 rounded-xl bg-gray-100 dark:bg-gray-700 flex items-center justify-center hover:bg-gray-200 dark:hover:bg-gray-600 transition-colors"
            >
              <Icon icon="mdi:arrow-right" class="text-xl text-gray-600 dark:text-gray-300" />
            </RouterLink>
          </div>
        </div>
        <div class="mt-4">
          <DynamicComponent :value="current.block_id" />
        </div>
      </div>

      <!-- Block Header -->
      <div class="bg-white dark:bg-gray-800 rounded-2xl shadow-sm border border-gray-200 dark:border-gray-700 p-6">
        <div class="flex items-center gap-3 mb-4">
          <div class="w-10 h-10 rounded-xl bg-gradient-to-r from-cyan-500 to-blue-500 flex items-center justify-center">
            <Icon icon="mdi:file-document" class="text-xl text-white" />
          </div>
          <h2 class="text-lg font-bold text-gray-800 dark:text-white">{{ $t('block.block_header') }}</h2>
        </div>
        <DynamicComponent :value="current.block?.header" />
      </div>

      <!-- Transactions -->
      <div class="bg-white dark:bg-gray-800 rounded-2xl shadow-sm border border-gray-200 dark:border-gray-700 p-6">
        <div class="flex items-center gap-3 mb-4">
          <div class="w-10 h-10 rounded-xl bg-gradient-to-r from-emerald-500 to-green-500 flex items-center justify-center">
            <Icon icon="mdi:swap-horizontal" class="text-xl text-white" />
          </div>
          <h2 class="text-lg font-bold text-gray-800 dark:text-white">{{ $t('account.transactions') }}</h2>
        </div>
        <TxsElement :value="current.block?.data?.txs" />
      </div>

      <!-- Last Commit -->
      <div class="bg-white dark:bg-gray-800 rounded-2xl shadow-sm border border-gray-200 dark:border-gray-700 p-6">
        <div class="flex items-center gap-3 mb-4">
          <div class="w-10 h-10 rounded-xl bg-gradient-to-r from-amber-500 to-orange-500 flex items-center justify-center">
            <Icon icon="mdi:check-all" class="text-xl text-white" />
          </div>
          <h2 class="text-lg font-bold text-gray-800 dark:text-white">{{ $t('block.last_commit') }}</h2>
        </div>
        <DynamicComponent :value="current.block?.last_commit" />
      </div>
    </div>
  </div>
</template>
