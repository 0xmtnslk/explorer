<script lang="ts" setup>
import { useDashboard } from '@/stores';
import { computed } from 'vue';
import { Icon } from '@iconify/vue';

const props = defineProps({
  name: {
    type: String,
    required: true,
  },
});

const dashboardStore = useDashboard();
const conf = computed(() => dashboardStore.chains[props.name] || {});

const price = computed(() => {
  const asset = conf.value?.assets?.[0];
  if (asset?.coingecko_id && dashboardStore.prices[asset.coingecko_id]) {
    return dashboardStore.prices[asset.coingecko_id].usd;
  }
  return null;
});

const priceChange = computed(() => {
  const asset = conf.value?.assets?.[0];
  if (asset?.coingecko_id && dashboardStore.prices[asset.coingecko_id]) {
    return dashboardStore.prices[asset.coingecko_id].usd_24h_change;
  }
  return null;
});

const formatPrice = (val: number) => {
  if (val >= 1) {
    return val.toFixed(2);
  } else if (val >= 0.01) {
    return val.toFixed(4);
  } else if (val >= 0.0001) {
    return val.toFixed(6);
  } else {
    return val.toFixed(8);
  }
};

const addFavor = (e: Event) => {
  e.stopPropagation();
  e.preventDefault();
  dashboardStore.favoriteMap[props.name] = !dashboardStore?.favoriteMap?.[props.name];
  window.localStorage.setItem('favoriteMap', JSON.stringify(dashboardStore.favoriteMap));
};
</script>

<template>
  <RouterLink
    :to="`/${name}`"
    class="group relative bg-white dark:bg-gray-800/50 hover:bg-gray-50 dark:hover:bg-gray-700/50 rounded-2xl border border-gray-200 dark:border-gray-700/50 p-4 cursor-pointer transition-all hover:shadow-lg hover:shadow-primary/5 hover:border-primary/30"
  >
    <!-- Favorite Star -->
    <button
      @click="addFavor"
      class="absolute top-3 right-3 p-1.5 rounded-lg hover:bg-gray-100 dark:hover:bg-gray-700 transition-colors"
      :class="{
        'text-yellow-500': dashboardStore?.favoriteMap?.[props.name],
        'text-gray-300 dark:text-gray-600 hover:text-yellow-400': !dashboardStore?.favoriteMap?.[props.name],
      }"
    >
      <Icon :icon="dashboardStore?.favoriteMap?.[props.name] ? 'mdi:star' : 'mdi:star-outline'" class="text-lg" />
    </button>

    <div class="flex items-center gap-4">
      <!-- Logo -->
      <div class="relative">
        <div class="w-12 h-12 rounded-xl overflow-hidden bg-gray-100 dark:bg-gray-700 ring-2 ring-gray-100 dark:ring-gray-700 group-hover:ring-primary/20 transition-all">
          <img 
            :src="conf.logo" 
            :alt="conf?.prettyName"
            class="w-full h-full object-cover"
            loading="lazy"
          />
        </div>
        <div 
          class="absolute -bottom-1 -right-1 w-4 h-4 rounded-full border-2 border-white dark:border-gray-800"
          :class="conf.networkType === 'testnet' ? 'bg-orange-500' : 'bg-emerald-500'"
        ></div>
      </div>

      <!-- Info -->
      <div class="flex-1 min-w-0">
        <h3 class="font-semibold text-gray-900 dark:text-white truncate capitalize group-hover:text-primary transition-colors">
          {{ conf?.prettyName || props.name }}
        </h3>
        <div class="flex items-center gap-2 mt-1">
          <span v-if="price" class="text-sm font-medium text-gray-600 dark:text-gray-300">
            ${{ formatPrice(price) }}
          </span>
          <span 
            v-if="priceChange !== null" 
            class="text-xs font-medium px-1.5 py-0.5 rounded"
            :class="priceChange >= 0 ? 'text-emerald-600 bg-emerald-500/10' : 'text-red-600 bg-red-500/10'"
          >
            {{ priceChange >= 0 ? '+' : '' }}{{ priceChange?.toFixed(1) }}%
          </span>
          <span v-if="!price" class="text-xs text-gray-400 dark:text-gray-500">
            {{ conf?.assets?.[0]?.symbol || '—' }}
          </span>
        </div>
      </div>
    </div>

    <!-- Hover Arrow -->
    <div class="absolute bottom-4 right-4 opacity-0 group-hover:opacity-100 transition-opacity">
      <Icon icon="mdi:arrow-right" class="text-primary text-lg" />
    </div>
  </RouterLink>
</template>
