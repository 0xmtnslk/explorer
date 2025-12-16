<script lang="ts" setup>
import MdEditor from 'md-editor-v3';
import PriceMarketChart from '@/components/charts/PriceMarketChart.vue';

import { Icon } from '@iconify/vue';
import { useBlockchain, useFormatter, useTxDialog, useWalletStore, useStakingStore, useParamStore, useBaseStore } from '@/stores';
import { onMounted, ref } from 'vue';
import { useIndexModule, colorMap, tickerUrl } from './indexStore';
import { computed } from '@vue/reactivity';

import CardStatisticsVertical from '@/components/CardStatisticsVertical.vue';
import ProposalListItem from '@/components/ProposalListItem.vue';
import ArrayObjectElement from '@/components/dynamic/ArrayObjectElement.vue';

const props = defineProps(['chain']);

const blockchain = useBlockchain();
const store = useIndexModule();
const walletStore = useWalletStore();
const format = useFormatter();
const dialog = useTxDialog();
const stakingStore = useStakingStore();
const paramStore = useParamStore();
const baseStore = useBaseStore();
const coinInfo = computed(() => {
  return store.coinInfo;
});

onMounted(() => {
  store.loadDashboard();
  walletStore.loadMyAsset();
  paramStore.handleAbciInfo();
  // if(!(coinInfo.value && coinInfo.value.name)) {
  // }
});
const ticker = computed(() => store.coinInfo.tickers[store.tickerIndex]);

const currName = ref('');
blockchain.$subscribe((m, s) => {
  if (s.chainName !== currName.value) {
    currName.value = s.chainName;
    store.loadDashboard();
    walletStore.loadMyAsset();
    paramStore.handleAbciInfo();
  }
});
function shortName(name: string, id: string) {
  return name.toLowerCase().startsWith('ibc/') || name.toLowerCase().startsWith('0x') ? id : name;
}

const comLinks = computed(() => {
  return [
    {
      name: 'Website',
      icon: 'mdi-web',
      href: store.homepage,
    },
    {
      name: 'Twitter',
      icon: 'mdi-twitter',
      href: store.twitter,
    },
    {
      name: 'Telegram',
      icon: 'mdi-telegram',
      href: store.telegram,
    },
    {
      name: 'Github',
      icon: 'mdi-github',
      href: store.github,
    },
  ];
});

// wallet box
const change = computed(() => {
  const token = walletStore.balanceOfStakingToken;
  return token ? format.priceChanges(token.denom) : 0;
});
const color = computed(() => {
  switch (true) {
    case change.value > 0:
      return 'text-green-600';
    case change.value === 0:
      return 'text-grey-500';
    case change.value < 0:
      return 'text-red-600';
  }
});

function updateState() {
  walletStore.loadMyAsset();
}

function trustColor(v: string) {
  return `text-${colorMap(v)}`;
}

const quantity = ref(100);
const qty = computed({
  get: () => {
    return parseFloat(quantity.value.toFixed(6));
  },
  set: (val) => {
    quantity.value = val;
  },
});
const amount = computed({
  get: () => {
    return quantity.value * ticker.value.converted_last.usd || 0;
  },
  set: (val) => {
    quantity.value = val / ticker.value.converted_last.usd || 0;
  },
});
</script>

<template>
  <div class="space-y-6">
    <!-- Hero Section with Chain Info - Modern Design -->
    <div v-if="coinInfo && coinInfo.name" class="relative overflow-hidden rounded-3xl">
      <!-- Animated Background - Light/Dark Mode Compatible -->
      <div class="absolute inset-0 bg-gradient-to-br from-gray-50 via-white to-gray-100 dark:from-slate-900 dark:via-slate-800 dark:to-slate-900"></div>
      <div class="absolute inset-0 bg-[radial-gradient(ellipse_at_top_right,_var(--tw-gradient-stops))] from-primary/10 dark:from-primary/20 via-transparent to-transparent"></div>
      <div class="absolute inset-0 bg-[radial-gradient(ellipse_at_bottom_left,_var(--tw-gradient-stops))] from-emerald-500/5 dark:from-emerald-500/10 via-transparent to-transparent"></div>
      <div class="absolute top-0 right-0 w-96 h-96 bg-primary/5 dark:bg-primary/10 rounded-full blur-3xl"></div>
      <div class="absolute bottom-0 left-0 w-72 h-72 bg-emerald-500/5 dark:bg-emerald-500/10 rounded-full blur-3xl"></div>
      <div class="absolute inset-0 border border-gray-200/50 dark:border-white/5 rounded-3xl pointer-events-none"></div>
      
      <div class="relative p-6 lg:p-8">
        <!-- Top Row: Header with Logo, Name, Rank, Social Links -->
        <div class="flex flex-wrap items-start justify-between gap-4 mb-6">
          <!-- Left: Coin Identity -->
          <div class="flex items-center gap-4">
            <div class="relative">
              <div class="w-16 h-16 rounded-2xl bg-gradient-to-br from-gray-200 to-gray-100 dark:from-white/20 dark:to-white/5 backdrop-blur-xl p-0.5 shadow-lg">
                <div class="w-full h-full rounded-2xl bg-white dark:bg-slate-800/80 flex items-center justify-center overflow-hidden">
                  <img v-if="coinInfo.image?.large" :src="coinInfo.image?.large" :alt="coinInfo.name" class="w-12 h-12 object-contain" />
                  <Icon v-else icon="mdi:currency-btc" class="text-3xl text-primary" />
                </div>
              </div>
              <div class="absolute -bottom-1 -right-1 w-5 h-5 rounded-full bg-emerald-500 border-2 border-white dark:border-slate-800 flex items-center justify-center">
                <Icon icon="mdi:check" class="text-white text-xs" />
              </div>
            </div>
            <div>
              <h1 class="text-2xl lg:text-3xl font-bold text-gray-900 dark:text-white flex items-center gap-2">
                {{ coinInfo.name }}
                <span class="text-lg text-gray-500 dark:text-gray-400 font-normal uppercase">{{ coinInfo.symbol }}</span>
              </h1>
              <div class="flex items-center gap-2 mt-1">
                <span class="inline-flex items-center gap-1 px-2.5 py-1 text-xs font-semibold rounded-lg bg-gradient-to-r from-amber-500/20 to-orange-500/20 text-amber-600 dark:text-amber-400 border border-amber-500/30">
                  <Icon icon="mdi:trophy" class="text-sm" />
                  Rank #{{ coinInfo.market_cap_rank }}
                </span>
              </div>
            </div>
          </div>

          <!-- Right: Social Links -->
          <div class="flex items-center gap-2">
            <a
              v-for="(item, index) of comLinks"
              :key="index"
              :href="item.href"
              target="_blank"
              class="group relative p-2.5 rounded-xl bg-gray-100 dark:bg-white/5 hover:bg-gray-200 dark:hover:bg-white/10 border border-gray-200 dark:border-white/10 hover:border-primary/50 transition-all"
              :title="item.name"
            >
              <Icon :icon="item?.icon" class="text-xl text-gray-500 dark:text-gray-400 group-hover:text-primary transition-colors" />
            </a>
          </div>
        </div>

        <!-- Main Content Grid -->
        <div class="grid grid-cols-1 lg:grid-cols-12 gap-6">
          <!-- Left Column: Price & Exchange Info -->
          <div class="lg:col-span-4 space-y-4">
            <!-- Live Price Card -->
            <div class="relative overflow-hidden rounded-2xl bg-white dark:bg-gradient-to-br dark:from-white/10 dark:to-white/5 backdrop-blur-xl border border-gray-200 dark:border-white/10 p-5 shadow-sm">
              <div class="absolute top-0 right-0 w-20 h-20 bg-primary/10 dark:bg-primary/20 rounded-full blur-2xl"></div>
              <div class="relative">
                <div class="flex items-center justify-between mb-1">
                  <span class="text-xs text-gray-500 dark:text-gray-400 uppercase tracking-wide">Live Price</span>
                  <span class="flex items-center gap-1 text-xs text-emerald-500 dark:text-emerald-400">
                    <span class="w-1.5 h-1.5 rounded-full bg-emerald-500 dark:bg-emerald-400 animate-pulse"></span>
                    Live
                  </span>
                </div>
                <div class="flex items-end gap-3">
                  <span class="text-3xl lg:text-4xl font-bold text-gray-900 dark:text-white">${{ ticker?.converted_last?.usd }}</span>
                  <span 
                    class="text-lg font-semibold mb-1 px-2 py-0.5 rounded-lg"
                    :class="store.priceChange >= 0 ? 'text-emerald-600 dark:text-emerald-400 bg-emerald-500/20' : 'text-red-600 dark:text-red-400 bg-red-500/20'"
                  >
                    {{ store.priceChange >= 0 ? '+' : '' }}{{ store.priceChange }}%
                  </span>
                </div>
              </div>
            </div>

            <!-- Exchange Selector -->
            <div class="dropdown dropdown-hover w-full">
              <label tabindex="0">
                <div class="flex items-center justify-between p-4 rounded-2xl bg-white dark:bg-white/5 hover:bg-gray-50 dark:hover:bg-white/10 border border-gray-200 dark:border-white/10 hover:border-primary/30 cursor-pointer transition-all group shadow-sm">
                  <div class="flex items-center gap-3">
                    <div class="p-2 rounded-xl bg-gradient-to-br from-blue-500/20 to-cyan-500/20">
                      <Icon icon="mdi:bank" class="text-xl text-blue-500 dark:text-blue-400" />
                    </div>
                    <div>
                      <div class="font-semibold text-gray-900 dark:text-white">{{ ticker?.market?.name || 'Select Exchange' }}</div>
                      <div class="text-xs text-primary">
                        {{ shortName(ticker?.base, ticker?.coin_id) }}/{{ shortName(ticker?.target, ticker?.target_coin_id) }}
                      </div>
                    </div>
                  </div>
                  <Icon icon="mdi:chevron-down" class="text-xl text-gray-400 group-hover:text-primary transition-colors" />
                </div>
              </label>
              <div class="dropdown-content pt-2 z-20 w-full">
                <div class="max-h-64 overflow-auto w-full shadow-2xl rounded-2xl border border-gray-200 dark:border-gray-700 bg-white dark:bg-slate-800">
                  <ul class="menu w-full p-2">
                    <li v-for="(item, index) in store.coinInfo.tickers" :key="index" @click="store.selectTicker(index)">
                      <div class="flex items-center justify-between hover:bg-gray-100 dark:hover:bg-white/5 rounded-xl p-3">
                        <div class="flex-1">
                          <div class="text-sm font-medium text-gray-900 dark:text-white" :class="trustColor(item.trust_score)">
                            {{ item?.market?.name }}
                          </div>
                          <div class="text-xs text-gray-500 dark:text-gray-400">
                            {{ shortName(item?.base, item?.coin_id) }}/{{ shortName(item?.target, item?.target_coin_id) }}
                          </div>
                        </div>
                        <div class="text-sm font-semibold text-gray-900 dark:text-white">${{ item?.converted_last?.usd }}</div>
                      </div>
                    </li>
                  </ul>
                </div>
              </div>
            </div>

            <!-- Action Buttons -->
            <div class="flex gap-3">
              <label class="flex-none p-3 rounded-xl bg-white dark:bg-white/5 hover:bg-gray-100 dark:hover:bg-white/10 border border-gray-200 dark:border-white/10 hover:border-primary/30 cursor-pointer transition-all shadow-sm" for="calculator">
                <Icon icon="mdi:calculator" class="text-xl text-gray-500 dark:text-gray-400 hover:text-primary" />
              </label>
              <a
                class="flex-1 flex items-center justify-center gap-2 py-3 px-4 rounded-xl font-semibold text-white transition-all shadow-lg"
                :class="store.trustColor === 'green' ? 'bg-gradient-to-r from-emerald-500 to-green-600 hover:from-emerald-600 hover:to-green-700' : 'bg-gradient-to-r from-amber-500 to-orange-500 hover:from-amber-600 hover:to-orange-600'"
                :href="tickerUrl(ticker.trade_url)"
                target="_blank"
              >
                <Icon icon="mdi:shopping" />
                {{ $t('index.buy') }} {{ coinInfo.symbol || '' }}
              </a>
            </div>
          </div>

          <!-- Right Column: Price Chart -->
          <div class="lg:col-span-8">
            <div class="h-full rounded-2xl bg-white dark:bg-white/5 backdrop-blur-xl border border-gray-200 dark:border-white/10 overflow-hidden shadow-sm">
              <div class="flex items-center justify-between px-5 py-3 border-b border-gray-100 dark:border-white/10">
                <div class="flex items-center gap-2">
                  <div class="p-1.5 rounded-lg bg-primary/10">
                    <Icon icon="mdi:chart-line" class="text-lg text-primary" />
                  </div>
                  <span class="font-semibold text-gray-900 dark:text-white">Price Chart</span>
                </div>
                <div class="flex items-center gap-1 text-xs text-gray-500 dark:text-gray-400">
                  <Icon icon="mdi:clock-outline" class="text-sm" />
                  14 Days
                </div>
              </div>
              <div class="p-4">
                <PriceMarketChart />
              </div>
            </div>
          </div>
        </div>

        <!-- Bottom: Categories -->
        <div class="mt-6 pt-6 border-t border-gray-200/50 dark:border-white/10">
          <div class="flex flex-wrap gap-2">
            <span 
              v-for="tag in coinInfo.categories" 
              :key="tag"
              class="px-3 py-1.5 text-xs font-medium rounded-full bg-gray-100 dark:bg-white/5 text-gray-600 dark:text-gray-300 border border-gray-200 dark:border-white/10 hover:border-primary/30 hover:text-primary transition-all cursor-default"
            >
              {{ tag }}
            </span>
          </div>
        </div>
      </div>

      <!-- Calculator Modal -->
      <input type="checkbox" id="calculator" class="modal-toggle" />
      <div class="modal">
        <div class="modal-box bg-white dark:bg-slate-800 border border-gray-200 dark:border-white/10">
          <h3 class="text-lg font-bold text-gray-900 dark:text-white flex items-center gap-2">
            <Icon icon="mdi:calculator" class="text-primary" />
            {{ $t('index.price_calculator') }}
          </h3>
          <div class="flex flex-col w-full mt-5">
            <div class="grid h-20 flex-grow card rounded-box place-items-center">
              <div class="join w-full">
                <label class="join-item btn btn-primary"><span class="uppercase">{{ coinInfo.symbol }}</span></label>
                <input type="number" v-model="qty" min="0" placeholder="Input a number" class="input grow input-bordered join-item bg-gray-50 dark:bg-slate-700 border-gray-200 dark:border-slate-600 text-gray-900 dark:text-white" />
              </div>
            </div>
            <div class="divider text-gray-400">=</div>
            <div class="grid h-20 flex-grow card rounded-box place-items-center">
              <div class="join w-full">
                <label class="join-item btn btn-success"><span>USD</span></label>
                <input type="number" v-model="amount" min="0" placeholder="Input amount" class="join-item grow input input-bordered bg-gray-50 dark:bg-slate-700 border-gray-200 dark:border-slate-600 text-gray-900 dark:text-white" />
              </div>
            </div>
          </div>
        </div>
        <label class="modal-backdrop bg-black/60" for="calculator">{{ $t('index.close') }}</label>
      </div>
    </div>

    <!-- Creative Network Stats Dashboard -->
    <div class="grid grid-cols-12 gap-4">
      <!-- Network Pulse - Block Height (Featured) -->
      <div class="col-span-12 md:col-span-6 lg:col-span-4 relative overflow-hidden rounded-2xl bg-gradient-to-br from-primary via-primary/90 to-emerald-600 p-6 text-white">
        <div class="absolute top-0 right-0 w-32 h-32 bg-white/10 rounded-full -translate-y-1/2 translate-x-1/2"></div>
        <div class="absolute bottom-0 left-0 w-24 h-24 bg-white/5 rounded-full translate-y-1/2 -translate-x-1/2"></div>
        <div class="relative z-10">
          <div class="flex items-center gap-2 mb-3">
            <div class="p-2 rounded-lg bg-white/20 backdrop-blur-sm">
              <Icon icon="mdi:cube-scan" class="text-2xl" />
            </div>
            <span class="text-sm font-medium uppercase tracking-wider opacity-90">Block Height</span>
          </div>
          <div class="text-4xl font-bold tracking-tight">
            {{ store.stats?.height?.stats || '0' }}
          </div>
          <div class="mt-2 flex items-center gap-2 text-sm opacity-80">
            <Icon icon="mdi:pulse" class="animate-pulse" />
            <span>Live Network</span>
          </div>
        </div>
      </div>

      <!-- Validators Card -->
      <div class="col-span-6 md:col-span-3 lg:col-span-2 relative overflow-hidden rounded-2xl bg-white/60 dark:bg-gray-800/60 backdrop-blur-sm border border-gray-200/50 dark:border-gray-700/50 p-5 group hover:border-blue-400/50 transition-all">
        <div class="absolute -top-6 -right-6 w-20 h-20 bg-gradient-to-br from-blue-500/20 to-cyan-500/20 rounded-full blur-xl group-hover:scale-150 transition-transform"></div>
        <div class="relative z-10">
          <div class="flex items-center justify-between mb-3">
            <div class="p-2.5 rounded-xl bg-gradient-to-br from-blue-500 to-cyan-500">
              <Icon icon="mdi:account-group" class="text-xl text-white" />
            </div>
          </div>
          <div class="text-2xl font-bold text-gray-900 dark:text-white">
            {{ store.stats?.validators?.stats || '0' }}
          </div>
          <div class="text-xs text-gray-500 dark:text-gray-400 uppercase tracking-wide mt-1">Validators</div>
        </div>
      </div>

      <!-- Inflation Card with Gauge -->
      <div class="col-span-6 md:col-span-3 lg:col-span-2 relative overflow-hidden rounded-2xl bg-white/60 dark:bg-gray-800/60 backdrop-blur-sm border border-gray-200/50 dark:border-gray-700/50 p-5 group hover:border-amber-400/50 transition-all">
        <div class="absolute -top-6 -right-6 w-20 h-20 bg-gradient-to-br from-amber-500/20 to-orange-500/20 rounded-full blur-xl group-hover:scale-150 transition-transform"></div>
        <div class="relative z-10">
          <div class="flex items-center justify-between mb-3">
            <div class="p-2.5 rounded-xl bg-gradient-to-br from-amber-500 to-orange-500">
              <Icon icon="mdi:chart-line-variant" class="text-xl text-white" />
            </div>
          </div>
          <div class="text-2xl font-bold text-gray-900 dark:text-white">
            {{ store.stats?.inflation?.stats || '0%' }}
          </div>
          <div class="text-xs text-gray-500 dark:text-gray-400 uppercase tracking-wide mt-1">Inflation</div>
        </div>
      </div>

      <!-- Supply Card -->
      <div class="col-span-6 md:col-span-4 lg:col-span-4 relative overflow-hidden rounded-2xl bg-white/60 dark:bg-gray-800/60 backdrop-blur-sm border border-gray-200/50 dark:border-gray-700/50 p-5 group hover:border-purple-400/50 transition-all">
        <div class="absolute -bottom-8 -right-8 w-28 h-28 bg-gradient-to-br from-purple-500/20 to-pink-500/20 rounded-full blur-xl group-hover:scale-150 transition-transform"></div>
        <div class="relative z-10 flex items-center justify-between">
          <div>
            <div class="flex items-center gap-2 mb-2">
              <div class="p-2.5 rounded-xl bg-gradient-to-br from-purple-500 to-pink-500">
                <Icon icon="mdi:cash-multiple" class="text-xl text-white" />
              </div>
            </div>
            <div class="text-2xl font-bold text-gray-900 dark:text-white">
              {{ store.stats?.supply?.stats || '-' }}
            </div>
            <div class="text-xs text-gray-500 dark:text-gray-400 uppercase tracking-wide mt-1">Total Supply</div>
          </div>
          <div class="hidden md:block">
            <div class="w-16 h-16 rounded-full border-4 border-purple-200 dark:border-purple-900 relative">
              <div class="absolute inset-1 rounded-full bg-gradient-to-br from-purple-500 to-pink-500 flex items-center justify-center">
                <Icon icon="mdi:infinity" class="text-white text-xl" />
              </div>
            </div>
          </div>
        </div>
      </div>

      <!-- Bonded Tokens with Progress -->
      <div class="col-span-6 md:col-span-4 lg:col-span-4 relative overflow-hidden rounded-2xl bg-white/60 dark:bg-gray-800/60 backdrop-blur-sm border border-gray-200/50 dark:border-gray-700/50 p-5 group hover:border-emerald-400/50 transition-all">
        <div class="absolute -bottom-8 -left-8 w-28 h-28 bg-gradient-to-br from-emerald-500/20 to-green-500/20 rounded-full blur-xl group-hover:scale-150 transition-transform"></div>
        <div class="relative z-10">
          <div class="flex items-center justify-between mb-3">
            <div class="flex items-center gap-2">
              <div class="p-2.5 rounded-xl bg-gradient-to-br from-emerald-500 to-green-600">
                <Icon icon="mdi:lock" class="text-xl text-white" />
              </div>
            </div>
            <span class="text-xs px-2 py-1 rounded-full bg-emerald-100 dark:bg-emerald-900/30 text-emerald-600 dark:text-emerald-400 font-medium">Staked</span>
          </div>
          <div class="text-2xl font-bold text-gray-900 dark:text-white">
            {{ store.stats?.bonded_tokens?.stats || '-' }}
          </div>
          <div class="text-xs text-gray-500 dark:text-gray-400 uppercase tracking-wide mt-1">Bonded Tokens</div>
          <div class="mt-3 h-2 bg-gray-200 dark:bg-gray-700 rounded-full overflow-hidden">
            <div class="h-full bg-gradient-to-r from-emerald-500 to-green-500 rounded-full" style="width: 65%"></div>
          </div>
        </div>
      </div>

      <!-- Community Pool -->
      <div class="col-span-12 md:col-span-4 lg:col-span-4 relative overflow-hidden rounded-2xl bg-gradient-to-br from-indigo-500 via-purple-500 to-pink-500 p-5 text-white group">
        <div class="absolute inset-0 bg-[url('data:image/svg+xml;base64,PHN2ZyB3aWR0aD0iNjAiIGhlaWdodD0iNjAiIHZpZXdCb3g9IjAgMCA2MCA2MCIgeG1sbnM9Imh0dHA6Ly93d3cudzMub3JnLzIwMDAvc3ZnIj48ZyBmaWxsPSJub25lIiBmaWxsLXJ1bGU9ImV2ZW5vZGQiPjxnIGZpbGw9IiNmZmYiIGZpbGwtb3BhY2l0eT0iMC4xIj48cGF0aCBkPSJNMzYgMzRjMC0yLjIxLTEuNzktNC00LTRzLTQgMS43OS00IDQgMS43OSA0IDQgNCA0LTEuNzkgNC00eiIvPjwvZz48L2c+PC9zdmc+')] opacity-30"></div>
        <div class="relative z-10 flex items-center justify-between">
          <div>
            <div class="flex items-center gap-2 mb-2">
              <div class="p-2.5 rounded-xl bg-white/20 backdrop-blur-sm">
                <Icon icon="mdi:bank" class="text-xl" />
              </div>
              <span class="text-sm font-medium opacity-90">Community Pool</span>
            </div>
            <div class="text-3xl font-bold">
              {{ store.stats?.community_pool?.stats || '-' }}
            </div>
          </div>
          <div class="text-right opacity-80">
            <Icon icon="mdi:treasure-chest" class="text-5xl opacity-50" />
          </div>
        </div>
      </div>
    </div>

    <!-- Active Proposals -->
    <div
      v-if="blockchain.supportModule('governance')"
      class="rounded-xl bg-white/60 dark:bg-gray-800/60 backdrop-blur-sm border border-gray-200/50 dark:border-gray-700/50 overflow-hidden"
    >
      <div class="flex items-center justify-between px-5 py-4 border-b border-gray-200/50 dark:border-gray-700/50">
        <div class="flex items-center gap-3">
          <div class="p-2 rounded-lg bg-gradient-to-br from-primary/10 to-emerald-500/10">
            <Icon icon="mdi:vote" class="text-xl text-primary" />
          </div>
          <h3 class="text-lg font-semibold text-gray-900 dark:text-white">{{ $t('index.active_proposals') }}</h3>
        </div>
        <RouterLink :to="`/${chain}/gov`" class="text-sm text-primary hover:underline font-medium">
          View All
        </RouterLink>
      </div>
      <div class="p-5">
        <ProposalListItem :proposals="store?.proposals" />
      </div>
      <div class="pb-8 text-center text-gray-500" v-if="store.proposals?.proposals?.length === 0">
        {{ $t('index.no_active_proposals') }}
      </div>
    </div>

    <!-- Wallet Section -->
    <div class="rounded-xl bg-white/60 dark:bg-gray-800/60 backdrop-blur-sm border border-gray-200/50 dark:border-gray-700/50 overflow-hidden">
      <div class="flex items-center justify-between px-5 py-4 border-b border-gray-200/50 dark:border-gray-700/50">
        <div class="flex items-center gap-3">
          <div class="p-2 rounded-lg bg-gradient-to-br from-primary/10 to-emerald-500/10">
            <Icon icon="mdi:wallet" class="text-xl text-primary" />
          </div>
          <h3 class="text-lg font-semibold text-gray-900 dark:text-white truncate max-w-[300px]">
            {{ walletStore.currentAddress || 'Not Connected' }}
          </h3>
        </div>
        <RouterLink
          v-if="walletStore.currentAddress"
          class="text-sm text-primary hover:underline font-medium"
          :to="`/${chain}/account/${walletStore.currentAddress}`"
        >{{ $t('index.more') }}</RouterLink>
      </div>
      
      <div class="grid grid-cols-1 md:grid-cols-4 gap-4 p-5">
        <div class="rounded-xl bg-gradient-to-br from-blue-500/5 to-primary/5 border border-gray-200/50 dark:border-gray-700/50 p-4">
          <div class="text-sm text-gray-500 dark:text-gray-400 mb-1">{{ $t('account.balance') }}</div>
          <div class="text-lg font-bold text-gray-900 dark:text-white">
            {{ format.formatToken(walletStore.balanceOfStakingToken) }}
          </div>
          <div class="text-sm" :class="color">${{ format.tokenValue(walletStore.balanceOfStakingToken) }}</div>
        </div>
        <div class="rounded-xl bg-gradient-to-br from-emerald-500/5 to-green-500/5 border border-gray-200/50 dark:border-gray-700/50 p-4">
          <div class="text-sm text-gray-500 dark:text-gray-400 mb-1">{{ $t('module.staking') }}</div>
          <div class="text-lg font-bold text-gray-900 dark:text-white">
            {{ format.formatToken(walletStore.stakingAmount) }}
          </div>
          <div class="text-sm" :class="color">${{ format.tokenValue(walletStore.stakingAmount) }}</div>
        </div>
        <div class="rounded-xl bg-gradient-to-br from-yellow-500/5 to-orange-500/5 border border-gray-200/50 dark:border-gray-700/50 p-4">
          <div class="text-sm text-gray-500 dark:text-gray-400 mb-1">{{ $t('index.reward') }}</div>
          <div class="text-lg font-bold text-gray-900 dark:text-white">
            {{ format.formatToken(walletStore.rewardAmount) }}
          </div>
          <div class="text-sm" :class="color">${{ format.tokenValue(walletStore.rewardAmount) }}</div>
        </div>
        <div class="rounded-xl bg-gradient-to-br from-purple-500/5 to-pink-500/5 border border-gray-200/50 dark:border-gray-700/50 p-4">
          <div class="text-sm text-gray-500 dark:text-gray-400 mb-1">{{ $t('index.unbonding') }}</div>
          <div class="text-lg font-bold text-gray-900 dark:text-white">
            {{ format.formatToken(walletStore.unbondingAmount) }}
          </div>
          <div class="text-sm" :class="color">${{ format.tokenValue(walletStore.unbondingAmount) }}</div>
        </div>
      </div>

      <div v-if="walletStore.delegations.length > 0" class="px-5 pb-4 overflow-auto">
        <table class="table table-compact w-full">
          <thead>
            <tr class="border-b border-gray-200 dark:border-gray-700">
              <th class="text-gray-600 dark:text-gray-400">{{ $t('account.validator') }}</th>
              <th class="text-gray-600 dark:text-gray-400">{{ $t('account.delegations') }}</th>
              <th class="text-gray-600 dark:text-gray-400">{{ $t('account.rewards') }}</th>
              <th class="text-gray-600 dark:text-gray-400">{{ $t('staking.actions') }}</th>
            </tr>
          </thead>
          <tbody>
            <tr v-for="(item, index) in walletStore.delegations" :key="index" class="border-b border-gray-100 dark:border-gray-800">
              <td>
                <RouterLink class="text-primary hover:underline" :to="`/${chain}/staking/${item?.delegation?.validator_address}`">
                  {{ format.validatorFromBech32(item?.delegation?.validator_address) }}
                </RouterLink>
              </td>
              <td class="text-gray-700 dark:text-gray-300">{{ format.formatToken(item?.balance) }}</td>
              <td class="text-gray-700 dark:text-gray-300">
                {{ format.formatTokens(walletStore?.rewards?.rewards?.find((el) => el?.validator_address === item?.delegation?.validator_address)?.reward) }}
              </td>
              <td>
                <div class="flex gap-2">
                  <label for="delegate" class="btn btn-xs btn-primary btn-ghost rounded-lg" @click="dialog.open('delegate', { validator_address: item.delegation.validator_address }, updateState)">
                    {{ $t('account.btn_delegate') }}
                  </label>
                  <label for="withdraw" class="btn btn-xs btn-primary btn-ghost rounded-lg" @click="dialog.open('withdraw', { validator_address: item.delegation.validator_address }, updateState)">
                    {{ $t('index.btn_withdraw_reward') }}
                  </label>
                </div>
              </td>
            </tr>
          </tbody>
        </table>
      </div>

      <div class="grid grid-cols-3 gap-4 px-5 pb-5">
        <label for="PingTokenConvert" class="btn bg-gradient-to-r from-primary to-emerald-500 text-white border-0 rounded-xl hover:opacity-90">{{ $t('index.btn_swap') }}</label>
        <label for="send" class="btn bg-gradient-to-r from-green-500 to-emerald-600 text-white border-0 rounded-xl hover:opacity-90" @click="dialog.open('send', {}, updateState)">{{ $t('account.btn_send') }}</label>
        <label for="delegate" class="btn bg-gradient-to-r from-blue-500 to-indigo-600 text-white border-0 rounded-xl hover:opacity-90" @click="dialog.open('delegate', {}, updateState)">{{ $t('account.btn_delegate') }}</label>
      </div>
      <Teleport to="body">
        <ping-token-convert :chain-name="blockchain?.current?.prettyName" :endpoint="blockchain?.endpoint?.address" :hd-path="walletStore?.connectedWallet?.hdPath"></ping-token-convert>
      </Teleport>
    </div>

    <!-- Application Versions & Recent Transactions -->
    <div class="grid grid-cols-1 lg:grid-cols-2 gap-6">
      <!-- Application Versions -->
      <div class="rounded-xl bg-white/60 dark:bg-gray-800/60 backdrop-blur-sm border border-gray-200/50 dark:border-gray-700/50 overflow-hidden">
        <div class="flex items-center justify-between px-5 py-4 border-b border-gray-200/50 dark:border-gray-700/50">
          <div class="flex items-center gap-3">
            <div class="p-2 rounded-lg bg-gradient-to-br from-primary/10 to-emerald-500/10">
              <Icon icon="mdi:application-cog" class="text-xl text-primary" />
            </div>
            <h3 class="text-lg font-semibold text-gray-900 dark:text-white">{{ $t('index.app_versions') }}</h3>
          </div>
          <RouterLink :to="`/${chain}/params`" class="text-sm text-primary hover:underline font-medium">
            View More
          </RouterLink>
        </div>
        <div class="p-5">
          <ArrayObjectElement :value="paramStore.appVersion?.items" :thead="false" />
        </div>
      </div>

      <!-- Recent Blocks -->
      <div class="rounded-xl bg-white/60 dark:bg-gray-800/60 backdrop-blur-sm border border-gray-200/50 dark:border-gray-700/50 overflow-hidden">
        <div class="flex items-center justify-between px-5 py-4 border-b border-gray-200/50 dark:border-gray-700/50">
          <div class="flex items-center gap-3">
            <div class="p-2 rounded-lg bg-gradient-to-br from-primary/10 to-emerald-500/10">
              <Icon icon="mdi:cube-outline" class="text-xl text-primary" />
            </div>
            <h3 class="text-lg font-semibold text-gray-900 dark:text-white">Recent Blocks</h3>
          </div>
          <RouterLink :to="`/${chain}/block`" class="text-sm text-primary hover:underline font-medium">
            View More
          </RouterLink>
        </div>
        <div class="p-5">
          <div v-if="baseStore.recents && baseStore.recents.length > 0" class="space-y-2">
            <RouterLink 
              v-for="(block, index) in baseStore.recents.slice(-5).reverse()" 
              :key="block.block?.header?.height"
              :to="`/${chain}/block/${block.block?.header?.height}`"
              class="flex items-center justify-between p-3 rounded-lg hover:bg-gray-100 dark:hover:bg-gray-700/50 transition-all"
            >
              <div class="flex items-center gap-3">
                <div class="p-2 rounded-lg bg-primary/10">
                  <Icon icon="mdi:cube" class="text-primary" />
                </div>
                <div>
                  <div class="text-sm font-medium text-gray-900 dark:text-white">
                    Block #{{ block.block?.header?.height }}
                  </div>
                  <div class="text-xs text-gray-500">{{ block.block?.data?.txs?.length || 0 }} transactions</div>
                </div>
              </div>
              <Icon icon="mdi:chevron-right" class="text-gray-400" />
            </RouterLink>
          </div>
          <div v-else class="text-center text-gray-500 dark:text-gray-400 py-4">
            <Icon icon="mdi:cube-outline" class="text-4xl mb-2 opacity-50" />
            <p>Loading blocks...</p>
          </div>
        </div>
      </div>
    </div>

    <!-- Node Info (for chains without CoinGecko) -->
    <div v-if="!store.coingeckoId" class="rounded-xl bg-white/60 dark:bg-gray-800/60 backdrop-blur-sm border border-gray-200/50 dark:border-gray-700/50 overflow-hidden">
      <div class="flex items-center gap-3 px-5 py-4 border-b border-gray-200/50 dark:border-gray-700/50">
        <div class="p-2 rounded-lg bg-gradient-to-br from-primary/10 to-emerald-500/10">
          <Icon icon="mdi:server-network" class="text-xl text-primary" />
        </div>
        <h3 class="text-lg font-semibold text-gray-900 dark:text-white">{{ $t('index.node_info') }}</h3>
      </div>
      <div class="p-5">
        <ArrayObjectElement :value="paramStore.nodeVersion?.items" :thead="false" />
      </div>
    </div>
  </div>
</template>

<route>
  {
    meta: {
      i18n: 'dashboard',
      order: 1,
    }
  }
</route>
