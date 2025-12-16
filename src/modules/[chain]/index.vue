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
    <!-- Hero Section with Chain Info -->
    <div v-if="coinInfo && coinInfo.name" class="relative overflow-hidden rounded-2xl bg-gradient-to-br from-primary/5 via-transparent to-emerald-500/5 border border-gray-200/50 dark:border-gray-700/50">
      <div class="absolute inset-0 bg-grid-pattern opacity-5"></div>
      
      <div class="relative p-6">
        <div class="grid grid-cols-1 lg:grid-cols-3 gap-6">
          <!-- Left: Chain Info -->
          <div class="space-y-4">
            <div class="flex items-center gap-4">
              <div class="text-2xl font-bold text-gray-900 dark:text-white">
                {{ coinInfo.name }}
                <span class="text-lg text-gray-500 dark:text-gray-400 uppercase">({{ coinInfo.symbol }})</span>
              </div>
            </div>
            
            <div class="flex items-center gap-2">
              <span class="px-3 py-1 text-xs font-medium rounded-full bg-gradient-to-r from-red-500/10 to-orange-500/10 text-red-500 border border-red-500/20">
                Rank #{{ coinInfo.market_cap_rank }}
              </span>
            </div>

            <!-- Social Links -->
            <div class="flex flex-wrap gap-2">
              <a
                v-for="(item, index) of comLinks"
                :key="index"
                :href="item.href"
                class="flex items-center gap-1.5 px-3 py-1.5 text-sm rounded-lg bg-white/50 dark:bg-gray-800/50 border border-gray-200 dark:border-gray-700 hover:border-primary/50 hover:text-primary transition-all"
              >
                <Icon :icon="item?.icon" class="text-base" />
                <span class="uppercase text-xs font-medium">{{ item?.name }}</span>
              </a>
            </div>

            <!-- Exchange Dropdown -->
            <div class="dropdown dropdown-hover w-full">
              <label>
                <div class="bg-white/60 dark:bg-gray-800/60 backdrop-blur-sm flex items-center justify-between px-4 py-3 cursor-pointer rounded-xl border border-gray-200 dark:border-gray-700 hover:border-primary/50 transition-all">
                  <div>
                    <div class="font-semibold text-lg text-gray-900 dark:text-white">
                      {{ ticker?.market?.name || '' }}
                    </div>
                    <div class="text-primary text-sm">
                      {{ shortName(ticker?.base, ticker?.coin_id) }}/{{ shortName(ticker?.target, ticker?.target_coin_id) }}
                    </div>
                  </div>
                  <div class="text-right">
                    <div class="text-xl font-bold text-gray-900 dark:text-white">
                      ${{ ticker?.converted_last?.usd }}
                    </div>
                    <div class="text-sm font-medium" :class="store.priceColor">{{ store.priceChange }}%</div>
                  </div>
                </div>
              </label>
              <div class="dropdown-content pt-2 z-20">
                <div class="h-64 overflow-auto w-full shadow-xl rounded-xl border border-gray-200 dark:border-gray-700">
                  <ul class="menu w-full bg-white dark:bg-gray-800 rounded-xl">
                    <li v-for="(item, index) in store.coinInfo.tickers" :key="index" @click="store.selectTicker(index)">
                      <div class="flex items-center justify-between hover:bg-gray-50 dark:hover:bg-gray-700/50 rounded-lg">
                        <div class="flex-1">
                          <div class="text-sm font-medium" :class="trustColor(item.trust_score)">
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
            <div class="flex gap-2">
              <label class="btn btn-primary !px-3" for="calculator">
                <Icon icon="mdi:calculator" class="text-xl" />
              </label>
              <a
                class="btn grow text-white"
                :class="{ '!bg-gradient-to-r !from-emerald-500 !to-green-600': store.trustColor === 'green', '!bg-gradient-to-r !from-yellow-500 !to-orange-500': store.trustColor === 'yellow' }"
                :href="tickerUrl(ticker.trade_url)"
                target="_blank"
              >
                <Icon icon="mdi:cart" class="mr-1" />
                {{ $t('index.buy') }} {{ coinInfo.symbol || '' }}
              </a>
            </div>
          </div>

          <!-- Right: Price Chart -->
          <div class="lg:col-span-2">
            <PriceMarketChart />
          </div>
        </div>

        <!-- Description -->
        <div class="mt-6 pt-6 border-t border-gray-200/50 dark:border-gray-700/50">
          <div class="max-h-[200px] overflow-auto text-sm text-gray-600 dark:text-gray-400">
            <MdEditor :model-value="coinInfo.description?.en" previewOnly></MdEditor>
          </div>
          <div class="mt-4 flex flex-wrap gap-2">
            <span v-for="tag in coinInfo.categories" class="px-3 py-1 text-xs font-medium rounded-full bg-gray-100 dark:bg-gray-700/50 text-gray-600 dark:text-gray-300">
              {{ tag }}
            </span>
          </div>
        </div>
      </div>

      <!-- Calculator Modal -->
      <input type="checkbox" id="calculator" class="modal-toggle" />
      <div class="modal">
        <div class="modal-box">
          <h3 class="text-lg font-bold">{{ $t('index.price_calculator') }}</h3>
          <div class="flex flex-col w-full mt-5">
            <div class="grid h-20 flex-grow card rounded-box place-items-center">
              <div class="join w-full">
                <label class="join-item btn"><span class="uppercase">{{ coinInfo.symbol }}</span></label>
                <input type="number" v-model="qty" min="0" placeholder="Input a number" class="input grow input-bordered join-item" />
              </div>
            </div>
            <div class="divider">=</div>
            <div class="grid h-20 flex-grow card rounded-box place-items-center">
              <div class="join w-full">
                <label class="join-item btn"><span>USD</span></label>
                <input type="number" v-model="amount" min="0" placeholder="Input amount" class="join-item grow input input-bordered" />
              </div>
            </div>
          </div>
        </div>
        <label class="modal-backdrop" for="calculator">{{ $t('index.close') }}</label>
      </div>
    </div>

    <!-- Modern Stats Cards -->
    <div class="grid grid-cols-2 md:grid-cols-3 lg:grid-cols-6 gap-4">
      <div v-for="(item, key) in store.stats" :key="key">
        <CardStatisticsVertical v-bind="item" />
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
